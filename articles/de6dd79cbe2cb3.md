---
title: "TerraformをGitHubActionsを使ってCI/CDを試す"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform", "GitHubActions"]
published: false
---

TerraformをGitHubActionsを使って実行します。

# 方針

`main`ブランチに`push`すると、`terraform init`, `terraform fmt -check`, `terraform plan`, `terraform apply`が実行されるようにします。

また今回作成するAWSのリソースはVPCとセキュリティグループとします。簡単に変更などがわかる かつ 料金がかからないためです。

（ゆくゆくは`main`にプルリクエストが送られたら、`terraform plan`まで行い、マージするときに`terraform apply`がしたい… もう少しGitHubActionsを勉強します）

# リポジトリにAWSの認証情報を登録

settings > secrets にAWSの`AWS_ACCESS_KEY_ID`と`AWS_SECRET_ACCESS_KEY`を登録します。
![](https://storage.googleapis.com/zenn-user-upload/9d140ed209c677621ebcebbc.png)

# TerraformのBackend S3を作成

`tfstate`を保存するためのS3バケットを作成します。
これがないとTerraformは作成したリソースがわからなくなるため、applyのたびにリソースが作成されるので、冪等性がなくなります。（経験談）

特に設定せず、デフォルトで作成しましたが、実行できました。

公式ドキュメントには設定するべきバケットポリシーが書いてあります

https://www.terraform.io/docs/language/settings/backends/s3.html

# Terraform作成

今回は前述の通りVPC（と、それに付帯するもの）と、セキュリティグループを作成します。

* main.tf
* network.tf
* sercurity_group.tf

それと変数の`terraform.tfvars`も作成してみます。

## main.tf

:::details main.tf
```hcl:main.tf
terraform {
  required_version = ">= 0.12"
  backend "s3" {
    bucket = "YOUR_BUCKET_NAME"
    key    = "terraform.tfstate"
    region = "ap-northeast-1"
  }
}

provider "aws" {
  region = "ap-northeast-1"
}
```
:::



## network.tf

:::details network.tf
```hcl:network.tf
########################
### VPC
# https://www.terraform.io/docs/providers/aws/r/vpc.html
########################
resource "aws_vpc" "main" {

  cidr_block = "172.30.0.0/16"

  tags = {
    Name = "main"
  }
}

########################
### Subnet
# https://www.terraform.io/docs/providers/aws/r/subnet.html
########################

# public
resource "aws_subnet" "public_1a" {

  vpc_id = aws_vpc.main.id

  availability_zone = "ap-northeast-1a"

  cidr_block = "172.30.1.0/24"

  tags = {
    Name = "main-public-1a"
  }

}

resource "aws_subnet" "public_1c" {

  vpc_id = aws_vpc.main.id

  availability_zone = "ap-northeast-1c"

  cidr_block = "172.30.2.0/24"

  tags = {
    Name = "main-public-1c"
  }

}

resource "aws_subnet" "public_1d" {

  vpc_id = aws_vpc.main.id

  availability_zone = "ap-northeast-1d"

  cidr_block = "172.30.3.0/24"

  tags = {
    Name = "main-public-1d"
  }

}


#########################
### Internet Gateway
# https://www.terraform.io/docs/providers/aws/r/internet_gateway.html
#########################
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

########################
### Route Table
# https://www.terraform.io/docs/providers/aws/r/route_table.html
########################
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-RouteTable-public"
  }
}

########################
### Route
# https://www.terraform.io/docs/providers/aws/r/route.html
########################
resource "aws_route" "public" {
  destination_cidr_block = "0.0.0.0/0"
  route_table_id         = aws_route_table.public.id
  gateway_id             = aws_internet_gateway.main.id
}

########################
### Association
# https://www.terraform.io/docs/providers/aws/r/route_table_association.html
########################
resource "aws_route_table_association" "public_1a" {
  subnet_id      = aws_subnet.public_1a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_1c" {
  subnet_id      = aws_subnet.public_1c.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_1d" {
  subnet_id      = aws_subnet.public_1d.id
  route_table_id = aws_route_table.public.id
}
```
:::

## security_group.tf

`myHouseIp`変数で家のIPを変数にいれて利用します

:::details security_group.tf
```hcl:security_group.tf
variable "myHouseIp" {}

resource "aws_security_group" "allow_tls" {
  name        = "allow_tls"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "SSH from My House"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.myHouseIp]
  }

  ingress {
    description      = "HTTP from ALL"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  ingress {
    description      = "TLS from ALL"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "allow_tls"
  }
}
```
:::


## terraform.tfvars

:::details terraform.tfvars
```hcl:terraform.tfvars
# 環境変数ファイル
myHouseIp = "XX.XX.XX.XX/32"
```
:::


# GitHubActionsの作成

リポジトリのActionsタブを開きます

![](https://storage.googleapis.com/zenn-user-upload/8a08bea64554b87138c63094.png)

TerraformのActionsがあるので、こちらを選択します。

![](https://storage.googleapis.com/zenn-user-upload/922f568bc05e40bf1c86443e.png)

こちらはTerraform Cloudというのを使うみたいなので、変更していきます。

:::details .github/workflows/terraform.yml
```yml:.github/workflows/terraform.yml
name: 'Terraform'

on:
  push:
    branches:
    - main
  pull_request:

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest
    environment: production

    defaults:
      run:
        shell: bash

    steps:

    - name: Checkout
      uses: actions/checkout@v2
    - name: configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1

    - name: Terraform Init
      run: terraform init

    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Plan
      run: terraform plan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve
```
:::


# mainにpush!!

これらをmainにpushすると、GitHubActionsが実行されAWSのリソースが作成されます。
適当にセキュリティグループなどを変更してmainにpushすると、リソースが変更されます。新たに作成されずに内容が変更されていることを確認してください。

# destroy…?

そういえば作ったリソースを削除する`terraform destroy`はどうやるの…?
GitHubActionsには`workflow_dispatch`というのがあり、これを利用することで手動で実行する事ができます。実運用ではうっかり押してしまうことがないように作成しないというのもいいかもしれませんが、練習段階では作成したいですよね。

作っておきました

:::details .github/workflows/terraform-destroy.yml
```yml:.github/workflows/terraform.yml/terraform-destroy.yml
name: terraform-destroy
on:
  workflow_dispatch:

jobs:
  terraform:
    name: "Terraform"
    runs-on: ubuntu-latest
    environment: production

    defaults:
      run:
        shell: bash

    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Destroy
        id: destroy
        run: terraform destroy -auto-approve
```
:::

# まとめ

まだまだ改良点があります。例えば、`*.tf`ファイルに変更がないのに、applyまでしてしまいます。GitHubActionsの利用時間は有限なので、なるべく無駄なActionはしないようにしたいです。
すべて実行完了したときはかなり嬉しいですね。楽しくなります。

# 参考

https://qiita.com/keitakn/items/db2e9c68019594885ac4
https://dev.classmethod.jp/articles/try-github-actions-setup-terraform/