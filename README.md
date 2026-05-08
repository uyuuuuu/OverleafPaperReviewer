# OverleafPaperReviewer
## 概要
オーバーリーフで作成した論文に対してclaudecodeが査読(レビュー)を行うためのGithubActions(by self-hosted runner)です。  
本リポジトリのworkflowは論文リポジトリで実行するActionから呼び出されるものであり、本リポジトリのみで動作するものではありません。

## 使用方法
- self-hosted runnerを起動しておく
- シークレットは論文リポジトリの Settings > Secrets and variables > Actions に設定：
  - OVERLEAF_TOKEN       : Overleaf の Git アクセストークン
  - OVERLEAF_PROJECT_ID  : Overleaf プロジェクト ID
- `/examples/workflow.yml` を論文リポジトリの `.github/workflows/review.yml` としてコピー
- 論文リポジトリのActionsタブから実行
