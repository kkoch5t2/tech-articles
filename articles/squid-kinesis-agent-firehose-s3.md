---
title: "SquidのアクセスログをKinesis AgentとAmazon Data FirehoseでS3へ集約する"
emoji: "📦"
type: "tech"
topics: ["aws", "squid", "kinesis", "firehose", "cloudformation"]
published: false
---

PrivateLink経由で使うSquidプロキシに、アクセスログの収集・保存を追加しました。EC2上のAmazon Kinesis Agentが実際の`access.log`を読み、Amazon Data Firehoseを通じてS3へ保存します。

今回は、Kinesis Agentの`aggregatedRecordSizeBytes`を変えたとき、複数行がどのように送信レコードになるのかも確認します。対象は実際のプロキシ通信で生成されたログです。ログ送信にはPythonによる自作処理を使っていません。

別アカウントからプロキシを使う仕組みと、Firehoseのレコード単位の料金については、次の記事で説明しています。

https://zenn.dev/kkoch5t2/articles/aws-privatelink-squid-proxy

https://zenn.dev/kkoch5t2/articles/firehose-direct-put-cost

## 全体構成と今回追加するリソース

### 通信経路とログ経路

アカウントBのLambdaから、Interface VPC Endpoint、アカウントAの内部NLB、Squidを通じて、`checkip.amazonaws.com`へHTTPS接続します。この通信経路は既存構成と同じです。

追加するのは、各Squid EC2からS3までのログ経路です。2台のKinesis Agentが共通のFirehoseストリームへ送り、保存先のS3バケットも共通にします。

![通信経路とログ収集経路を分けた構成図](/images/squid-agent/architecture.png)
*検証構成の再現図です。既存のプロキシ通信を実線、今回追加したログ経路をオレンジの破線で示しています。SquidとKinesis Agentは同じEC2内で動かします。*

Kinesis AgentはEC2の公開IPv4とInternet Gatewayを使い、Firehoseの公開APIへHTTPSで接続します。今回はFirehose用のInterface Endpointを追加していません。

| 配置 | リソース・設定 |
|---|---|
| A側EC2 | Amazon Linux 2023、t3.microを2AZに1台ずつ。SquidとKinesis Agentを同居 |
| A側ログ保存 | Direct PUTのFirehose 1本、非公開のS3バケット1個、配信用IAMロール |
| B側 | 外部へのデフォルトルートがないVPC、2AZのInterface Endpoint、テスト用Lambda |
| Kinesis Agentの認証 | EC2インスタンスロール。対象ストリームの`firehose:PutRecordBatch`を許可 |
| 収集対象 | `/var/log/squid/access.log*`。CONNECT通信を記録し、非圧縮の旧ログも追跡 |

NLBのTCPヘルスチェックもSquidのログに現れたため、比較に混ざらないよう、今回の`access_log`には既存の`CONNECT` ACLを指定しました。Squidが許可する接続は、従来どおり`checkip.amazonaws.com:443`だけです。

### 検証の範囲と費用

リージョンはシドニー（`ap-southeast-2`）です。検証全体の上限を1,000アクセス・Squidログ10MiBとし、100万行の料金検証は行いません。

EC2×2、NLB、B側Endpointの2AZ分、公開IPv4×2、8GBのgp3×2を合わせた固定部分は、料金APIで確認した単価では**約$0.09/時、約$2.15/日**です。これに使用NLCU、通信、Firehose、S3、Lambdaなどの従量料金が加わります。無料枠・割引・税は含めていません。

Kinesis AgentのCloudWatch独自メトリクスは`cloudwatch.emitMetrics: false`で無効にし、比較にはFirehose標準メトリクスを使います。

## CloudFormationでログ収集環境を構築する

### テンプレートの分け方

既存のプロキシ検証用テンプレートは変更せず、リポジトリの`logging/`配下に拡張版を用意しました。

| 順序 | テンプレート | 内容 |
|---|---|---|
| 1 | `00-log-storage.json` | A側のS3、Firehose、配信用IAMロール |
| 2 | `01-provider-logging.json` | A側の2AZプロキシ構成とKinesis Agentの導入・設定・起動 |
| 3 | `02-consumer-logging.json` | B側のVPC、Endpoint、テスト用Lambda |
| 4 | Endpoint接続の承認 | A側でBの接続要求を確認して承認 |

コードと操作手順は、次のリポジトリにまとめています。

https://github.com/kkoch5t2/aws-privatelink-proxy/tree/feat/squid-kinesis-agent-logging/logging

:::message
初回構築ではKinesis Agentパッケージの旧式のサービス登録方式で起動が止まりました。EC2を保持したままSSMでネイティブのsystemd unitを設定し、未作成の接続リソースは`01b-existing-provider-link.json`で補完しています。配布する拡張版UserDataには修正を取り込んでいますが、修正後テンプレートによる環境全体の新規作成を追加で繰り返したわけではありません。実際に使ったスタックと補完手順はREADME・証跡に記録しています。
:::

操作例ではA側のAWSプロファイルを`provider`、B側を`consumer`としています。各作成の間に状態を確認し、先行するスタックが完成してから次へ進みます。保持環境で使った補完スタックの指定はREADMEを参照してください。

```powershell
# A・Bの認証先と既存リソースを確認する
python logging/labctl.py preflight

# A側のログ保存先を作成する
python logging/labctl.py storage

# CREATE_COMPLETEを確認してから次の作成へ進む
python logging/labctl.py status

# A側のプロキシとKinesis Agentを作成する
python logging/labctl.py provider

# A側の完成後、B側の利用環境を作成する
python logging/labctl.py consumer

# B側Endpointの接続要求をA側で承認する
python logging/labctl.py accept
```

### AL2023へのKinesis Agent導入とサービス管理

AL2023の公式リポジトリにある`aws-kinesis-agent`を使用します。今回のEC2では`2.0.14-1.amzn2023`、Squidは`6.13-1.amzn2023.0.7`でした。JavaはCorretto 17を指定しています。

```bash
# AL2023のリポジトリからJavaとKinesis Agentを導入する
sudo dnf -y install java-17-amazon-corretto-headless
sudo dnf -y install squid aws-kinesis-agent jq

# ネイティブunitで管理するKinesis Agentの状態を確認する
sudo systemctl status aws-kinesis-agent --no-pager
```

UserDataは`/etc/aws-kinesis/agent.json`とsystemd unitを作り、自動起動を有効にします。Kinesis AgentにはSquidログを読むグループ権限を付け、新しく作られるログも`0640 squid:squid`で読めるようにしました。

チェックポイントは再起動後も残る`/var/lib/aws-kinesis-agent/checkpoints`に置きます。収集位置の記録を消して最初から送り直す方法は使いません。

### Kinesis AgentとFirehoseの設定

```json:agent.json（設定例）
{
  "cloudwatch.emitMetrics": false,
  "checkpointFile": "/var/lib/aws-kinesis-agent/checkpoints",
  "firehose.endpoint": "https://firehose.ap-southeast-2.amazonaws.com",
  "flows": [{
    "filePattern": "/var/log/squid/access.log*",
    "deliveryStream": "YOUR_STREAM_NAME",
    "initialPosition": "START_OF_FILE",
    "aggregatedRecordSizeBytes": 5120,
    "maxBufferAgeMillis": 60000,
    "maxBufferSizeBytes": 4194304,
    "maxBufferSizeRecords": 500,
    "minTimeBetweenFilePollsMillis": 100
  }]
}
```

`initialPosition`は、チェックポイントがないファイルを最初に読む位置です。既存のチェックポイントがある場合の継続位置とは分けて考えます。

![EC2から取得した初期Kinesis Agent設定の画面](/images/squid-agent/agent-config-initial.png)
*EC2から取得した実際の初期設定をNAB7のエディタで表示した画面。集約は0から比較を開始しています。*

Firehose側はS3向けのバッファを60秒・5MiBとし、圧縮、データ変換、動的パーティショニング、区切り文字の追加は無効にしています。改行はSquidログに含まれるものをそのまま送ります。

![Firehoseのバッファと圧縮設定の実画面](/images/squid-agent/firehose-buffer.png)
*検証中に撮影したFirehoseの設定画面。60秒・5MiBはS3配信側の設定です。*

![S3送信先・改行の区切り文字・動的パーティショニングの設定](/images/squid-agent/firehose-destination.png)
*S3送信先の実画面。改行の区切り文字の追加と、動的パーティショニングは無効です。*

![Firehoseのデータ変換が無効である設定画面](/images/squid-agent/firehose-transform.png)
*検証中に撮影した変換設定。Lambdaによる変換とレコード形式変換を使用していません。*

## Squid経由で通信してアクセスログを生成する

### 両方のEC2を通ることを確認する

NLBの両ターゲットが`healthy`、B側Endpointが`available`であることを確認してから、LambdaでHTTPS通信を発生させます。今回のテスト用LambdaはEndpointのAZ別DNS名を選択できるため、各AZのSquidを個別に通して確認しました。

```powershell
# AZ1のSquidを通して少量の実通信を行う
python logging/labctl.py generate --route az1 --count 3 --run smoke-az1

# AZ2のSquidを通して同じ通信を行う
python logging/labctl.py generate --route az2 --count 3 --run smoke-az2
```

この管理スクリプトはLambdaの実行と証跡の保存を行います。Firehoseにログを送るのは、EC2上のKinesis Agentです。

### ログに送信元とリクエストIDを残す

Squidのログフォーマットに`source=az1/az2`、EC2のインスタンスID、検証用のリクエストIDを追加しました。

![S3へ到着した両EC2の実際のSquidログ](/images/squid-agent/s3-log-content-capture.png)
*保存済みのS3ログ原本から、AZ1・AZ2の実ログを1行ずつNAB7で表示して撮影しました。IPアドレスとEC2 IDをマスクし、読みやすく折り返しています。架空ログや再送データではありません。*

HTTPSのURLパスはCONNECTの中のTLS通信に含まれ、Squidには見えません。検証用IDはCONNECT時の`X-Lab-Request`ヘッダーに入れ、Squidの`%{X-Lab-Request}>h`で記録しています。

初回の少量疎通では6アクセス中、最初の1件がタイムアウトし、その後の5件は成功しました。タイムアウトした通信を自動再送したり、設定変更なしに原因を断定したりはしていません。集約比較は、この初回疎通と区間を分けて行っています。

## S3に保存されたログを確認する

### S3オブジェクトと入力レコードは別の単位

S3には、Firehoseの配信バッファに応じて作られたオブジェクトが保存されます。**S3ファイルが1個になったことだけでは、Kinesis Agentが1レコードに集約したとは判断できません。** 複数のFirehose入力レコードも、同じS3ファイルに入ります。

![集約比較中のS3到着オブジェクト](/images/squid-agent/s3-arrival-comparison.png)
*集約比較中に撮影した保存先の実画面。オブジェクト数と、Firehoseの入力レコード数は異なります。*

### 原本同士を照合する

EC2の現行ログとローテーション済みログ、S3に届いたオブジェクトをそれぞれ原本のまま保存します。改行を含む行の内容と出現回数を比較し、同一行の重複も確認します。

```powershell
# S3オブジェクト、EC2ログ、設定、Kinesis Agentログを原本として保存する
python logging/labctl.py collect --run final

# 改行を含む行単位の一致と、成功したリクエストIDの到着を確認する
python logging/verify_logs.py --snapshot final
```

| 確認項目 | 最終結果 |
|---|---:|
| EC2原本 / S3原本 | どちらも802行・147,591 B |
| S3のログオブジェクト | 11個 |
| 欠損行 / 余分な行 | 0行 / 0行 |
| 成功した通信のID | 785件すべてが1回ずつ到着 |
| 改行 | 全行LFを保持 |
| 送信元 | AZ1・AZ2とも401行 |

![EC2原本とS3原本の再照合結果](/images/squid-agent/verification-final-capture.png)
*保存済みのEC2・S3ログ原本をNAB7で再照合した結果画面です。802行・147,591 Bの一致、欠損・余分な行0、成功ID785件を確認しています。この照合で追加通信は発生させていません。*

全802行には、CONNECTに収集対象を絞る前のヘルスチェック17行も含まれます。これらも原本として保持し、集約比較の表には混ぜていません。

ログの全体順序は、複数EC2から共通ストリームへ送る構成では判定基準にしていません。送信元の識別と、行の内容・回数・改行を照合しています。

## Kinesis Agentによる集約効果を確認する

### 待機時間・APIバッチ・行の集約を区別する

| 設定・動作 | 役割 |
|---|---|
| `aggregatedRecordSizeBytes` | パーサーが読んだ完全な行を、指定バイト数を目安に1レコードへ連結する |
| `maxBufferAgeMillis` | 作成済みのレコードを、送信バッファに保持する時間の上限 |
| `PutRecordBatch` | 複数の入力レコードを1回のAPIリクエストに載せる |
| Firehoseの60秒・5MiB | 受信後、S3へ配信する側のバッファ設定 |

`aggregatedRecordSizeBytes`は、指定サイズに達するまで後続の行を待つタイマーではありません。読み込んだ範囲に完全な行が1行しかなければ、サイズを大きくしても、その1行でレコードができる場合があります。

このため、送信待機を60秒にしても「60秒分が必ず1レコードになる」とは限りません。設定値を目標サイズと読み替えず、実際の`IncomingRecords`を確認します。

### 通常の追記と、蓄積したログの読み込みを比較する

各方式は、各AZのEC2へ順に60アクセスずつ、合計120アクセスでそろえました。ファイルのポーリングは100ms、送信バッファの時間上限は60秒のままにしています。

「通常の追記」ではKinesis Agentを動かしながらログを発生させます。「蓄積後」ではKinesis Agentを正常停止した状態で同じ件数のログを発生させ、開始後に読み込ませます。この2つは別の条件として比較します。

| 読み込み条件 | 集約サイズ | アクセス数 | IncomingRecords | IncomingBytes | 平均B/レコード |
|---|---:|---:|---:|---:|---:|
| 通常の追記 | 0 | 120 | 120 | 22,120 B | 184.3 B |
| 通常の追記 | 5,120 B | 120 | 120 | 22,240 B | 185.3 B |
| 通常の追記 | 51,200 B | 120 | 120 | 22,120 B | 184.3 B |
| 蓄積後に開始 | 0 | 120 | 120 | 21,940 B | 182.8 B |
| 蓄積後に開始 | 5,120 B | 120 | 6 | 22,180 B | 3,696.7 B |
| 蓄積後に開始 | 51,200 B | 120 | 2 | 22,060 B | 11,030.0 B |

![6条件のCloudWatch Sum・Kinesis Agent送信成功・EC2原本の比較画面](/images/squid-agent/comparison-sums-capture.png)
*初回検証時のCloudWatch API応答とKinesis Agent送信成功記録・EC2原本の照合結果を、NAB7で表示して撮影しました。`l`は通常追記、`b`は蓄積後の開始です。画面下部は51,200 B設定の保存済みAPI応答から、時刻・Sum・単位を抜粋しています。*


通常の追記では、集約サイズを5,120 B、51,200 Bにしても120レコードのままでした。今回の追記間隔と100msのポーリングでは、読み込む範囲に1行ずつ現れるためです。送信バッファに60秒保持する動作が、その後の行との再集約を行うわけではありません。

一方、停止中に蓄積したログを読む条件では、5,120 Bで6レコード、51,200 Bで2レコードになりました。サイズの目安を大きくすると1レコードに載る行は増えましたが、51,200 B設定でも実際の平均は11,030 Bです。指定値まで埋まることを保証する設定ではありません。

Squidログの内容は可変なので、同じアクセス数・条件でも入力バイト数は一定ではありません。表では実際のバイト数も並べています。蓄積後の結果を、そのまま低頻度のリアルタイム収集の削減率として扱うことはできません。

![Firehose標準モニタリングの実画面](/images/squid-agent/firehose-monitoring-comparison.png)
*集約比較と継続収集の検証後に撮影。画面のグラフは1秒あたりの平均値で、表の数値はAPIで取得した各計測区間のSumです。*

各計測区間の`IncomingBytes`と`IncomingRecords`はSumで取得し、Kinesis Agentが成功応答を記録したレコード数・バイト数、EC2/S3の原本と照合しました。前方式の未送信レコードが次の方式に混ざらないよう、区間を分けています。

### 料金は入力レコードごとの切り上げで計算する

Direct PUTの計算上の課金対象量は、**各入力レコードを5,120 B単位で切り上げて合計**します。`IncomingBytes`の総量を最後に1回だけ切り上げる計算ではありません。

| 条件 | 入力レコード数 | 計算上の課金対象量 | 同条件の集約なし比 |
|---|---:|---:|---:|
| 通常の追記：3設定とも | 120 | 614,400 B | 削減なし |
| 蓄積後：集約なし | 120 | 614,400 B | 基準 |
| 蓄積後：5,120 B | 6 | 30,720 B | 95.0%削減 |
| 蓄積後：51,200 B | 2 | 30,720 B | 95.0%削減 |

51,200 B設定では、Kinesis Agentの成功記録で**10,910 Bと11,150 Bの2レコード**を確認しました。それぞれを15,360 Bへ切り上げるため、合計は30,720 Bです。総受信量22,060 Bを1回だけ切り上げて25,600 Bとする計算は使いません。

5,120 B設定の6レコードも、それぞれ5,120 Bの課金対象となります。今回は、51,200 Bへ変更するとレコード数はさらに減りましたが、切り上げ後の課金対象量は変わりませんでした。

これらは今回の実測に基づく課金対象量の計算です。Cost Explorerでこの検証分だけの料金を測定した数値ではありません。また、前回のPython検証の削減率をKinesis Agentの結果として流用していません。

## ローテーション・再起動・送信失敗への対応

### ログローテーション後も収集を継続する

今回は`copytruncate`ではなく、ファイルのrename、新しいファイルの作成、Squidによる再オープンを使います。Kinesis Agentは`access.log*`を監視するため、旧ファイルも非圧縮のまま残します。

```bash
# ローテーションを実行し、旧ログと新しいログの権限を確認する
sudo logrotate -f /etc/logrotate.d/squid
sudo ls -l /var/log/squid/access.log*
```

`logfile_rotate 0`でSquid自身による二重の連番ローテーションを避け、`create 0640 squid squid`で新ファイルの権限をそろえます。

各EC2でローテーション前後に5アクセスずつ、合計20件を発生させました。旧ファイルと新ファイルの両方を原本として保存し、全20件がS3へ1回ずつ到着することを確認しました。

### Kinesis Agentを再起動する

```bash
# チェックポイントを保持したままKinesis Agentを正常に再起動する
sudo systemctl restart aws-kinesis-agent

# 再起動後の状態と進捗ログを確認する
sudo systemctl status aws-kinesis-agent --no-pager
sudo tail -n 30 /var/log/aws-kinesis-agent/aws-kinesis-agent.log
```

各EC2で正常再起動の前後に5アクセスずつ、合計20件を発生させました。チェックポイントを保持した状態で、全20件が欠損・重複なくS3へ到着しました。確認したのはKinesis Agentプロセスの正常再起動です。

### 送信できない間のログを確認する

接続拒否を模擬するため、Kinesis Agentの送信先を一時的に`https://127.0.0.1:9`へ切り替えました。Squidの通信先やB側のネットワーク設定は変えていません。

両EC2で`Connection refused`と`Retriable send error ... Will retry`を確認しました。送信できない間に生成した10件と、元の設定へ戻してKinesis Agentを再起動した後の10件は、全20件がS3へ1回ずつ到着しています。復旧時にもチェックポイントは削除していません。

![ローテーション・再起動・一時送信失敗の照合結果と実エラーログ](/images/squid-agent/continuity-capture.png)
*保存済みの送信記録とS3原本を照合し、各20件が1回ずつ到着したことを確認した画面です。下部は初回検証で両EC2に残った実エラーログの抜粋です。保存済み証跡をNAB7で表示して撮影しており、障害を再発生させた画像ではありません。*

今回の結果だけで、異常終了や応答喪失を含むすべての条件で重複が発生しないとは説明できません。収集位置の記録、送信の再試行、S3到着データを合わせて確認し、必要なら下流でリクエストIDなどによる重複排除を設計します。

検証環境とログは確認のため保持しています。S3にライフサイクル削除は設定せず、EC2のログやKinesis Agentログも自動期限切れにしていません。環境を保持する間は、EC2・NLB・Endpointなどの課金が続きます。


**参考資料**

- [AWS：Kinesis Agentの設定](https://docs.aws.amazon.com/firehose/latest/dev/agent-config-settings.html)
- [AWS：Kinesis Agentの導入](https://docs.aws.amazon.com/firehose/latest/dev/download-install.html)
- [AWS：Firehoseの料金](https://aws.amazon.com/firehose/pricing/)
- [Amazon Kinesis Agentの公開集約実装](https://github.com/awslabs/amazon-kinesis-agent/blob/aa0d9089ed6b0ce9c10871f403bfcf20b74a461f/src/com/amazon/kinesis/streaming/agent/tailing/AggregationSplitter.java)
