# 生成AIが書いたIaCコードのレビュー・改善記録

## 目的

生成AIにインフラのコード(IaC)を書かせ、その内容を人間がレビューし、実務レベルまで改善するプロセスを練習する。「AIが生成した構成案を人間が評価・改善する」という、AIと人間の協調開発の一連の流れを体験することが目的。

対象は、AWS上にApache・PHP・MySQLの環境を構築するTerraformコード。「生成AIがそれらしく書いたコード」を題材とし、レビュー・修正を行った。

---

## 1. レビュー対象のコード(生成AI想定の初期案)

```hcl
provider "aws" {
  region = "ap-northeast-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"

  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              apt update -y
              apt install -y apache2 php libapache2-mod-php php-mysql mysql-server
              mysql -e "CREATE USER 'admin'@'localhost' IDENTIFIED BY 'password123';"
              mysql -e "GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost';"
              systemctl start apache2
              EOF
}

resource "aws_security_group" "web_sg" {
  name = "web-sg"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

一見動きそうに見えるが、実務の観点でレビューすると複数の問題を含んでいる。

---

## 2. レビューで指摘した項目

| No. | 指摘事項 | 重大度 | 分類 |
|---|---|---|---|
| 1 | MySQLのパスワード(`password123`)がコードに直書きされている | 高 | AI固有の問題 |
| 2 | SSH(22番ポート)が`0.0.0.0/0`(全世界)に開放されている | 高 | AI固有の問題 |
| 3 | `key_name`(SSHキーペア)の指定が無く、実質SSH接続できない | 高 | AI固有の問題(機能不全) |
| 4 | HTTPS(443番)の設定が無い | 中 | 発展的な改善点(自分のコードにも共通) |
| 5 | AMI IDがハードコードされており、将来的に無効化するリスクがある | 中 | 発展的な改善点(自分のコードにも共通) |

指摘4・5については、レビュー当初は「AIの生成コードだけの欠陥」だと捉えていたが、見直したところ自分が事前に作成していた`main.tf`にも同様の書き方をしていた箇所であり、「AI特有の問題」ではなく「初学者が見落としやすい、実務における一般的な改善ポイント」として位置づけを修正した。この気づき自体が、レビューの精度を高める重要なプロセスだった。

---

## 3. 改善したコード

```hcl
provider "aws" {
  region = "ap-northeast-1"
}

# ハードコードを避け、Ubuntuの最新AMIを動的に検索する
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical公式

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*"]
  }
}

# パスワードはコードに書かず、外部から渡す変数にする
variable "db_password" {
  description = "MySQL admin password"
  type        = string
  sensitive   = true
}

resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow SSH from my IP and HTTP from anywhere"

  ingress {
    description = "SSH from my IP only"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["自分のIPアドレス/32"] # 全世界(0.0.0.0/0)から自分のIPのみに変更
  }

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id # ハードコードをやめて動的取得に変更
  instance_type          = "t3.micro"
  key_name               = "my-web-server-key" # 追加:これがないとSSHできない
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              apt update -y
              apt install -y apache2 php libapache2-mod-php php-mysql mysql-server
              mysql -e "CREATE USER 'admin'@'localhost' IDENTIFIED BY '${var.db_password}';"
              mysql -e "GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost';"
              systemctl start apache2
              EOF

  tags = {
    Name = "web-server-reviewed"
  }
}
```

適用時は、パスワードをコマンドラインから渡す。

```
terraform apply -var="db_password=安全なパスワード"
```

---

## 4. 学び・まとめ

生成AIが出力するIaCコードは、「一見動きそうで実際に動く」レベルには到達しやすい一方、以下の観点は人間が意識的にレビューしないと見過ごされやすいことがわかった。

1. **機密情報の扱い**:パスワードやAPIキーなどをコードに直書きしていないか。`variable`と`sensitive = true`を用い、値はコード外(環境変数・コマンドライン引数・シークレット管理サービス)から渡す。
2. **ネットワークの開放範囲**:管理用ポート(SSH等)が全世界に開放されていないか。用途に応じて必要最小限のソースに絞る。
3. **リソース間の依存関係の抜け**:キーペアの指定漏れのように、動作はしても「使えない」状態になっていないか。
4. **将来的なメンテナンス性**:ハードコードされた値(AMI IDなど)が時間経過で陳腐化しないか。`data`ブロックなどによる動的取得を検討する。
5. **状態ファイル(state)の扱い**:Terraformの状態ファイルには機密情報が平文で残ることがあるため、Gitには含めず、暗号化されたリモートバックエンドで管理する。

これらは、今後生成AIを開発プロセスに組み込んでいく上で、常に持っておくべきレビューの観点だと考える。

---

## 改訂履歴

| 日付 | 内容 |
|---|---|
| 2026-07-18 | 初版作成 |
