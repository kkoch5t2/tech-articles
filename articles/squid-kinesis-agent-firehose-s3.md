---
title: "SquidのアクセスログをKinesis AgentとAmazon Data FirehoseでS3へ集約する"
emoji: "📦"
type: "tech"
topics: ["aws", "squid", "kinesis", "firehose", "cloudformation"]
published: false
---

PrivateLink経由で使うSquidプロキシに、アクセスログの収集・保存を追加しました。EC2上のAmazon Kinesis Agentが実際の`access.log`を読み、Firehose用のInterface VPC Endpointを経由してAmazon Data Firehoseへ送り、S3へ保存します。

今回は、Kinesis Agentの`aggregatedRecordSizeBytes`を変えたとき、複数行がどのように送信レコードになるのかも確認します。対象は実際のプロキシ通信で生成されたログです。ログ送信にはPythonによる自作処理を使っていません。

別アカウントからプロキシを使う仕組みと、Firehoseのレコード単位の料金については、次の記事で説明しています。

https://zenn.dev/kkoch5t2/articles/aws-privatelink-squid-proxy

https://zenn.dev/kkoch5t2/articles/firehose-direct-put-cost

## 全体構成と今回追加するリソース

### 通信経路とログ経路

アカウントBのLambdaから、Interface VPC Endpoint、アカウントAの内部NLB、Squidを通じて、`checkip.amazonaws.com`へHTTPS接続します。この通信経路は既存構成と同じです。

追加するのは、各Squid EC2からS3までのログ経路です。A側VPCの2AZにFirehose用Interface VPC EndpointのENIを配置し、2台のKinesis Agentから共通のFirehoseストリームへ送ります。保存先のS3バケットも共通です。

![通信経路とログ収集経路を分けた構成図](/images/squid-agent/architecture-overview.png)
*上段はプロキシ通信、下段は追加したログ収集の経路です。上下に描いているのは同じ2台のEC2で、SquidとKinesis Agentを同居させています。*

ログ送信は **Kinesis Agent → Firehose用Interface Endpoint → Firehose → S3** です。Kinesis AgentからFirehoseへの通信には、EC2の公開IPv4やInternet Gatewayを使いません。

一方、Squidが外部サイトへ接続する通信には、従来どおり公開IPv4とInternet Gatewayを使います。プロキシの外部接続と、AWSへのログ送信は別の経路です。

| 配置 | リソース・設定 |
|---|---|
| A側EC2 | Amazon Linux 2023、t3.microを2AZに1台ずつ。SquidとKinesis Agentを同居 |
| A側ログ送信 | Firehose用Interface Endpoint 1個・2AZのENI、Private DNS有効、専用セキュリティグループ |
| A側ログ保存 | Direct PUTのFirehose 1本、非公開のS3バケット1個、配信用IAMロール |
| B側 | 外部へのデフォルトルートがないVPC、2AZのInterface Endpoint、テスト用Lambda |
| Kinesis Agentの認証 | EC2インスタンスロール。対象ストリームへの`firehose:PutRecordBatch`を、このFirehose用Endpoint経由に限定 |
| 収集対象 | `/var/log/squid/access.log*`。CONNECT通信を記録し、非圧縮の旧ログも追跡 |

NLBのTCPヘルスチェックもSquidのログに現れたため、比較に混ざらないよう、今回の`access_log`には既存の`CONNECT` ACLを指定しました。Squidが許可する接続は、従来どおり`checkip.amazonaws.com:443`だけです。

### 検証の範囲と費用

リージョンはシドニー（`ap-southeast-2`）です。検証全体の上限を1,000アクセス・Squidログ10MiBとし、100万行の料金検証は行いません。

EC2×2、NLB、B側EndpointとFirehose用Endpointの各2AZ分、公開IPv4×2、8GBのgp3×2を合わせた固定部分は、料金APIで確認した単価では**約$0.116/時、約$2.78/日**です。これに使用NLCU、通信、Firehose、S3、Lambdaなどの従量料金が加わります。無料枠・割引・税は含めていません。

Firehose用Endpointの固定料金は、シドニーでは1AZあたり$0.013/時、2AZで$0.026/時です。これとは別にEndpointのデータ処理料金も発生します。

Kinesis AgentのCloudWatch独自メトリクスは`cloudwatch.emitMetrics: false`で無効にし、比較にはFirehose標準メトリクスを使います。

## CloudFormationでログ収集環境を構築する

### テンプレートの分け方

既存のプロキシ検証用テンプレートは変更せず、リポジトリの`logging/`配下に拡張版を用意しました。

| 順序 | テンプレート | 内容 |
|---|---|---|
| 1 | `00-log-storage.json` | A側のS3、Firehose、配信用IAMロール |
| 2 | `01-provider-logging.json` | A側の2AZプロキシ構成、Firehose用Endpoint、Kinesis Agentの導入・設定・起動 |
| 3 | `02-consumer-logging.json` | B側のVPC、Endpoint、テスト用Lambda |
| 4 | Endpoint接続の承認 | A側でBの接続要求を確認して承認 |

コードと操作手順は、次のリポジトリにまとめています。

https://github.com/kkoch5t2/aws-privatelink-proxy/tree/feat/squid-kinesis-agent-logging/logging

操作例ではA側のAWSプロファイルを`provider`、B側を`consumer`としています。各作成の間に状態を確認し、先行するスタックが完成してから次へ進みます。

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

### Firehoseへの送信をPrivateLinkに限定する

![EC2からFirehose用Endpointへ送信する経路と3か所のアクセス制御](/images/squid-agent/firehose-privatelink-detail.png)
*この節専用の構成図です。通常のFirehoseサービス名をPrivate DNSでEndpoint ENIへ解決し、SG・Endpointポリシー・EC2ロールで送信を制御します。*

Firehose用EndpointはA側VPCに作ります。B側からSquidへ接続するためのEndpointとは、配置先も接続先も異なります。

| Endpoint | 配置 | 接続先 | ポート |
|---|---|---|---|
| プロキシ用 | B側VPC・2AZ | A側のEndpoint Service / 内部NLB | TCP 3128 |
| Firehose用 | A側VPC・2AZ | Amazon Data Firehose | TCP 443 |

`01-provider-logging.json`では、次の設定でFirehose用Endpointを作成します。ポリシーも同じテンプレートに定義しています。

```json:01-provider-logging.json（抜粋）
{
  "FirehoseEndpoint": {
    "Type": "AWS::EC2::VPCEndpoint",
    "Properties": {
      "VpcEndpointType": "Interface",
      "VpcId": {
        "Ref": "ProviderVpc"
      },
      "ServiceName": {
        "Fn::Sub": "com.amazonaws.${AWS::Region}.kinesis-firehose"
      },
      "PrivateDnsEnabled": true,
      "SubnetIds": [
        {
          "Ref": "PublicSubnetAz1"
        },
        {
          "Ref": "PublicSubnetAz2"
        }
      ],
      "SecurityGroupIds": [
        {
          "Ref": "FirehoseEndpointSecurityGroup"
        }
      ]
    }
  }
}
```

Private DNSを有効にすると、EC2内で`firehose.ap-southeast-2.amazonaws.com`を名前解決した結果がEndpoint ENIのプライベートIPになります。Kinesis Agentの`firehose.endpoint`は、この通常のサービスURLを使います。URLの文字列だけで公開経路かどうかを判断するのではなく、EC2からの名前解決も確認します。通常のサービス名が返すIPは2AZのENIに対応しますが、常に同一AZのENIを選ぶ設定ではありません。

![Firehose用Interface EndpointのAWSコンソール画面](/images/squid-agent/firehose-endpoint-console.png)
*今回作成したFirehose用Endpointの実画面です。使用可能（available）、Private DNS有効、2AZのサブネットとENIのプライベートIPを確認しています。一部のリソース識別子をマスクしています。*

アクセス制御は次の3か所に設定しました。

- Endpointのセキュリティグループ：Squid EC2のセキュリティグループからTCP 443だけを許可する。
- Endpointポリシー：EC2インスタンスロールによる、対象ストリームへの`firehose:PutRecordBatch`だけを許可する。
- EC2インスタンスロール：`aws:SourceVpce`条件で、このFirehose用Endpointからの送信だけを許可する。

```json:EC2インスタンスロールの送信権限（抜粋）
{
  "Effect": "Allow",
  "Action": "firehose:PutRecordBatch",
  "Resource": {
    "Fn::Sub": "arn:${AWS::Partition}:firehose:${AWS::Region}:${AWS::AccountId}:deliverystream/${FirehoseStreamName}"
  },
  "Condition": {
    "StringEquals": {"aws:SourceVpce": {"Ref": "FirehoseEndpoint"}}
  }
}
```

このロールのFirehose送信権限はEndpoint経由に限定しています。Private DNSの設定が崩れて公開APIへ向かった場合も、送信権限が成立しない構成です。

### Amazon Linux 2023へのKinesis Agent導入とサービス管理

![Amazon Linux 2023でのKinesis Agent導入とsystemdによるサービス管理](/images/squid-agent/amazon-linux-agent-service.png)
*この節専用の構成図です。UserDataでパッケージ・設定・権限を用意し、systemdでSquidとKinesis Agentを管理します。Agentはログを読み取り、収集位置をチェックポイントへ保存します。*

Amazon Kinesis Agentは、EC2上で常駐するJava製のログ収集サービスです。本構成ではKinesis Data Streamsを挟まず、Firehoseへ直接送信します。

Amazon Linux 2023の公式リポジトリにある`aws-kinesis-agent`を使用します。今回のEC2では`2.0.14-1.amzn2023`、Squidは`6.13-1.amzn2023.0.7`でした。JavaはCorretto 17を指定しています。

```bash
# Amazon Linux 2023のリポジトリからJavaとKinesis Agentを導入する
sudo dnf -y install java-17-amazon-corretto-headless
sudo dnf -y install squid aws-kinesis-agent jq

# ネイティブunitで管理するKinesis Agentの状態を確認する
sudo systemctl status aws-kinesis-agent --no-pager
```

**EC2側で実行した確認コマンド（bash）**

構築直後、SSM経由で次のコマンドを実行しました。

```bash
cloud-init status --wait
systemctl is-active squid aws-kinesis-agent
systemctl is-enabled squid aws-kinesis-agent
systemctl show aws-kinesis-agent -p FragmentPath -p ExecMainStatus -p NRestarts
rpm -q squid aws-kinesis-agent
```

**Windowsで保存済みの実行結果を確認する（PowerShell）**

上のbashはEC2側の操作です。以降のターミナル画像は、WindowsからSSM履歴や保存ファイルを取得した画面です。各画像の直前には、画像内で実行しているコマンドをそのまま掲載します。

撮影した端末でA側の認証に使ったプロファイル名は`medallion-lab`です。画像と取得コマンドはこの名前でそろえています。構築手順の例にある`provider`を使う場合は、プロファイル名を置き換えてください。

まず、各画面で使用する共通変数を設定します。これは取得コマンドの事前設定です。

```powershell
# 共通設定。IDとプロファイル名は自分の環境の値に置き換える
$awsA=@('--profile','medallion-lab','--region','ap-southeast-2','--no-cli-pager')
$ssm=@('--profile','medallion-lab','--region','ap-southeast-2','--command-id','YOUR_INITIAL_INSPECTION_COMMAND_ID','--no-cli-pager')
$az1='YOUR_AZ1_INSTANCE_ID'
$az2='YOUR_AZ2_INSTANCE_ID'
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
aws ssm get-command-invocation @ssm --instance-id $az1 --query "[Status,ExecutionStartDateTime]" --output text
aws ssm get-command-invocation @ssm --instance-id $az1 --query StandardOutputContent --output text | Select-Object -First 12
aws ssm get-command-invocation @ssm --instance-id $az2 --query "[Status,ExecutionStartDateTime]" --output text
aws ssm get-command-invocation @ssm --instance-id $az2 --query StandardOutputContent --output text | Select-Object -First 12
```

![AWS CLIで両EC2のSSM実行履歴を取得したターミナルの実画面](/images/squid-agent/ssm-history-terminal.png)
*2026年11月16日に、AWS CLIのget-command-invocationで検証時のSSM実行履歴を取得し、実際のWindows Terminalを撮影しました。AZ1・AZ2の成功状態、実行日時、サービス状態とパッケージを確認できます。画像の`@ssm`は共通引数、`$az1`・`$az2`は各EC2 IDです。出力は先頭12行の抜粋です。2026年11月15日の検証時の記録を取得した画面です。*

`active`の2行はSquidとKinesis Agentの稼働、`enabled`の2行は両サービスの自動起動を示します。`FragmentPath`からネイティブunitが読み込まれていることも確認しました。

UserDataは`/etc/aws-kinesis/agent.json`とsystemd unitを作り、自動起動を有効にします。Kinesis AgentにはSquidログを読むグループ権限を付け、新しく作られるログも`0640 squid:squid`で読めるようにしました。

**EC2上で直接確認する場合（bash）**

unitファイルから、実行ユーザー、squidグループへの追加、Corretto 17の指定を確認します。

```bash
sudo cat /etc/systemd/system/aws-kinesis-agent.service
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
$unitOutput = aws ssm get-command-invocation @ssm --instance-id $az1 --query StandardOutputContent --output text
[regex]::Match(($unitOutput -join "`n"),'(?ms)^\[Unit\].*?^WantedBy=multi-user.target').Value
```

![SSM実行履歴からunitファイルを取得した実ターミナル画面](/images/squid-agent/agent-unit-terminal.png)
*AWS CLIで構築直後のSSM実行履歴を取得し、unitファイルの該当部分を表示した実画面です。ユーザー・グループ・Javaの指定と、サービス管理の設定を確認できます。*

チェックポイントは再起動後も残る`/var/lib/aws-kinesis-agent/checkpoints`に置きます。収集位置の記録を消して最初から送り直す方法は使いません。

**EC2上で直接確認する場合（bash）**

ログとチェックポイントの所有者・権限を確認します。構築直後はテストアクセス前のため、`access.log`は0 Bでした。

```bash
sudo ls -l /var/log/squid /var/lib/aws-kinesis-agent
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
aws ssm get-command-invocation @ssm --instance-id $az1 --query StandardOutputContent --output text | Select-Object -Last 10
aws ssm get-command-invocation @ssm --instance-id $az2 --query StandardOutputContent --output text | Select-Object -Last 10
```

![両EC2のログ権限とチェックポイントを取得した実ターミナル画面](/images/squid-agent/agent-permissions-terminal.png)
*構築直後のSSM実行履歴から、両EC2の出力末尾10行を取得した実画面です。squidログの権限と、Agentユーザーが所有するチェックポイントを確認できます。*

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

比較は、EC2から取得した初期設定の`aggregatedRecordSizeBytes: 0`から開始しました。上の設定例は5,120 Bで集約する場合です。

`firehose.endpoint`に指定したURLは、Private DNSによってVPC内のEndpoint ENIへ解決されます。FirehoseからS3への配信はAWSサービス側で行われ、このEC2がS3へ直接ログを送る構成ではありません。

Firehose側はS3向けのバッファを60秒・5MiBとし、圧縮、データ変換、動的パーティショニング、区切り文字の追加は無効にしています。改行はSquidログに含まれるものをそのまま送ります。

![FirehoseのS3配信設定のAWSコンソール画面](/images/squid-agent/firehose-settings-console.png)
*今回のFirehoseのAWSコンソール実画面です。S3配信バッファ60秒・5MiB、圧縮なし、改行の区切り文字追加と動的パーティショニングが無効であることを確認しています。*

## Squid経由で通信してアクセスログを生成する

### 両方のEC2を通ることを確認する

NLBの両ターゲットが`healthy`、B側Endpointが`available`であることを確認してから、LambdaでHTTPS通信を発生させます。今回のテスト用LambdaはEndpointのAZ別DNS名を選択できるため、各AZのSquidを個別に通して確認しました。

疎通前には、対象ターゲットグループとB側Endpointの状態を確認します。ARNとEndpoint IDは自分の環境の値に置き換えます。

```powershell
# A側のNLBターゲット状態を確認する
aws elbv2 describe-target-health --profile provider --region ap-southeast-2 --target-group-arn 'YOUR_TARGET_GROUP_ARN' --query 'TargetHealthDescriptions[].{Instance:Target.Id,Port:Target.Port,State:TargetHealth.State}' --output table

# B側のプロキシ接続用Endpointの状態を確認する
aws ec2 describe-vpc-endpoints --profile consumer --region ap-southeast-2 --vpc-endpoint-ids 'YOUR_CONSUMER_ENDPOINT_ID' --query 'VpcEndpoints[].{State:State,Type:VpcEndpointType}' --output table
```

```powershell
# AZ1のSquidを通して少量の実通信を行う
python logging/labctl.py generate --route az1 --count 3 --run smoke-az1

# AZ2のSquidを通して同じ通信を行う
python logging/labctl.py generate --route az2 --count 3 --run smoke-az2
```

この管理スクリプトはLambdaの実行と証跡の保存を行います。Firehoseにログを送るのは、EC2上のKinesis Agentです。

初回疎通では、各AZで3回ずつすべてHTTP 200となり、送信元パブリックIPもそれぞれのSquid EC2と一致しました。

`$proof`は、検証時のreadiness.jsonとLambda応答JSONを保存したフォルダーです。ここでは通信を発生させるコマンドではなく、保存記録を読み出すコマンドを実行しています。

```powershell
$proof='C:\PATH\TO\SAVED_EVIDENCE'
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
(Get-Content "$proof\readiness.json" -Raw | ConvertFrom-Json).targets.TargetHealthDescriptions | Select-Object @{n='EC2';e={if($_.Target.Id -eq $az1){'AZ1'}else{'AZ2'}}},@{n='Port';e={$_.Target.Port}},@{n='State';e={$_.TargetHealth.State}} | Format-Table
(Get-Content "$proof\readiness.json" -Raw | ConvertFrom-Json).endpoint.VpcEndpoints | Select-Object State,VpcEndpointType | Format-Table
(Get-Content "$proof\az1-response.json" -Raw | ConvertFrom-Json).requests | Select-Object request_id,status | Format-Table
(Get-Content "$proof\az2-response.json" -Raw | ConvertFrom-Json).requests | Select-Object request_id,status | Format-Table
```

![検証時のNLB状態と両AZのLambda成功応答を読み出した実ターミナル画面](/images/squid-agent/both-az-results-terminal.png)
*検証時に保存したreadiness.jsonと両AZのLambda応答ファイルを、実際のPowerShellで読み出した画面です。NLBの2台をAZ1・AZ2として表示し、各AZで3アクセスずつHTTP 200となった記録を確認できます。撮影時点のNLB状態を取得した画面ではなく、疎通直前・初回疎通の保存記録です。*

最初の少量疎通では、各AZで3アクセスずつ、計6アクセスがすべてHTTP 200で成功しました。EC2原本とS3原本はともに6行・1,137 Bで一致し、欠損・余分な行は0でした。集約比較は、この疎通確認と計測区間を分けて行います。

### どのEC2の、どのアクセスのログがS3へ届いたかを確認する

2台のSquidのログは同じS3へ集まります。そこで、**「ログを出したEC2」と「テストで発生させた1回のアクセス」**を区別する情報を、Squidのログに追加しました。

| ログの項目 | 意味 | 誰が付けるか |
|---|---|---|
| `source=az1` / `source=az2` | ログを書いたSquid EC2の区別。アクセスしたクライアントの意味ではありません | 各EC2のSquidログ設定 |
| `instance=...` | そのSquid EC2のインスタンスID | UserDataで取得し、Squidログ設定に追加 |
| `request=...` | テストアクセス1回ごとの識別番号（リクエストID） | テスト用Lambdaが発行 |

たとえば`fresh-20261115-az1-000`は、初回疎通テストでAZ1側を通した最初のアクセスに付けたIDです。この値がLambdaの成功応答とS3ログの両方に残っていれば、**そのアクセスのログがS3まで届いた**と照合できます。

前のLambda応答画面と、次のS3ログ画面で、同じリクエストIDを照合します。

S3のオブジェクト一覧と、初回疎通のログを取得します。画像で実行したコマンドには、表示するIPとEC2 IDをマスクする処理も含めています。

バケット名とオブジェクトのキーを設定します。検索するリクエストIDも、対象の検証に合わせてください。

```powershell
$bucket='YOUR_BUCKET_NAME'
$s3Uri='s3://YOUR_BUCKET_NAME/YOUR_OBJECT_KEY'
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
aws s3api list-objects-v2 @awsA --bucket $bucket --prefix squid/ --query 'Contents[:3].{Key:Key,Size:Size}' --output table
aws s3 cp $s3Uri - --profile medallion-lab --region ap-southeast-2 --no-cli-pager | Select-String -Pattern 'request=fresh-20261115-az[12]-000$' | ForEach-Object { $_.Line -replace '\b(?:\d{1,3}\.){3}\d{1,3}\b','[IP]' -replace 'i-[0-9a-f]{17}','[EC2-ID]' }
```

![S3のオブジェクト一覧と実ログをAWS CLIで取得した実ターミナル画面](/images/squid-agent/s3-logs-terminal.png)
*AWS CLIでS3から実際に取得した画面です。上部が先頭3オブジェクトのキーとサイズ、下部が初回疎通の各AZの最初のログです。IPとEC2 IDは表示時にマスクしています。sourceでEC2を、requestでLambda応答との一致を確認できます。*

HTTPSでは、SquidはCONNECT要求で接続先のホストとポートを受け取りますが、その後のTLS通信内のURLパスは読めません。今回はLambdaがCONNECT時の`X-Lab-Request`ヘッダーへ検証用IDを入れ、Squidの`%{X-Lab-Request}>h`でその値をログに残しています。これは検証の照合用に追加した仕組みです。

## PrivateLink経由の送信とS3保存を確認する

### Firehoseへのプライベート接続を確認する

EC2内でサービス名を名前解決し、Firehose用EndpointのENIに割り当てられたプライベートIPと一致することを確認します。

```bash
# EC2内でFirehoseの名前解決先を確認する
getent ahostsv4 firehose.ap-southeast-2.amazonaws.com
```

今回の両EC2では、名前解決先が2AZのEndpoint ENIのIPと一致しました。Endpointは`available`、Private DNSは有効です。

さらに、配布したEC2ロールの送信ポリシーをIAMポリシーシミュレーターで評価し、`aws:SourceVpce`に今回のEndpointを指定すると許可、条件がない場合は`implicitDeny`となることを確認しました。実際の収集では、Kinesis Agentの送信成功記録とS3に保存されたログも合わせて確認します。

検証時に保存された名前解決結果は、どちらもFirehose用Endpoint ENIのプライベートIP `10.240.1.218`・`10.240.2.201`と一致しています。

WindowsからEndpointの状態・ENIのIPと、両EC2の名前解決の実行履歴を取得します。

Endpoint ID・ENI IDと、名前解決を確認したSSMコマンドIDを設定します。

```powershell
$endpointId='YOUR_FIREHOSE_ENDPOINT_ID'
$eniIds=@('YOUR_ENDPOINT_ENI_1_ID','YOUR_ENDPOINT_ENI_2_ID')
$dnsId='YOUR_DNS_CHECK_SSM_COMMAND_ID'
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
aws ec2 describe-vpc-endpoints @awsA --vpc-endpoint-ids $endpointId --query 'VpcEndpoints[].{State:State,PrivateDnsEnabled:PrivateDnsEnabled}' --output table
aws ec2 describe-network-interfaces @awsA --network-interface-ids $eniIds --query 'NetworkInterfaces[].{AZ:AvailabilityZone,IP:PrivateIpAddress}' --output table
aws ssm get-command-invocation @awsA --command-id $dnsId --instance-id $az1 --query StandardOutputContent --output text | Select-Object -First 1
aws ssm get-command-invocation @awsA --command-id $dnsId --instance-id $az2 --query StandardOutputContent --output text | Select-Object -First 1
```

![Endpointの状態とENIのIPおよび両EC2の名前解決履歴を取得した実ターミナル画面](/images/squid-agent/private-dns-terminal.png)
*上部は撮影時にAWS CLIで取得したEndpointのavailable・Private DNS有効とENIのIP、下部はSSMに残る検証時の両EC2の名前解決結果です。現在のEndpoint情報と、検証時のEC2実行履歴を区別して確認しています。*

| 送信ポリシー単体の評価条件 | 結果 |
|---|---|
| 今回のEndpointを`aws:SourceVpce`に指定 | `allowed` |
| `aws:SourceVpce`の条件なし | `implicitDeny` |

### S3オブジェクトと入力レコードは別の単位

S3には、Firehoseの配信バッファに応じて作られたオブジェクトが保存されます。**S3ファイルが1個になったことだけでは、Kinesis Agentが1レコードに集約したとは判断できません。** 複数のFirehose入力レコードも、同じS3ファイルに入ります。

S3から取得したログオブジェクトは10個です。前のS3取得画面では、先頭3個のキーとサイズを確認できます。これはS3オブジェクトの一覧であり、Firehoseの入力レコード一覧ではありません。

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
| EC2原本 / S3原本 | どちらも786行・146,012 B |
| S3のログオブジェクト | 10個 |
| 欠損行 / 余分な行 | 0行 / 0行 |
| 成功した通信のID | 786件すべてが1回ずつ到着 |
| 改行 | 全行LFを保持 |
| 送信元 | AZ1・AZ2とも393行 |

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
| 通常の追記 | 0 B | 120 | 120 | 22,240 B | 185.3 B |
| 通常の追記 | 5,120 B | 120 | 120 | 22,240 B | 185.3 B |
| 通常の追記 | 51,200 B | 120 | 120 | 22,240 B | 185.3 B |
| 蓄積後に開始 | 0 B | 120 | 120 | 22,180 B | 184.8 B |
| 蓄積後に開始 | 5,120 B | 120 | 6 | 22,240 B | 3,706.7 B |
| 蓄積後に開始 | 51,200 B | 120 | 2 | 22,300 B | 11,150.0 B |

6条件すべてで、CloudWatch APIのSumとKinesis Agentの送信成功記録について、レコード数・バイト数が一致しました。EC2原本のバイト数とも一致しています。

通常の追記では、集約サイズを5,120 B、51,200 Bにしても120レコードのままでした。今回の追記間隔と100msのポーリングでは、読み込む範囲に1行ずつ現れるためです。送信バッファに60秒保持する動作が、その後の行との再集約を行うわけではありません。

一方、停止中に蓄積したログを読む条件では、5,120 Bで6レコード、51,200 Bで2レコードになりました。51,200 B設定でも実際の平均は11,150.0 Bです。指定値まで埋まることを保証する設定ではありません。

Squidログの内容は可変なので、同じアクセス数・条件でも入力バイト数は一定ではありません。表では実際のバイト数も並べています。蓄積後の結果を、そのまま低頻度のリアルタイム収集の削減率として扱うことはできません。

蓄積後の51,200 B設定について、検証区間のメトリクスを取得します。画像内と同じプロファイル名・ストリーム名・期間を示します。自分の環境で実行する場合は、それぞれの値を置き換えてください。

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
$cw=@('--profile','medallion-lab','--region','ap-southeast-2','--namespace','AWS/Firehose','--dimensions','Name=DeliveryStreamName,Value=squidvpce-store-squid','--start-time','2026-11-15T09:49:00Z','--end-time','2026-11-15T09:53:00Z','--period','60','--statistics','Sum','--no-cli-pager')
aws cloudwatch get-metric-statistics @cw --metric-name IncomingRecords --query Datapoints --output json
aws cloudwatch get-metric-statistics @cw --metric-name IncomingBytes --query Datapoints --output json
```

![AWS CLIで検証区間のCloudWatchメトリクスを取得したターミナルの実画面](/images/squid-agent/cloudwatch-cli-terminal.png)
*実際のWindows Terminalで、CloudWatchのget-metric-statisticsを実行して撮影しました。2026年11月15日09:49〜09:53 UTCの検証区間を指定し、IncomingRecordsのSumが2、IncomingBytesのSumが22,300 Bであることを再取得しています。検証時のメトリクスを後から取得した画面です。*

各計測区間の`IncomingBytes`と`IncomingRecords`はSumで取得し、Kinesis Agentが成功応答を記録したレコード数・バイト数、EC2/S3の原本と照合しました。前方式の未送信レコードが次の方式に混ざらないよう、区間を分けています。

### 料金は入力レコードごとの切り上げで計算する

Direct PUTの計算上の課金対象量は、**各入力レコードを5,120 B単位で切り上げて合計**します。`IncomingBytes`の総量を最後に1回だけ切り上げる計算ではありません。

| 条件 | 入力レコード数 | 計算上の課金対象量 | 同条件の集約なし比 |
|---|---:|---:|---:|
| 通常の追記：3設定とも | 120 | 614,400 B | 削減なし |
| 蓄積後：集約なし | 120 | 614,400 B | 基準 |
| 蓄積後：5,120 B | 6 | 30,720 B | 95.0%削減 |
| 蓄積後：51,200 B | 2 | 30,720 B | 95.0%削減 |

51,200 B設定では、Kinesis Agentの成功記録で**11,150 Bのレコードを2つ**確認しました。各レコードを5,120 B単位で切り上げ、15,360＋15,360＝30,720 Bと計算します。総受信量を最後に1回だけ切り上げる計算は使いません。

5,120 B設定の6レコードは、それぞれ5,120 Bの課金対象となります。今回は、51,200 Bへ変更するとレコード数はさらに減りましたが、切り上げ後の課金対象量は変わりませんでした。

これらは今回の実測に基づく課金対象量の計算です。Cost Explorerでこの検証分だけの料金を測定した数値ではありません。また、前回のPython検証の削減率をKinesis Agentの結果として流用していません。

## ローテーション・再起動・送信失敗への対応

### ログローテーション後も収集を継続する

今回は`copytruncate`ではなく、ファイルのrename、新しいファイルの作成、Squidによる再オープンを使います。Kinesis Agentは`access.log*`を監視するため、旧ファイルも非圧縮のまま残します。

**EC2上で行う操作（bash）**

```bash
# ローテーションを実行し、旧ログと新しいログの権限を確認する
sudo logrotate -f /etc/logrotate.d/squid
sudo ls -li /var/log/squid/access.log*
```

`logfile_rotate 0`でSquid自身による二重の連番ローテーションを避け、`create 0640 squid squid`で新ファイルの権限をそろえます。

`ls -li`の左端のinodeが異なり、新旧のファイルが別々に存在することを確認しました。両方ともsquid:squid・0640です。

ローテーションを行ったSSMコマンドIDを指定します。画像はその実行履歴を取得したものです。

```powershell
$rotateId='YOUR_ROTATION_SSM_COMMAND_ID'
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
aws ssm get-command-invocation @awsA --command-id $rotateId --instance-id $az1 --query StandardOutputContent --output text
aws ssm get-command-invocation @awsA --command-id $rotateId --instance-id $az2 --query StandardOutputContent --output text
```

![両EC2のローテーション後のファイル状態をSSM履歴から取得した実画面](/images/squid-agent/rotation-history-terminal.png)
*AWS CLIでローテーション直後のSSM実行履歴を取得した実ターミナル画面です。各EC2に、新しいaccess.logと、inodeが異なる日付付きの旧ログがあることを確認できます。*

各EC2でローテーション前後に5アクセスずつ、合計20件を発生させました。旧ファイルと新ファイルの両方を原本として保存し、全20件がS3へ1回ずつ到着することを確認しました。

### Kinesis Agentを再起動する

**EC2上で行う操作（bash）**

```bash
# チェックポイントを保持したままKinesis Agentを正常に再起動する
sudo systemctl restart aws-kinesis-agent

# 再起動後の状態と進捗ログを確認する
sudo systemctl status aws-kinesis-agent --no-pager
sudo tail -n 30 /var/log/aws-kinesis-agent/aws-kinesis-agent.log
```

再起動を行ったSSMコマンドIDを指定します。画像はその実行履歴を取得したものです。

```powershell
$restartId='YOUR_RESTART_SSM_COMMAND_ID'
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
aws ssm get-command-invocation @awsA --command-id $restartId --instance-id $az1 --query "[Status,ExecutionStartDateTime,StandardOutputContent]" --output text
aws ssm get-command-invocation @awsA --command-id $restartId --instance-id $az2 --query "[Status,ExecutionStartDateTime,StandardOutputContent]" --output text
```

![両EC2のKinesis Agent再起動結果をSSM履歴から取得した実画面](/images/squid-agent/restart-history-terminal.png)
*検証時の再起動を実行したSSMコマンドについて、Success・実行日時・activeの出力をAWS CLIで取得した実画面です。撮影時に再起動操作を行ったものではありません。*

各EC2で正常再起動の前後に5アクセスずつ、合計20件を発生させました。チェックポイントを保持した状態で、全20件が欠損・重複なくS3へ到着しました。確認したのはKinesis Agentプロセスの正常再起動です。

### 送信できない間のログを確認する

接続拒否を模擬するため、Kinesis Agentの送信先を一時的に`https://127.0.0.1:9`へ切り替えました。Squidの通信先やB側のネットワーク設定は変えていません。

両EC2で`Connection refused`と`Retriable send error ... Will retry`を確認しました。送信できない間に生成した10件と、元の設定へ戻してKinesis Agentを再起動した後の10件は、全20件がS3へ1回ずつ到着しています。復旧時にもチェックポイントは削除していません。

保存した送信記録とS3原本を照合した結果は次のとおりです。

| 条件 | HTTP 200 | S3へ1回ずつ到着 |
|---|---:|---:|
| ローテーション | 20件 | 20件 |
| 正常再起動 | 20件 | 20件 |
| 一時送信失敗からの復旧 | 20件 | 20件 |

**EC2上で直接確認する場合（bash）**

```bash
sudo grep -E 'Connection refused|Retriable send error' /var/log/aws-kinesis-agent/aws-kinesis-agent.log | tail -n 1
```

送信失敗時のログを確認したSSMコマンドIDを指定します。画像はその実行履歴を取得したものです。

```powershell
$failureId='YOUR_FAILURE_OBSERVATION_SSM_COMMAND_ID'
```

**画像で実行したコマンド（Windows / PowerShell）**

```powershell
aws ssm get-command-invocation @awsA --command-id $failureId --instance-id $az1 --query StandardOutputContent --output text | Select-Object -First 1
aws ssm get-command-invocation @awsA --command-id $failureId --instance-id $az2 --query StandardOutputContent --output text | Select-Object -First 1
```

![両EC2の送信失敗時の再試行ログをSSM履歴から取得した実画面](/images/squid-agent/retry-history-terminal.png)
*送信失敗の観測コマンドのSSM実行履歴をAWS CLIで取得し、各EC2の先頭のエラー行を表示した実画面です。Connection refusedとWill retryを確認できます。障害を再発生させた画像ではありません。*

今回の結果だけで、異常終了や応答喪失を含むすべての条件で重複が発生しないとは説明できません。収集位置の記録、送信の再試行、S3到着データを合わせて確認し、必要なら下流でリクエストIDなどによる重複排除を設計します。

**参考資料**

- [AWS：FirehoseとAWS PrivateLink](https://docs.aws.amazon.com/firehose/latest/dev/vpc.html)
- [AWS：Interface EndpointのPrivate DNS](https://docs.aws.amazon.com/vpc/latest/privatelink/interface-endpoints.html)
- [AWS：PrivateLinkの料金](https://aws.amazon.com/privatelink/pricing/)
- [AWS：IAMのグローバル条件キー（aws:SourceVpce）](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-sourcevpce)

- [AWS：Kinesis Agentの設定](https://docs.aws.amazon.com/firehose/latest/dev/agent-config-settings.html)
- [AWS：Kinesis Agentの導入](https://docs.aws.amazon.com/firehose/latest/dev/download-install.html)
- [AWS：Firehoseの料金](https://aws.amazon.com/firehose/pricing/)
- [Amazon Kinesis Agentの公開集約実装](https://github.com/awslabs/amazon-kinesis-agent/blob/aa0d9089ed6b0ce9c10871f403bfcf20b74a461f/src/com/amazon/kinesis/streaming/agent/tailing/AggregationSplitter.java)
