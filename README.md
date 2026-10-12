# OverleafReviewer

## 使用方法
### 初回準備
論文リポジトリを新しく使い始めるときに、1回だけ行います。
1. Overleaf の GitHub 連携で、論文リポジトリを `tdufi-idl` に作ります。
2. `/examples/workflow.yml` を、論文リポジトリの `.github/workflows/review.yml` としてコピーします。
3. 論文リポジトリの **Settings > Actions > General > Workflow permissions** で「Read and write permissions」を選び、「Allow GitHub Actions to create and approve pull requests」にチェックを入れます。
4. `base` ブランチを作ります。テンプレートを書式の基準として使うので、Overleaf からテンプレートを取り込んだ直後のコミット（本文を書き始める前のもの）から作ります。
   ```sh
   git push origin <そのコミット>:refs/heads/base
   ```
5. organization secret の `REVIEWER_READ_TOKEN` が、この論文リポジトリから使えることを確認します（organization の owner が設定します）。
6. GitHub ホストランナーで動かす場合は、下の「GitHub ホストランナー（ubuntu-latest）で動かすための準備」に従って `CLAUDE_CODE_OAUTH_TOKEN` を登録します。

### 実行環境
Actions 画面から実行するときに、`runner` で実行環境を選べます。

| runner | 説明 |
|--------|------|
| `self-hosted`（デフォルト） | セルフホストランナーで実行します。ランナーにログイン済みの Claude Code をそのまま使います。 |
| `ubuntu-latest` | GitHub ホストランナーで実行します。実行のたびに Claude Code をインストールし、下記のトークンで認証します。 |

### GitHub ホストランナー（ubuntu-latest）で動かすための準備
1. 手元の PC で Claude Code にログインした状態で、次のコマンドを実行してトークンを発行します。
   ```sh
   claude setup-token
   ```
2. 論文リポジトリの **Settings > Secrets and variables > Actions** を開き、**New repository secret** から次の名前でトークンを登録します。
   - Name: `CLAUDE_CODE_OAUTH_TOKEN`
   - Secret: 1 で表示されたトークン

`self-hosted` で動かす場合、この secret は不要です。secret が未設定ならランナーのログイン情報が使われます。

## 仕組み
1. 論文リポジトリの `main` を全履歴付きで checkout し、このリポジトリを `_template/` に checkout します。
2. `main` が `base` より進んでいれば、`main` → `base` の PR を作るか更新します。
3. 差分の比較元を決めます。`last-review` タグがあればそれを、なければ `origin/base` を使います。`diff_only` で比較元から変更がなければ、レビューをスキップします。
4. `base` ブランチの中身（提出先のテンプレート）を `_base/` に展開します。
5. `_template/prompts/` の指示文をつなぎ、末尾に比較元と対象（HEAD）を書き足して、`claude -p` に標準入力で渡します。論文の本文は指示文に入れません。Claude Code が論文リポジトリの `.tex` と `.bib` を自分で読みます。
   - 使えるツールは、ファイルの読み取り（Read / Grep / Glob）と、読み取り専用の Bash コマンド（`git diff` / `log` / `show` / `ls-files` / `ls-tree` / `grep` / `blame`、`ls`、`cat`、`head`、`tail`、`wc`、`grep`）だけです。ファイルの編集やほかのコマンド、ファイルへのリダイレクトは実行できません。
   - 使えるツールを読み取りに絞っているので、論文に悪意のある指示が書かれていても、影響はレビュー結果の内容までにとどまります。指示文でも、論文の中の指示には従わないように伝えています。
   - 提出先の学会・論文誌を特定し、その書式に従っているかも確認します。書式は `_base/` のテンプレートを正とします。
6. 結果は `--json-schema` で決めた形式で受け取ります。
   - `summary`：`venue`（提出先）、`format`（発表形式と分量）、`overall`（全体の評価）、`main_issues`（主な問題点、5件まで）
   - `comments`：`path` / `line` / `start_line` / `body`。`proofread` と `final` のときは `suggestion`（修正案）と `basis`（その根拠）も
   - `notes_for_maintainer`：運用者向けのメモ（読めなかったファイルなど）。PR には投稿せず、Actions のログと Summary にだけ出します
   - `approved`
7. `summary` を見出し付きの Markdown にして PR のレビューとして投稿し、`comments` は各ファイルの該当行に行コメントします。
   - 行範囲が逆向き（`start_line` > `line`）なら入れ替えてから投稿します。
   - `basis` があれば「根拠：」として本文に付けます。`basis` のない修正案は外して投稿します。
   - 修正案があるコメントには suggestion 形式のブロックを付けます。
   - PR の差分に含まれない行への指摘は、`path:L行番号` の形で会話タブにまとめて投稿します（修正案は suggestion ではない普通のコードブロックとして載せます）。
8. `last-review` タグを HEAD に移します。次回の `diff_only` はここからの差分になります。
9. `final` かつ `full_review` で `approved` が true なら、承認の通知を投稿します。

### 指示文（prompts/）
| ファイル | 内容 | 使われる段階 |
|----------|------|--------------|
| `common.txt` | 読み手、進め方、提出先の書式の扱い、分量、指摘の書き方、summary と運用者向けメモの書き方、指摘しないこと | すべて |
| `structure.txt` | 構成・論理（\ref と \label の対応を含む） | structure, final |
| `proofread.txt` | 表現・表記、提出先の書式、修正案（suggestion）と根拠（basis）の出し方 | proofread, final |
| `references.txt` | \cite と .bib の対応、書誌情報の欠けと表記の揺れ | structure, proofread, final |
| `final.txt` | 総合判断と approved の基準 | final |
| `diff_only.txt` / `full_review.txt` | レビュー範囲（差分 / 全体）の読み方 | review_mode に応じて |

## 修正案の取り込み方
誤字や表記揺れなど、直し方に迷いがない指摘には修正案（suggestion）が付きます。
1. PR の行コメントで「Commit suggestion」を押します。複数をまとめて反映するときは、それぞれ「Add suggestion to batch」を押してから、まとめてコミットします。
2. Overleaf のメニューから GitHub を開きます。「Recent commits in GitHub」に反映したコミットが表示されるので、「Pull GitHub changes into Overleaf」で取り込みます。表示されないときは右上の更新ボタンを押してください。

> **注意**：取り込む前に Overleaf での編集を止めてください。GitHub 側の変更と Overleaf 側の編集が同時にあると競合します。
