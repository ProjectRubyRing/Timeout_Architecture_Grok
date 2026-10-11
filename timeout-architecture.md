# タイムアウト設計書

オンラインとバッチが同じ Aurora MySQL を使うときの待ち時間の決め方

- 対象構成の確認日: 2026-10-11
- 対象 DB: Aurora Serverless v2、MySQL 8.4.7 互換
- オンライン: ブラウザ → API Gateway REST API → VPC Link v2 → GWLBE → GWLB → Palo Alto → ALB → ECS Fargate（OpenJDK 21 Red Hat build、JBoss EAP 8.1）→ RDS Proxy → Aurora
- バッチ: EC2 上の Java（同じクラスファイルを `java` コマンドで実行）→ RDS Proxy → Aurora
- 数字の出典は末尾。実機のパラメータグループと、JDBC が GWLB を通るかどうかは、適用前に確認する。

この資料の推奨値は、画面向けの同期 API を 20 秒以内に終わらせる前提である。それを超える処理は、タイムアウトを伸ばす前に非同期（受付だけ返して、あとから結果を取りに行く）にする。

---

## 1. 最初に持つ絵

タイムアウトは「いつまで待つか」の約束である。

電話に置き換える。

- 質問している間の待ち時間。相手が考えている。これがリクエスト予算。
- 話が終わったあと、受話器を置いたまま無言でつないでおく時間。これがアイドル寿命。
- 電話をかけるとき、呼び出し音を何回まで聞くか。これが接続確立の待ち時間。

この三つは別の時計である。一つの不等式を全部にかぶせると、どこかで事故が起きる。

### 1.1 リクエスト予算（今の質問）

順番は、内側の作業が先に終わり、外側の待ちが長く残る。

```text
ロック待ち 10秒
  < SELECT の実行上限 20秒
  < JBoss のトランザクション 22秒
  < JDBC の socketTimeout 26秒
  < API Gateway 29秒
  < ブラウザ 35秒
  < ALB の無通信打ち切り 60秒（通常はここまで届かない）
```

教室のたとえ。

生徒（ブラウザ）が 35 秒待つ。受付（API Gateway）は 29 秒で「時間切れ」と紙を返す。先生（JBoss）は 22 秒で宿題をやめる。先生が先にやめるので、受付はその「できなかった」という紙を、まだ待っている生徒に渡せる。

逆に受付が先にドアを閉めると、教室では宿題が続き、生徒は失敗だと思って同じボタンを押す。あとから 1 回目が成功すると、更新が二回走る。これがこの構成で一番高い事故である。

### 1.2 アイドル寿命（無言の電話）

順番は、使う側が先に置き、奥の電話局が長く覚えておく。

```text
JBoss のプールが遊休接続を閉じる（約 5〜8 分）
  < オンライン用 RDS Proxy が放置クライアントを閉じる（15 分）
  < バッチ用 RDS Proxy が放置クライアントを閉じる（2 時間）
  < Aurora が放置セッションを閉じる（3 時間）
```

HTTP の別経路では、ALB が 60 秒の無通信で先に閉じ、JBoss Undertow は 70 秒まで接続を覚えておく。ALB が先に丁寧に切る。アプリが先に切ると、ALB は切れた接続へ次のリクエストを載せ、ブラウザには HTTP 502 が返る。

### 1.3 接続確立（呼び出し音）

呼ぶ側を短くする。

```text
オンラインの connectTimeout 5秒
  < バッチの connectTimeout 8秒
  < Aurora の connect_timeout 10秒
```

ALB から ECS への TCP 接続確立は 10 秒固定で、設定項目がない。

---

## 2. なぜ時計がいくつもあるか

1990 年代の TCP は、切れた線をいつまでも握らないためにタイムアウトを持った。当時の keepalive（生存確認パケット）の初期値は 2 時間である。電話代の高い時代に、確認パケットを頻繁に出したくなかった。Linux の `tcp_keepalive_time=7200` はその名残で、今も初期値のまま動いている。

MySQL の `wait_timeout=28800`（8 時間）は、人が `mysql` コマンドを開いたまま昼休みに行く、という使い方の初期値である。接続プールが「使っていない接続を何百本も握る」世界では長すぎる。切られたあとの次の SQL が `MySQL server has gone away` になる。

InnoDB のロック待ち 50 秒は、バッチが他の処理を少し待たせる前提の初期値である。画面のクリックには長い。待ち切れないと `ERROR 1205`（Lock wait timeout exceeded）になる。初期設定では、そのとき戻るのは失敗した SQL だけで、トランザクション全体は開いたままになる（`innodb_rollback_on_timeout=OFF`）。前の SQL のロックは残る。

JBoss のトランザクション 300 秒（5 分）は、アプリケーションサーバの中で画面もバッチも動いていた時代の折衷である。API Gateway の 29 秒より 10 倍長い。この差が、画面には失敗と出て、サーバではコミットが続く、という穴になる。

接続プールの「接続が借りられるまで 30 秒待つ」（`blocking-timeout-millis` の初期値 30000）も同じ時代の数字である。プールが空のとき、リクエストスレッドが 30 秒間ただ並ぶ。API が 29 秒で切れたあとも、サーバの中では行列が残る。

2010 年代にロードバランサの無通信 60 秒が広まった。1 分間なにも流れない HTTP は、人がもういない、と判断する数字である。ここで言葉が二つに割れた。

- 504 は、待ったが返事が来なかった。
- 502 は、相手が予告なく切れた。多くは、アプリの keepalive がロードバランサより短いときに起きる。

2015 年の API Gateway は、同期呼び出しの上限を 29 秒にした。Lambda を同期で受ける入口を、無制限の待ちから守るためである。以降、29 秒を超える処理はキューに入れて後で見る、という設計が定番になった。2024 年 6 月、Regional と Private の REST API は、クォータ申請で 29 秒を超えて伸ばせるようになった。伸ばすと、そのリージョンのスロットル枠を下げることがある。Edge-optimized API と HTTP API は対象外である。画面 API は 29 秒のままにする。

NAT Gateway や、以前の GWLB の TCP 無通信は 350 秒だった。300 秒でも 360 秒でもない。ファイアウォールを 300 秒に合わせた人が、50 秒差で切られた。2024 年 9 月、GWLB の TCP アイドルは 60〜6000 秒で変更できるようになった。初期値は互換のため 350 秒のままである。2026 年 9 月、無通信で切るときに TCP RST を返せるようになった。初期値はオフのままなので、何もしないと今も「黙って消える」。

Palo Alto の確立済み TCP の初期値は 3600 秒である。SAP や SSH のように長いセッションを、ファイアウォールが先に忘れると業務が切れる、という判断である。GWLB が 350 秒でフローを消し、ファイアウォールは 3600 秒覚えている、という差が問題になった。ファイアウォールの時間を 350 秒未満にしても、Palo Alto は RST を出さずにセッションを消すことがある。だから「ファイアウォールを短くする」だけでは黒穴は埋まらない。黒穴を避ける方法は、350 秒より前にエンドポイントが keepalive を出すか、エンドポイント自身が先に FIN で閉じることである。

RDS Proxy は、Lambda やコンテナが DB 接続を開きすぎる問題から生まれた。クライアント接続をまとめ、DB 接続を再利用する。放置クライアントは初期値 30 分で閉じる。DB 接続の借り出しは初期値 120 秒待つ。クライアント接続の寿命は、アイドルでなくても 24 時間で閉じる。この 24 時間は変更できない。

あとから分かった制約がピンニングである。一時テーブル、ユーザ変数、`GET_LOCK`、`LOCK TABLES`、16KB を超える SQL、追跡していない `SET`、サーバ側プリペアドステートメントがあると、そのクライアントは特定の DB 接続に固定される。固定された接続は他のリクエストに回せない。Proxy を入れたのに接続数が減らない、の典型原因がこれである。

Aurora Serverless v2 の `max_connections` は、今の ACU ではなく最大 ACU のメモリから決まる。縮小時に接続を切らないため、値は固定である。最大 ACU を変えたあとは再起動で反映される。1 ACU はおおよそ 2GiB である。公式の初期値は次のとおり。

| 最大 ACU | Aurora MySQL の max_connections 初期値 |
| ---: | ---: |
| 1 | 90 |
| 4 | 135 |
| 8 | 1,000 |
| 16 | 2,000 |
| 32 | 3,000 |
| 64 | 4,000 |
| 128 | 5,000 |
| 192 | 6,000 |
| 256 | 6,000 |

縮小時もこの接続上限は残る。ただしメモリは減る。遊んでいる接続が多いと、小さい ACU に戻るのを邪魔する。RDS Proxy を置く理由はここにある。

Aurora は `wait_timeout` と `interactive_timeout` の小さい方を、対話でも非対話でもすべての放置セッションに使う。片方だけ変えても、もう片方が短ければそちらが効く。

`aurora_transaction_timeout` は Aurora MySQL 8.4.8 以降にある。長いトランザクションが InnoDB のパージを止め、履歴が膨らむのを防ぐために追加された。初期値は 0（無効）。このシステムの 8.4.7 にはこのパラメータはない。8.4.7 のままでは、トランザクションの上限はアプリ側が持つ。

---

## 3. 経路で、どの時計が動くか

### 3.1 オンラインの 1 回のクリック

```text
ブラウザ 35秒待つ
  │  HTTP
  ▼
API Gateway  統合タイムアウト 29秒（ここが画面の天井）
  │  VPC Link v2（独自のリクエスト上限はない。29秒に従う）
  ▼
GWLBE → GWLB  TCP無通信 350秒（通常のAPIでは無関係。無言が350秒続いたフローを消す）
  ▼
Palo Alto  確立TCP 3600秒（同じく、短いHTTPではALBが先に閉じる）
  ▼
ALB  無通信60秒で504。接続確立は10秒固定。keepaliveの記憶は3600秒
  ▼
JBoss EAP 8.1 / Undertow
  トランザクション 22秒
  プールから接続を借りる待ち 3秒
  HTTP無リクエスト 70秒
  ▼
JDBC（データソースのURL。JNDIで取る人は全員これ）
  connectTimeout 5秒
  socketTimeout 26秒
  ▼
RDS Proxy（オンライン用）
  クライアント放置 15分
  DB接続の借り出し 3秒
  初期化クエリで lock wait 10秒、SELECT上限 20秒
  ▼
Aurora MySQL 8.4.7
  放置セッション 3時間
  接続ハンドシェイク待ち 10秒
```

VPC Link v2 のターゲット種別は ALB、NLB、Cloud Map である。GWLBE そのものに「VPC Link 専用タイムアウト」という項目はない。検査アプライアンスへ流すのは、ルートテーブルが GWLBE を指している、という形であることが多い。統合タイムアウトがリクエストの天井、GWLB の 350 秒がフローの無通信天井、という二段になる。

JDBC の経路は、記載された構成では GWLB と Palo Alto を通らない。ECS から RDS Proxy へ VPC 内で直接行く前提である。東西の通信もファイアウォールへ回している場合は、JDBC の keepalive を 350 秒未満にし、GWLB の TCP RST を有効にする。この資料の数字は、DB 区間を検査装置に通さない前提である。

### 3.2 1 秒もパケットが出ないとき、誰が先に鳴るか

画面 API で SQL が無言のまま走っているとき。

1. 10 秒で InnoDB ロック待ちが鳴る（オンライン用 Proxy の初期化クエリ）。
2. SELECT は 20 秒でサーバが止める（`max_execution_time`。UPDATE や DELETE には効かない）。
3. 22 秒で JBoss がトランザクションを戻す。
4. それでもソケットが無言なら 26 秒でドライバが切る。
5. 29 秒で API Gateway が HTTP 504 を返す。
6. ブラウザは 35 秒まで待つので、その 504 の本文を受け取れる。
7. ALB の 60 秒は、API Gateway が先に鳴るので、通常の同期 API では予備である。

### 3.3 バッチ

スケジューラが `java` でクラスを起動する。EAP はいない。JNDI もいない。JBoss の `standalone.xml` に書いたタイムアウトはバッチに届かない。

同じ JAR でも、接続の作り方がコンテナの外では別になる。共有コードが `new InitialContext().lookup("java:jboss/datasources/...")` だけだと、EC2 の `java` コマンドでは失敗する。バッチ用の JDBC URL を渡す差し込み口が必要である。推測どおり、バッチの socket タイムアウトは Connector/J の URL に書く。オンラインの socket タイムアウトは、JNDI データソースの URL に書けば、そのデータソースを使うオンライン処理で共通になる。

バッチの 1 本の SQL は、結果を返すまでパケットが流れないことがある。`socketTimeout` は「次のパケットまでの無言」の上限なので、10 分に設定すると、10 分無言の正当な UPDATE も切れる。バッチでは `socketTimeout=0`（無制限）にし、死んだ相手の発見は TCP keepalive に任せる。

EC2 の初期 keepalive は 2 時間である。これでは途中の装置が先に忘れる。次に変える。

```text
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 5
```

無通信でも、おおよそ 60 + 10 × 5 = 110 秒で死んだ相手に気づく。生きているサーバは ACK を返すので、2 時間かかる SQL は切れない。

Fargate のオンライン側は、リクエスト中の無言を `socketTimeout=26秒` が切る。OS の 2 時間 keepalive が残っていても、同期 API の予算はドライバが守る。プールの遊休接続は JBoss が約 5〜8 分で閉じる。

---

## 4. 推奨値一覧

「変更」は初期値から動かすもの。「維持」は初期値のままにするもの。重要は、ここを誤ると二重更新か 502 になる段である。

### 4.1 オンラインのリクエスト予算

| 層 | パラメータ | 初期値 | 推奨 | 扱い | 重要 |
| --- | --- | --- | --- | --- | --- |
| ブラウザ | `fetch` / XHR の timeout | 0（無制限） | 35000 ミリ秒 | 変更 | はい |
| API Gateway REST | 統合タイムアウト `timeoutInMillis` | 29000 ミリ秒 | 29000 ミリ秒 | 維持 | はい |
| API Gateway | アイドル接続 | 310 秒 | 変更不可 | 維持 | |
| VPC Link v2 | リクエスト上限 | なし（統合タイムアウトに従う） | 追加しない | 維持 | |
| ALB | `idle_timeout.timeout_seconds` | 60 秒（1〜4000） | 60 秒 | 維持 | はい |
| ALB | ターゲットへの TCP 接続確立 | 10 秒 | 変更不可 | 維持 | |
| ALB | `client_keep_alive.seconds` | 3600 秒（60〜604800） | 3600 秒 | 維持 | |
| Undertow | `no-request-timeout` | 60000 ミリ秒 | 70000 ミリ秒 | 変更 | はい |
| Undertow | `read-timeout` | 未設定（無制限） | 70000 ミリ秒 | 変更 | |
| Undertow | `write-timeout` | 未設定（無制限） | 70000 ミリ秒 | 変更 | |
| Undertow | `request-parse-timeout` | 未設定 | 10000 ミリ秒 | 変更 | |
| JBoss transactions | `default-timeout` | 300 秒 | 22 秒 | 変更 | はい |
| JBoss transactions | `maximum-timeout` | 31536000 秒（365日） | 維持。0 を指定したトランザクションの天井 | 維持 | |
| データソース | `query-timeout` | なし（秒） | 22 秒 | 変更 | はい |
| データソース | `set-tx-query-timeout` | false | true | 変更 | |
| JDBC | `socketTimeout` | 0（無制限） | 26000 ミリ秒 | 変更 | はい |
| RDS Proxy 初期化クエリ | `max_execution_time` | 0（SELECT 無制限、ミリ秒） | 20000 | 変更 | はい |
| RDS Proxy 初期化クエリ | `innodb_lock_wait_timeout` | グローバル 50 秒 | セッション 10 秒 | 変更 | はい |

ブラウザを 35 秒にする理由は、API Gateway の 504 本文を受け取らせるためである。ブラウザを 20 秒にすると、ユーザには通信エラーが出て、サーバは 29 秒まで処理を続ける。POST の自動リトライはしない。GET で同じ結果になるものだけ、1 回まで再試行できる。

統合タイムアウトを 60 秒や 300 秒に上げる変更は、この表の全部を組み直す作業になる。一つの API だけ上げると、JBoss の 22 秒と ALB の 60 秒の前提が崩れる。

`max_execution_time` は SELECT だけに効く。INSERT、UPDATE、DELETE、DDL は止まらない。更新文の天井はロック待ち 10 秒と、JTA の 22 秒である。

`set-tx-query-timeout=true` は、トランザクションの残り時間を JDBC のクエリタイムアウトに載せる。Connector/J は別接続で `KILL QUERY` を出す実装である。RDS Proxy の接続 ID とサーバのスレッド ID が一致せず、KILL が空振りすることがある。SELECT の本線は Proxy 初期化クエリの `max_execution_time` に置く。KILL は補助である。

### 4.2 オンラインのプールと接続

| 層 | パラメータ | 初期値 | 推奨 | 扱い | 重要 |
| --- | --- | --- | --- | --- | --- |
| データソース | `blocking-timeout-millis`（CLI 名 `blocking-timeout-wait-millis`） | 30000 ミリ秒 | 3000 ミリ秒 | 変更 | はい |
| データソース | `idle-timeout-minutes` | 未指定時 30 分 | 5 分 | 変更 | はい |
| データソース | `background-validation` | false | true | 変更 | |
| データソース | `background-validation-millis` | 未設定 | 60000 ミリ秒 | 変更 | |
| データソース | `validate-on-match` | 実装依存。借りるたびに検査すると遅い | false | 変更 | |
| データソース | 検査クラス | なし | `MySQLValidConnectionChecker` | 変更 | |
| データソース | 例外ソータ | なし | `MySQLExceptionSorter` | 変更 | |
| データソース | `allocation-retry` | 0 | 0 | 維持 | |
| データソース | `use-ccm` | true | true | 維持 | |
| JDBC | `connectTimeout` | 0（無制限） | 5000 ミリ秒 | 変更 | はい |
| JDBC | `tcpKeepAlive` | true | true | 維持 | |
| JDBC | `useServerPrepStmts` | false | false | 維持 | はい |
| JDBC | `cachePrepStmts` | false | false | 維持 | |
| JDBC | `useLocalSessionState` | false | true | 変更 | |
| プール最大 | `max-pool-size` | 20 がよくある初期 | 最大 ACU の接続数の範囲で決める | 別計算 | |

`blocking-timeout` は「プールの空きを待つ時間」だけである。新しい TCP 接続を張る時間は含まない。接続そのものは `connectTimeout=5秒` が切る。最悪でも、空き待ち 3 秒 + 接続 5 秒で、29 秒の予算の中に収まる。

空き待ちを 30 秒のままにすると、プール枯渇のとき API Gateway が先に 504 を返し、JBoss のスレッドはまだ並んでいる。

`idle-timeout-minutes=5` の実時間は、IdleRemover の周期ぶん遅れる。周期は、全プールの最小アイドル時間の半分である。5 分設定なら周期 2.5 分、閉じるのはおおよそ 5〜7.5 分。オンライン Proxy の 15 分より前に閉じる。

`min-pool-size` を大きくし、`use-strict-min=true` にすると、アイドル時間を過ぎてもその本数は残る。残したい本数だけ min に置く。

バックグラウンド検査の `COM_PING` や `SELECT 1` は、RDS Proxy の放置タイマーをリセットする。AWS は、クエリも `COM_PING` も放置扱いにしない、と書いている。だからオンラインのプール接続は、Proxy の 15 分では閉じない。閉じるのは JBoss のアイドル除去である。Proxy の 15 分は、検査が止まった接続と、アプリが返却し忘れた接続の受け皿である。

`useServerPrepStmts=true` はサーバ側プリペアドになり、RDS Proxy が接続を固定しやすい。初期値 false のクライアント側エミュレーションを維持する。`useLocalSessionState=true` は、ドライバが同じ分離レベルを何度もサーバへ問い合わせる回数を減らす。

オンラインの URL に `sessionVariables=...` を付けない。接続のたびに `SET` が走り、Proxy のピンニング条件になる。セッション変数は Proxy の初期化クエリへ置く。初期化クエリの `SET` は、クライアントの SQL としてのピンニングを避ける公式の置き場である。

### 4.3 アイドル寿命と Proxy、Aurora

| 層 | パラメータ | 初期値 | 推奨 | 扱い | 重要 |
| --- | --- | --- | --- | --- | --- |
| RDS Proxy（オンライン） | `IdleClientTimeout` | 1800 秒（範囲 1 分〜8 時間） | 900 秒 | 変更 | はい |
| RDS Proxy（オンライン） | `ConnectionBorrowTimeout` | 120 秒（範囲 0〜300 秒） | 3 秒 | 変更 | はい |
| RDS Proxy（オンライン） | `MaxConnectionsPercent` | 100 | 70 | 変更 | はい |
| RDS Proxy（オンライン） | `MaxIdleConnectionsPercent` | 50（MySQL） | 50 | 維持 | |
| RDS Proxy（バッチ） | `IdleClientTimeout` | 1800 秒 | 7200 秒 | 変更 | はい |
| RDS Proxy（バッチ） | `ConnectionBorrowTimeout` | 120 秒 | 60 秒 | 変更 | |
| RDS Proxy（バッチ） | `MaxConnectionsPercent` | 100 | 20 | 変更 | はい |
| RDS Proxy（バッチ） | `MaxIdleConnectionsPercent` | 50 | 50 | 維持 | |
| RDS Proxy | クライアント接続の最大寿命 | 24 時間 | 変更不可 | 維持 | |
| RDS Proxy | バックエンドを遊休とみなす無通信 | 5 分 | 変更不可。本数の上限は MaxIdleConnectionsPercent | 維持 | |
| Aurora | `wait_timeout` | 28800 秒 | 10800 秒 | 変更 | はい |
| Aurora | `interactive_timeout` | 28800 秒 | 10800 秒 | 変更 | はい |
| Aurora | `connect_timeout` | 10 秒 | 10 秒 | 維持 | |
| Aurora | `net_read_timeout` | 30 秒 | グローバルは 30 秒。バッチ Proxy の初期化クエリで 120 秒 | 分割 | |
| Aurora | `net_write_timeout` | 60 秒 | グローバルは 60 秒。バッチ Proxy の初期化クエリで 300 秒 | 分割 | |
| Aurora | `innodb_lock_wait_timeout` | 50 秒 | グローバルは 50 秒のまま。オンライン初期化クエリ 10 秒、バッチ初期化クエリ 120 秒 | 分割 | はい |
| Aurora | `max_execution_time` | 0 | グローバルは 0。オンライン初期化クエリだけ 20000 | 分割 | はい |
| Aurora | `innodb_rollback_on_timeout` | OFF | OFF | 維持 | はい |
| Aurora | `lock_wait_timeout`（メタデータロック） | 31536000 秒 | 維持。DDL の話であり、画面 SQL の時計ではない | 維持 | |
| Aurora | `aurora_transaction_timeout` | 8.4.7 には無い。8.4.8 以降の初期値は 0 | 8.4.7 では設定しない | 対象外 | はい |

Proxy を二つに分ける理由は、放置時間、借り出し待ち、初期化クエリが Proxy 単位で、クライアント単位では分けられないからである。AWS はターゲットあたり Proxy 一つを好み、複数にするときは `MaxConnectionsPercent` の合計を 100 以下にする、と書いている。ここは 70 + 20 = 90 とし、残り 10% を管理接続と直接接続に残す。

最大 ACU が 1 のとき `max_connections` は 90 である。オンライン Proxy の 70% は約 63 本、バッチは約 18 本。タスク数 × `max-pool-size` が、このバックエンド本数を大きく超えると、借り出し 3 秒で失敗する。プールを増やす前に最大 ACU を見る。

Proxy はバックエンド接続を時々 ping し、MySQL 側の放置タイマーをリセットする。だから `wait_timeout` を Proxy より少し長くしても、ping がある限り DB は先に切りにくい。それでも ping 間隔は文書に固定値として出ていない。バッチ Proxy の 2 時間より長い 3 時間を、両方のタイムアウトに明示して余白を作る。

`interactive_timeout` を 8 時間のままにすると、Aurora は min(10800, 28800)=10800 を使うので結果は同じに見える。将来どちらかを短く触ったとき、小さい方が全員に効く。両方を 10800 に揃える。

`net_read_timeout` は、サーバがクライアントからの続きのパケットを待つ時間である。クエリ実行時間ではない。大きな INSERT の送信が 30 秒止ると切れる。画面の小さいリクエストは 30 秒で足りる。バッチの大きな送受信だけ、その Proxy の初期化クエリで伸ばす。

`innodb_rollback_on_timeout=OFF` のとき、1205 で戻るのはその SQL だけである。アプリは 1205 を受けたらトランザクションを明示的にロールバックする。JTA の 22 秒は、戻し忘れたときの天井になる。ON にするとトランザクション全体が戻るが、起動時の値で、バッチにも効く。共有 DB では OFF のまま、戻し方をアプリに置く。

### 4.4 バッチの JDBC と OS

| 層 | パラメータ | 初期値 | 推奨 | 扱い | 重要 |
| --- | --- | --- | --- | --- | --- |
| JDBC | `connectTimeout` | 0 | 8000 ミリ秒 | 変更 | はい |
| JDBC | `socketTimeout` | 0 | 0（無制限） | 維持 | はい |
| JDBC | `tcpKeepAlive` | true | true | 維持 | はい |
| OS | `tcp_keepalive_time` | 7200 秒 | 60 秒 | 変更 | はい |
| OS | `tcp_keepalive_intvl` | 75 秒 | 10 秒 | 変更 | |
| OS | `tcp_keepalive_probes` | 9 | 5 | 変更 | |
| ジョブスケジューラ | ジョブ打ち切り | 製品依存 | 業務ごとに決める。DB の 120 秒とは別 | 追加 | はい |

バッチが SQL を出さずに 2 時間以上止まると、バッチ Proxy がクライアント接続を閉じる。ファイル処理の長いステップがあるジョブは、再接続するか、2 時間未満で一度 SQL を通す。`SELECT 1` は放置タイマーをリセットするので、接続は残る。その代わり、その接続は「仕事をしている」と Proxy に見える。

バッチのロック待ち 120 秒は、オンラインの 10 秒より長い。オンラインは 10 秒で 1205 を受けて戻る。バッチが 1 本のトランザクションを 2 時間開いたままだと、オンラインは何度 1205 になっても、ロックの持ち主は残る。チャンクごとにコミットする。8.4.7 にはトランザクション時間のサーバ側上限がない。長いトランザクションは `RollbackSegmentHistoryListLength` の増加で気づく。

ジョブ全体の打ち切り（想定時間の 1.5 倍、など）はスケジューラの仕事である。JDBC の `socketTimeout=0` は、その代わりをしない。

### 4.5 経路装置（オンラインの HTTP が通る場所）

| 層 | パラメータ | 初期値 | 推奨 | 扱い | 重要 |
| --- | --- | --- | --- | --- | --- |
| GWLB | `tcp.idle_timeout.seconds` | 350 秒（60〜6000） | 350 秒 | 維持 | |
| GWLB | アイドル時 TCP RST | false（2026-09 追加、既定オフ） | 有効にする | 変更 | はい |
| GWLB | 350 秒を超える場合の ENI | Nitro 系の確立 TCP 追跡は 350 秒という説明がある | 350 のままなら変更しない。伸ばすならターゲット ENI の `TcpEstablishedTimeout` を GWLB 以上にする | 条件付き | |
| Palo Alto | TCP（確立後の無応答） | 3600 秒 | 3600 秒 | 維持 | |
| Palo Alto | TCP ハンドシェイク完了前 | 10 秒 | 10 秒 | 維持 | |
| Palo Alto | TCP ハーフクローズ | 120 秒 | 120 秒 | 維持 | |
| ALB | 登録解除の遅延 `deregistration_delay.timeout_seconds` | 300 秒 | 30 秒 | 変更 | |
| Fargate | コンテナ停止待ち（stopTimeout） | 30 秒 | 30 秒。登録解除のあとに SIGTERM が来る順にする | 維持 | |

GWLB を 350 秒より長くすると、ファイアウォールのフェイルオーバー後も古いフローの貼り付きが残る。画面 HTTP は ALB の 60 秒が先に FIN で閉じるので、GWLB を伸ばす理由がない。

TCP RST は 2026-09-03 の機能で、初期値はオフである。リスナー属性 `send_tcp_reset.on_idle_timeout.enabled` が文書にある。アナウンスでは、ターゲット異常、登録解除、アイドル満了をターゲットグループ側で個別に有効化できる、とも書かれている。コンソールの項目名は適用時に画面で確認する。オフのままだと、フローが消えても両端は数分、再送を続ける。

Palo Alto のアプリケーション識別（`ssl`、`web-browsing`、`mysql`）には、グローバル TCP とは別のタイムアウトがある。`show session info` と、通過しているアプリのオブジェクトを見る。HTTP 経路は ALB が 60 秒で閉じるので、アプリタイムアウトが 3600 秒でも画面 API は壊れにくい。JDBC をこのファイアウォールに通したときだけ、`mysql` のタイムアウトと GWLB 350 秒の差が本線になる。

登録解除 300 秒は、デプロイのたびに古いタスクが 5 分リクエストを持ち続ける。画面 API の予算が 29 秒なので、ドレインは 30 秒で足りる。処理中のリクエストを 22 秒のトランザクションで終わらせてからタスクを止める。

### 4.6 やらないこと

- オンラインの業務コードで `SET` を実行しない。初期化クエリに寄せる。
- `autoReconnect=true` を付けない。切れた接続の上で SQL を再送し、更新が二回走ることがある。
- XA データソースを RDS Proxy 経由にしない。Proxy は XA を前提にしていない。オンラインがリソース一つなら、非 XA データソース + JTA（最後のリソースを 1 フェーズでコミットする動き）で足りる。二つ以上の資源を 2 フェーズコミットする要件が来たら、Proxy の外で設計を変える。
- 最小 ACU を 0（自動停止）にしない。再開は 5 秒を超えることがあり、`connectTimeout=5秒` のオンラインは失敗する。失敗を遅くするために connectTimeout を伸ばすと、止まった DB へ画面スレッドが長くぶら下がる。共有の本番は最小 0.5 以上、余裕があれば 1 以上にする。
- グローバルの `innodb_lock_wait_timeout` を 10 秒にして、バッチの付け忘れに頼らない、でもない。グローバルは 50 秒のままにし、オンラインは Proxy 初期化クエリの 10 秒で必ず短くする。バッチ Proxy は 120 秒。初期化クエリが空だと、両方とも 50 秒になり、オンラインは JTA 22 秒が主天井、ロック解放は 50 秒まで遅れる。初期化クエリの適用は必須である。
- 8.4.7 に `aurora_transaction_timeout` を書かない。パラメータが無く、適用で失敗する。上げるなら 8.4.8 以降にして、グローバルは 0 のまま、オンライン Proxy の初期化クエリでセッション値 22 を検討する。その前に、そのバージョンの初期化クエリでセッション変数が受け入れられることを確認する。

---

## 5. 設定する場所

### 5.1 オンラインの JDBC URL

JBoss のデータソース `connection-url`。JNDI 利用者は個別に socket を指定できない。ここが共通点である。

```text
jdbc:mysql://<online-proxy-endpoint>:3306/<db>?sslMode=REQUIRED&connectTimeout=5000&socketTimeout=26000&tcpKeepAlive=true&useServerPrepStmts=false&cachePrepStmts=false&useLocalSessionState=true
```

`sessionVariables` は付けない。

### 5.2 バッチの JDBC URL

バッチ起動側の設定。EAP の XML には書かない。Connector/J のバージョンはオンラインのモジュールと同じものに固定する。

```text
jdbc:mysql://<batch-proxy-endpoint>:3306/<db>?sslMode=REQUIRED&connectTimeout=8000&socketTimeout=0&tcpKeepAlive=true&useServerPrepStmts=false&cachePrepStmts=false&useLocalSessionState=true
```

### 5.3 RDS Proxy の初期化クエリ

オンライン:

```sql
SET SESSION innodb_lock_wait_timeout = 10, max_execution_time = 20000
```

バッチ:

```sql
SET SESSION innodb_lock_wait_timeout = 120, net_read_timeout = 120, net_write_timeout = 300
```

### 5.4 JBoss EAP 8.1 CLI の例

リスナー名とデータソース名は実機に合わせる。HTTPS リスナーがあるなら、同じ 70000 を付ける。ALB で TLS を終わらせ、JBoss が HTTP だけなら `http-listener=default` で足りる。

```text
/subsystem=transactions:write-attribute(name=default-timeout,value=22)

/subsystem=undertow/server=default-server/http-listener=default:write-attribute(name=no-request-timeout,value=70000)
/subsystem=undertow/server=default-server/http-listener=default:write-attribute(name=read-timeout,value=70000)
/subsystem=undertow/server=default-server/http-listener=default:write-attribute(name=write-timeout,value=70000)
/subsystem=undertow/server=default-server/http-listener=default:write-attribute(name=request-parse-timeout,value=10000)

/subsystem=datasources/data-source=<NAME>:write-attribute(name=blocking-timeout-wait-millis,value=3000)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=idle-timeout-minutes,value=5)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=query-timeout,value=22)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=set-tx-query-timeout,value=true)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=background-validation,value=true)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=background-validation-millis,value=60000)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=validate-on-match,value=false)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=valid-connection-checker-class-name,value=org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker)
/subsystem=datasources/data-source=<NAME>:write-attribute(name=exception-sorter-class-name,value=org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLExceptionSorter)
```

XML に直接書く場合、空き待ちの要素名は `blocking-timeout-millis` である。CLI の属性名は `blocking-timeout-wait-millis` である。同じ設定の二つの名前である。

`idle-timeout-minutes` の変更は再起動が必要、と EAP の属性説明にある。反映手順は変更ウィンドウで確認する。

トランザクション 22 秒はサーバ全体の初期値である。どうしても長いオンライン処理があるなら、その処理だけ `@TransactionTimeout` で上書きする。上書きした処理を、統合タイムアウト 29 秒の同期 API に載せることはできない。

### 5.5 Aurora のパラメータグループ

```text
wait_timeout = 10800
interactive_timeout = 10800
```

`connect_timeout`、`innodb_lock_wait_timeout`、`max_execution_time`、`innodb_rollback_on_timeout` は初期値のまま。ロック待ちと SELECT 上限は Proxy の初期化クエリがセッションで上書きする。

### 5.6 アプリの戻し方

`SQLException` のエラーコード 1205 を受けたら、そのトランザクションをロールバックして HTTP エラーを返す。次の SQL を同じトランザクションで続けない。

POST は、ブラウザのタイムアウトや 504 を見て自動でもう一度送らない。業務で再送が必要なら、同じ要求 ID で 1 回だけ効くようにする。

---

## 6. 失敗したとき、どこで鳴ったか

| ユーザやログに出るもの | 鳴った時計 | サーバ側で処理は続いているか |
| --- | --- | --- |
| ブラウザの通信エラー。API の JSON が無い | ブラウザが API Gateway より先に切った。推奨 35 秒になっていない | 続いていることがある |
| API Gateway の HTTP 504。本文は `{"message":"Endpoint request timed out"}` | 統合タイムアウト 29 秒 | JTA が 22 秒なら、通常はもう戻している。JTA が初期値 300 秒だと続く |
| ALB の HTTP 504。`HTTPCode_ELB_504_Count` | ターゲットが無通信のままアイドル秒に達した、または接続確立 10 秒に失敗した | アプリが生きていれば処理は別問題 |
| ALB の HTTP 502 | アプリが ALB より先に接続を閉じた。Undertow 60 秒と ALB 60 秒の同着が典型 | そのリクエストは届いていない |
| ALB の HTTP 460 | クライアントが、ALB のアイドルより先に切った | ALB から先は動き始めていることがある |
| ALB の HTTP 408 | クライアントがリクエストを送り切る前に無通信 | アプリには届いていない |
| `IJ000655` No managed connections available within configured blocking timeout | プール空き待ち。推奨後は 3 秒 | SQL は始まっていない |
| ドライバの `CommunicationsException` | `socketTimeout` か、途中の装置が RST/黒穴にした | サーバは次に書き出そうとしたときに気づいて止める。SELECT は `max_execution_time` の方が確実 |
| MySQL 1205 | ロック待ち。オンライン 10 秒、バッチ 120 秒 | 文だけが戻る。トランザクションは開いている |
| MySQL 3024 | `max_execution_time`。SELECT が 20 秒を超えた | その SELECT は止まっている |
| `MySQL server has gone away` | 放置で切られたあとに SQL を出した。検査していないプール接続、または Proxy と DB の寿命の逆転 | その接続は死んでいる。ソータがプールから捨てる |

オンラインで Proxy のピンニング数がリクエストと一緒に増えるときは、業務コードの `SET`、一時テーブル、16KB 超の SQL、サーバ側プリペアドを疑う。メトリクス名はコンソールの RDS Proxy メトリクスで `SessionPinned` を含むものを見る。

---

## 7. 動作の確認手順

適用後に、本番の全量を流す前へ、次を 1 本ずつ見る。

1. オンライン Proxy に接続し、`SELECT @@SESSION.innodb_lock_wait_timeout, @@SESSION.max_execution_time;` が 10 と 20000 であること。
2. バッチ Proxy に接続し、ロック待ちが 120、`net_read_timeout` が 120、`net_write_timeout` が 300 であること。
3. Aurora で `SELECT @@GLOBAL.wait_timeout, @@GLOBAL.interactive_timeout;` がどちらも 10800 であること。`@@GLOBAL.aurora_transaction_timeout` は 8.4.7 ではエラーになる。エラーになることをもって「無い」と判断してよい。
4. 意図的に 25 秒かかる SELECT を同期 API から実行し、API Gateway の 504 ではなく、アプリのエラー応答が 22 秒前後で返ること。
5. その直後に同じ主キーへ更新を二回送っても、1 回目が遅れて成功しないこと。トランザクションが戻っていること。
6. プールを使い切り、3 秒前後で `IJ000655` になること。30 秒間スレッドが眠らないこと。
7. ALB のターゲットを外し、登録解除が 30 秒で完了し、その間の新規リクエストが別タスクへ行くこと。
8. バッチで 3 分無言の `SELECT SLEEP(180)` が、`socketTimeout=0` と短縮 keepalive の下で最後まで終わること。相手を強制停止した接続は、おおよそ 2 分以内にバッチ側が通信エラーにすること。

---

## 8. この資料が置いている前提

実測で崩れていたら、数字より前提を先に直す。

- API は Regional または Private の REST API である。Edge-optimized は 29 秒から伸ばせないが、この推奨は伸ばさないので、種別が Edge でもリクエスト天井は同じ 29 秒である。VPC Link の有無は別問題である。
- 同期 API の業務処理は 20 秒で足りる。足りない API は非同期にする。
- JDBC は GWLB / Palo Alto を通らない。
- オンラインとバッチで RDS Proxy を二つに分けられる。一つにまとめる場合、`IdleClientTimeout` と借り出しと初期化クエリは両方の厳しい方に合わせられない。一つにするなら、初期化クエリは空にせざるを得ず、ロック待ちの分離ができなくなる。その場合は設計をやり直す。
- バッチは JBoss の外で動き、独自の JDBC URL を持つ。
- データソースは一つで、XA ではない。
- 最小 ACU は 0 ではない。
- DB エンジンは Aurora MySQL 8.4.7 である。8.4.8 のトランザクションタイムアウトは使わない。

Connector/J の `connectTimeout`、`socketTimeout`、`tcpKeepAlive`、`useServerPrepStmts` の初期値は、現行の Connector/J 文書（8.4 系の説明を含む）で確認した。パッチ版でこれら四つの初期値が変わった、という記述は見当たらない。使う JAR の版はオンラインとバッチで揃える。

`idle-timeout-minutes` の「未指定なら 30 分」は、Red Hat が長らく文書化している IronJacamar の動作である。EAP 8.1 の実機では、管理 CLI の `include-defaults=true` で値を見る。

GWLB の TCP RST の画面上の項目は、2026-09-03 公開直後の機能である。文書上のリスナー属性と、アナウンスのターゲットグループ単位の三トリガーが併記されている。有効化のクリック位置は、そのアカウントのコンソールで確認する。

---

## 9. 出典

確認日は 2026-10-11。ページの「最終更新」は AWS 側の表示に従う。

- API Gateway REST のクォータ。統合タイムアウトは 50 ミリ秒〜29 秒。Regional と Private は引き上げ可。引き上げはリージョンのスロットル枠に影響し得る。アイドル接続 310 秒は引き上げ不可。文書更新表示 2026-10-10。
  https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-execution-service-limits-table.html
- 統合タイムアウトを 29 秒より伸ばす手順（Regional / Private REST のみ）。
  https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html
- 2024-06-04、29 秒を超える統合タイムアウトの発表。
  https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-api-gateway-integration-timeout-limit-29-seconds/
- レスポンスのストリーミングは、バッファしない別モード。Regional / Private の無通信は 5 分、最長 15 分という説明がある。今回の同期 API の推奨には使わない。文書更新表示 2026-10-10。
  https://docs.aws.amazon.com/apigateway/latest/developerguide/response-transfer-mode.html
- ALB のアイドル初期値 60 秒、範囲 1〜4000。アプリのアイドルを ALB より長くする、という説明。短くすると 502。
  https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html
- ALB 属性。`client_keep_alive.seconds` 初期値 3600、範囲 60〜604800。
  https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_LoadBalancerAttribute.html
- ALB の 504 に、ターゲット接続確立 10 秒と、アイドルまでの無応答が含まれる。460 はクライアントが先に閉じたとき。
  https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html
- GWLB の TCP アイドル。初期値 350 秒、範囲 60〜6000。350 秒を超えるときはターゲット ENI の確立 TCP 追跡を合わせる。
  https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/update-idle-timeout.html
- 2024-09-05、GWLB の TCP アイドルが変更可能になった発表。それ以前は 350 秒固定。
  https://aws.amazon.com/about-aws/whats-new/2024/09/aws-gateway-load-balancer-tcp-idle-timeout/
- 2026-09-03、GWLB の TCP RST。初期値では無効。
  https://aws.amazon.com/about-aws/whats-new/2026/09/aws-gateway-load-balancer-tcp-reset/
- RDS Proxy の接続設定。放置クライアント初期値 1800 秒。借り出し初期値 120 秒。MySQL の `MaxConnectionsPercent` 初期値 100。`MaxIdleConnectionsPercent` 初期値 50。バックエンドの遊休判定は 5 分。複数 Proxy の合計は 100 以下。
  https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-best-practices.configuration.html
  https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-connections.html
- ピンニングの条件と、追跡する変数、初期化クエリ。
  https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-pinning.html
- Aurora Serverless v2 の `max_connections` は最大 ACU 基準。MySQL の表（1 ACU で 90、8 ACU で 1000、16 ACU で 2000）。
  https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html
- `aurora_transaction_timeout` は Aurora MySQL 8.4.8 以降。初期値 0。8.4.7 では使えない。
  https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.TransactionTimeout.html
- Aurora は `wait_timeout` と `interactive_timeout` の最小値を、すべての放置セッションに使う。
  https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html
- MySQL 8.4 の `connect_timeout` 初期値 10 秒、`innodb_lock_wait_timeout` 初期値 50 秒、`max_execution_time` 初期値 0、`wait_timeout` 初期値 28800 秒。`max_execution_time` は SELECT のみ。1205 のとき初期設定では文だけが戻る。
  https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html
- Connector/J のネットワーク属性。`connectTimeout` 0、`socketTimeout` 0、`tcpKeepAlive` true。`useServerPrepStmts` false、`cachePrepStmts` false。
  https://dev.mysql.com/doc/connector-j/en/connector-j-reference-configuration-properties.html
- JBoss EAP 8.1 のトランザクション `default-timeout` は 300 秒。`maximum-timeout` は 31536000 秒。
  https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/8.1/html/configuration_guide/transaction-manager-configuration-options_default
- EAP 8.1 Undertow。`no-request-timeout` の表の値は 60000 ミリ秒。`read-timeout` と `write-timeout` は空欄。
  https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/8.1/html-single/configuration_guide/index
- データソースの `blocking-timeout-millis` 初期値 30000 は IronJacamar / WildFly のデータソーススキーマの説明。`idle-timeout-minutes` 未指定時 30 分、`query-timeout` 未指定時はタイムアウトなし、は Red Hat の datasource 属性説明。EAP 8.1 では CLI の include-defaults で実値を確認する。
- Palo Alto のセッションタイムアウト。TCP 確立後の初期値 3600 秒、ハンドシェイク前 10 秒、ハーフクローズ 120 秒。
  https://docs.paloaltonetworks.com/ngfw/networking/session-settings-and-timeouts/configure-session-timeouts
- GWLB 350 秒と Palo Alto の無言クローズの組み合わせで、FIN が届かなくなる事例。ファイアウォールのタイムアウト短縮だけでは両端が気づけない、という Palo Alto の説明。
  https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA14u000000kFMPCA2

---

## 10. 一枚に畳むと

画面の 1 回は、10 秒でロックを諦め、20 秒で SELECT を止め、22 秒でトランザクションを戻し、26 秒でソケットを保険として切り、29 秒で API Gateway が 504 を返し、ブラウザは 35 秒待つ。ALB は 60 秒、Undertow の無言は 70 秒にして、502 の同着を避ける。

無言の JDBC 接続は、プールが約 5〜8 分、オンライン Proxy が 15 分、バッチ Proxy が 2 時間、Aurora が 3 時間で手放す。

バッチは JBoss の時計を使わない。`socketTimeout` は無制限のまま、EC2 の keepalive を約 110 秒で死活できるように短くし、ロック待ちだけバッチ Proxy の初期化クエリで 120 秒にする。

8.4.7 は、トランザクションそのもののサーバ側タイマーをまだ持っていない。長いバッチトランザクションを短く切る装置は、チャンクのコミットと、ジョブスケジューラの打ち切りである。
