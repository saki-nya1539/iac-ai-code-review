# iac-ai-code-review

生成AIが書いたTerraformコード(IaC)を人間がレビューし、セキュリティ・保守性の観点で改善した記録です。「AIが生成した構成案を人間が評価・改善する」という、AIと人間の協調開発プロセスの実践例としてまとめています。

## 内容

- 生成AIが書いたと想定したTerraformコード(レビュー対象)
- 発見した問題点(認証情報の直書き、ネットワークの過剰な開放、リソース参照漏れなど)
- 改善後のコード
- レビューを通じて得た、生成AIコードをチェックする際の観点のまとめ

## 関連リポジトリ

- [aws-lamp-terraform](https://github.com/saki-nya1539/aws-lamp-terraform) : 本レビューの元になった実際のTerraform実装
- [aws-lamp-setup-guide](https://github.com/saki-nya1539/aws-lamp-setup-guide) : AWS環境構築の手順書
