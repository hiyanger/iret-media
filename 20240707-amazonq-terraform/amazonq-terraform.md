title:【もくもく会ブログリレー6日目】Amazon Q DeveloperはTerraformのコードを賢く補完してくれるのか

この記事は「[もくもく会ブログリレー](https://iret.media/tag/%E3%82%82%E3%81%8F%E3%82%82%E3%81%8F%E4%BC%9A%E3%83%96%E3%83%AD%E3%82%B0%E3%83%AA%E3%83%AC%E3%83%BC)」 6日目の記事です。

2024/4/30、Amazon Q DeveloperがGAされました。もともとCodeWhispererでTerraform（hcl）もコード補完はできましたが、改めて精度を確かめてみます。

<a href="https://aws.amazon.com/jp/about-aws/whats-new/2024/04/amazon-q-developer-generally-available/">Amazon Q Developer の一般提供を開始</a>

## 概要
### 精度
精度はまだ不完全ではあるものの、十分に役に立つレベルでした。書きはじめは予測内容に戸惑うかもしれませんが、プロンプトのコツを掴めばある程度は効率化を図れます。

### 価格
単純にコード補完をしたいだけなら無料です。Copilotは有料なので、ここはAmazon Q Developerの強みです。その他セキュリティスキャンなどいくつかの機能で回数が制限されていますが、Pro版にすることで各機能の上限を上げることが可能です。

<a href="https://aws.amazon.com/jp/q/developer/pricing/">Amazon Q Developer の料金</a>

### LLM
Bedrockで利用できる複数のLLMを組み合わせているようです。そのため、なにかのLLMに依存することもなさそうなので、今後もかなり期待できそうです。中にはClaudeがあったりします。

<a href="https://xtech.nikkei.com/atcl/nxt/column/18/02662/112900002/">AWSが生成AIアシスタント「Amazon Q」発表、クラウド熟知し開発を支援</a>

## 導入
VSCodeでの利用方法を記述します。

### 1.AWS Builder IDの取得
<a href="https://profile.aws.amazon.com/">こちら</a>から「AWS Builder ID」を取得します

### 2.VSCodeの拡張機能設定
① 拡張機能から「Amazon Q」をインストールします

<img src="https://iret.media/wp-content/uploads/2024/07/q_terraform_image_1.png" alt="" width="329" height="657" class="alignnone size-full wp-image-107733" />

② 「Use For Free」→「Continue」でサインインします

<img src="https://iret.media/wp-content/uploads/2024/07/q_terraform_image_2.png" alt="" width="550" height="651" class="alignnone size-full wp-image-107734" />

③ 「Allow access」でアクセスを許可します

<img src="https://iret.media/wp-content/uploads/2024/07/q_terraform_image_3-720x433.png" alt="" width="720" height="433" class="alignnone size-large wp-image-107735" />

④ VSCode画面下部に「▷ Amazon Q」が表示されればOKです
   
<img src="https://iret.media/wp-content/uploads/2024/07/q_terraform_image_4-720x547.png" alt="" width="720" height="547" class="alignnone size-large wp-image-107736" />

### 3.コードの書き方
コメント（#）がプロンプトになっています。コードを書き始めると、以下のような感じで予測してくれます。予測の内容が問題なければ「TABキー」を押します。

<img src="https://iret.media/wp-content/uploads/2024/07/q_terraform_image_5.png" alt="" width="265" height="112" class="alignnone size-full wp-image-107737" />

## リソースを作ってみる
プロンプトは最低限を意識して、S3、VPC、サブネット、セキュリティグループ、EC2、その他Terraform特有の記述を書いてみました。基本のリソースは一連の `main.tf` に記述しています。

※あくまで私自身の環境で作り出したコードなので、同じプロンプトでも同じコードにならない可能性高いです。（実際に数回出力してみると出力内容がかわったりします）予測精度の参考程度にしてください。

### S3 / 結果：◯
とりあえずかけました。バケット名は自分で書きましょう。

```
# amazonqというS3バケットを作成
resource "aws_s3_bucket" "amazonq" {
    bucket = "XXXXXXX" # 自分で記載
}
```

### VPC / 結果：×
CIDRは172...で作られました。dsn系の記述でアンダーバーが足りません。タグは記述に失敗しているのと、思わしくない名称が設定されています。

思わしくない記述は後のコードでも変わらないので、以降必要な箇所は手動で直しておきます。

```
# amazonqというVPCを作成
resource "aws_vpc" "amazonq" {
    cidr_block = "172.31.0.0/16"
    enable_dnsSupport = true # 削除
    enable_dnsHostnames = true # 削除
    tags { # =を追加
        Name = "Amazon Q VPC" # amazonqに変更
    }
}
```

### サブネット / 結果：◯
VCPを手動で救った分、問題なく記述できています。

```
# amazonqというサブネットを作成
resource "aws_subnet" "amazonq" {
    vpc_id = aws_vpc.amazonq.id
    cidr_block = "172.31.0.0/24"
    tags = {
        Name = "amazonq"
    }
}
```

### セキュリティグループ / 結果：◯
問題なく記述できました。

```
# amazonqというセキュリティグループを作成
resource "aws_security_group" "amazonq" {
    name = "amazonq"
    vpc_id = aws_vpc.amazonq.id
    tags = {
        Name = "amazonq"
    }
}
```

### EC2 / 結果：×
サブネットやセキュリティグループを指定してくれませんでした。AMI IDも含めて、自身で補完してあげましょう。

```
# amazonqというEC2を作成
resource "aws_instance" "amazonq" {
    instance_type = "t2.micro"
    # サブネットを指定
    # セキュリティグループを指定
    ami = "ami-XXXXX" # ami IDを指定
    tags = {
        Name = "amazonq"
    }
}
```

### Terraform特有の記述（provider、backend、variable、output、moduleなど） / 結果：△
プロンプトの調整や自身での補完は必要なものの、ある程度のレベルまでは記述してくれました。

```
# AWSプロバイダを設定する
provider "aws" {
    region = "" # AZは自分で入力
}

# amazonq0628というバケット名のS3にtfstateを配置する
terraform {
    backend "s3" {
        bucket = "XXXXXXX" # 対象バケットは自分で入力
        key    = "tfstate"
        region = "ap-northeast-1"
    }
}

# 変数nameにamazonqを格納する
variable "name" {
    # 型を記述
    default = "amazonq"
}

# VPCIDを出力する
output "vpc_id" {
    value = aws_vpc.amazonq.id # IDは自分で入力
}

# moduleのcommonを取得する
module "common" {
    source = "./modules/common"
    name = var.name
}
```

## その他考慮点
### セキュリティスキャン
Amazon Q Developerではセキュリティスキャンも可能です。これもまた非常に有用な機能です。

#### 使い方
Amazon Qから「Run Project Scan」を選択します。

<img src="https://iret.media/wp-content/uploads/2024/07/q_terraform_image_6.png" alt="" width="605" height="288" class="alignnone size-full wp-image-107738" />

本記事で記載したコードをスキャンすると、EBS暗号化、EC2のインスタンス詳細監視無効が出力されました。

<img src="https://iret.media/wp-content/uploads/2024/07/q_terraform_image_7.png" alt="" width="1595" height="111" class="alignnone size-full wp-image-107739" />

### GitHub Copilot
Copilotも使ってみましたが、精度と速度ともに非常に高いレベルで記述してくれました。驚いたのは次のプロンプト（VPCが作り終わったらサブネット、セキュリティグループを作ったらEC2をみたいな流れ）さえも高い精度で表示してくれることでした。

なお、Copilotでは高い精度の補完機能を得られるものの、個人利用で$10/月、ビジネスで$19/月の課金が必要です。

### IaCとしてのコード補完の考え方
#### プロンプトの使い方
コードは100%狙ったものを出そうとすると、プロンプトにこだわる必要があります。これを行うと、かえってコーディングの速度を下げてしまいますし、プロンプトも非常に長くなってしまいます。そのため、使いかたとしては簡易なプロンプトでブロックの基盤を作り出し、パラメータの過不足は自分で補うという書き方が現状のベストだと感じました。

#### パラメータの信用性
足りないパラメータは補えばいいですが、悪影響を及ぼすパラメータが書かれてしまうと厄介です。これは外部のコードを参照したときも起こり得ますが、参照先が不明（検証されていない）であるということと、流れでコードが書けてしまうという点から、よりこの悪影響を促進することが考えられます。

これまで通り公式ドキュメントは大事にしつつ、現段階では参考程度の記述だと信用のハードルを下げておくのが良さそうです。

## 明日の記事
明日の記事は、佐藤竜也さんの「Amazon IVSのログデータのアーカイブ公開する方法をAWSサービスで模索してみた」です。iret配信王の異名をもつ佐藤竜也さん。各地の配信イベントで活躍している彼の記事は、配信に興味がある人、必見です！！