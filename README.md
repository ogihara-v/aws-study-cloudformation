# AWS 3層構成 CloudFormation テンプレート（監視・WAF対応）

AWS上に **WAF・ALB を含む3層構成（Web / App / DB）** を、CloudFormation（IaC）で構築するテンプレートです。ネットワーク・サーバー・データベース・ロードバランサーに加え、**CloudWatch による監視通知**と **WAF による攻撃防御・ログ収集**までを、1つのテンプレートでコード化しています。

---

## 構成図（通信の流れ）

```
インターネット
    │  HTTP(80)
    ▼
  WAF（WebACL / AWSマネージドルール）  … ALBに関連付け
    │
    ▼
  ALB（Application Load Balancer）   … パブリックサブネット(1a/1c)
    │  HTTP(8080)
    ▼
  EC2（Amazon Linux 2023 / Webサーバー）  … パブリックサブネット(1a)
    │  MySQL(3306)
    ▼
  RDS（MySQL）  … プライベートサブネット(1a/1c)
```

監視・ログの流れ

```
EC2 ──CPU使用率──▶ CloudWatch Alarm ──▶ SNS Topic ──▶ メール通知
WAF ──検査ログ────▶ CloudWatch Logs（aws-waf-logs-study / 保持7日）
```

---

## 作成されるリソース

| 分類 | リソース | 説明 |
|---|---|---|
| ネットワーク | VPC | 10.0.0.0/16 |
| | パブリックサブネット ×2 | AZ 1a / 1c（ALB・EC2用） |
| | プライベートサブネット ×2 | AZ 1a / 1c（RDS用） |
| | Internet Gateway | 外部との出入口 |
| | ルートテーブル / ルート / 関連付け | 0.0.0.0/0 → IGW |
| セキュリティ | ALB用SG | インターネットから 80 を許可 |
| | EC2用SG | ALBから 8080、SSH 22（管理者IPのみ）を許可 |
| | RDS用SG | EC2から 3306 を許可 |
| アプリ | EC2 | t2.micro / Amazon Linux 2023（最新AMIをSSMから取得） |
| ロードバランサー | ALB / Target Group / Listener | HTTP、ヘルスチェック付き |
| データベース | RDS（MySQL） | db.t4g.micro / プライベート配置 |
| 監視・通知 | SNS Topic | アラーム通知の宛先（メール購読） |
| | CloudWatch Alarm | EC2のCPU使用率を監視し、閾値超過でSNSへ通知 |
| セキュリティ(WAF) | WAF WebACL | AWSマネージドルール（Core rule set）を適用 |
| | CloudWatch Logs ロググループ | WAFの検査ログを保存（保持7日） |
| | WAF Logging Configuration | WebACL と ロググループを紐付け |
| | WebACL Association | WAFをALBに関連付け |

---

## セキュリティ設計

### ネットワーク層（最小権限のSG連鎖）

各層のセキュリティグループを分離し、「1つ前のリソースからのみ許可」する連鎖構成にしています。

- **ALB用SG**：インターネット（0.0.0.0/0）から HTTP(80) を受ける
- **EC2用SG**：ALBのSGからの 8080 のみ許可（+ 管理用SSH 22 は管理者IP/32に限定）
- **RDS用SG**：EC2のSGからの 3306 のみ許可（外部からは直接アクセス不可）

DBはプライベートサブネットに配置し、インターネットから隔離しています。

### アプリケーション層（WAF）

ALBの前段にWAFを配置し、AWSマネージドルール **Core rule set（AWSManagedRulesCommonRuleSet）** を適用しています。SQLインジェクション・クロスサイトスクリプティングなど、OWASPで挙げられる一般的な攻撃パターンを検知・ブロックします。

- デフォルトアクションは `Allow`（ルールに合致しない通常の通信は許可）
- ルールに合致した通信のみブロック
- 検査結果はCloudWatch Logsに出力し、後から検証可能

---

## 前提条件

- AWSアカウント
- 対象リージョン：**東京（ap-northeast-1）**
- 対象リージョンに、EC2用のキーペアを事前に作成しておくこと

---

## デプロイ手順

1. AWSマネジメントコンソールで、リージョンを **東京（ap-northeast-1）** に設定
2. CloudFormation → 「スタックの作成」
3. テンプレートファイル `aws-study.yaml` をアップロード
4. パラメータを入力
   - `KeyPairName`：EC2に設定するキーペアを選択
   - `MyIP`：SSH接続を許可する管理者のグローバルIP（`203.0.113.5/32` 形式）
   - `DBUser`：DBのマスターユーザー名（デフォルト: admin）
   - `DBPassword`：DBのマスターパスワード（8文字以上）
   - `AlarmTopicAddress`：アラーム通知を受け取るメールアドレス
5. スタックを作成（RDS作成のため完了まで10〜20分程度かかります）
6. 作成完了後、届いたSNSの確認メールから **Confirm subscription** をクリック（未承認だと通知が届きません）
7. 「出力（Outputs）」タブを確認
   - `ELBDNSName`：Webアクセス用のALBのDNS名
   - `RDSInstanceEndpoint`：DB接続用のRDSエンドポイント

---

## 動作確認

### アプリケーション

1. EC2にSSH接続し、Webアプリ（8080で待ち受け）を起動
2. ブラウザで `http://（ELBDNSNameのDNS名）` にアクセス
3. WAF → ALB → EC2 → RDS の経路で、アプリの画面が表示されれば成功

> 注：EC2の8080はALB経由のみ許可のため、EC2のIPへ直接アクセスはできません。必ずALBのDNS名でアクセスします。

### 監視アラーム

EC2に負荷をかけ、アラームが発報して通知が届くことを確認します。

```bash
# 負荷をかける（1vCPUを占有）
yes > /dev/null &

# CPU使用率を確認
top -o %CPU

# 負荷を停止
killall yes
```

CloudWatchのアラーム状態が `OK` から `アラーム状態` に変わり、登録したメールアドレスに通知が届きます。
評価期間が5分平均のため、発報まで5〜10分程度の余裕を見て負荷をかけ続けます。

### WAFログ

CloudWatch Logs のロググループ `aws-waf-logs-study` に、WAFの検査ログが出力されます。

---

## 工夫した点

- **機密情報の秘匿**：DBパスワードは `Parameters` の `NoEcho: true` で、テンプレートに直書きせずデプロイ時に入力する設計
- **SSHアクセスの限定**：管理用SSHは `0.0.0.0/0` を避け、`MyIP` パラメータで管理者IP（/32）のみに限定
- **最新AMIの動的取得**：EC2のAMIは、SSMパラメータ（`{{resolve:ssm:...}}`）で常に最新のAmazon Linux 2023を取得。AMI IDの直書きを回避
- **RDSバージョンの委譲**：`EngineVersion` を明示せず、AWSのデフォルトバージョンに委ねることで、バージョン廃止によるデプロイ失敗を回避
- **多層防御**：ネットワーク層（SGの連鎖）に加え、アプリケーション層（WAF）でも防御を行う構成
- **リソース参照の使い分け**：`Ref` の戻り値がリソースごとに異なる（SNS/ALBはARN、WAF WebACLはID）ため、ARNが必要な箇所では `!GetAtt xxx.Arn` を使用。ロググループのARNは末尾の `:*` を避けるため `!Sub` で組み立て
- **セクション分割**：ネットワーク / セキュリティ / アプリ / ロードバランサー / データベース / 監視 / WAF の層ごとにコメントで区切り、可読性を確保

---

## 使用技術

AWS CloudFormation / VPC / EC2 / ALB(ELBv2) / RDS(MySQL) / Security Group / IAM /
SSM Parameter Store / CloudWatch / CloudWatch Logs / SNS / AWS WAF (WAFv2)

---

## 学習の位置づけ

本テンプレートは、AWS設計・構築の学習（ハンズオン）の成果物です。IaCによるインフラ構築の理解を目的に、公式リファレンスを参照しながら各リソースを手書きで作成しました。
監視・WAFについては、先にマネジメントコンソールで設定して各項目の意味を確認したうえで、コードに落とし込んでいます。

---

## 更新履歴

- 監視（CloudWatch Alarm / SNS）と WAF（WebACL / ログ出力 / ALB関連付け）を追加
- 講師フィードバックを反映（SSHのIP限定、RDS EngineVersionの指定解除）
- RDSのDeletionPolicyを明示していなかったため、スタック削除時に自動スナップショットが作成され続け、想定外の保管料が発生。
  DeletionPolicy: Delete を明示して再発防止