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

## スクリーンショット
<img width=50% alt="db244efa3a58543c0f5bdad8b81633ce" src="https://github.com/user-attachments/assets/d4204f10-07b0-495c-847a-f2571c4c18f5" />
<img width=50% alt="bfbdf8713d0ed06f1f15a0b6de6d5ee8_partial_blur" src="https://github.com/user-attachments/assets/a31a8126-8def-418b-a925-b792c50c9394" />

