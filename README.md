# public-zenn-docs

Zenn向けの公開用ドキュメントリポジトリです。`articles/` と `books/` を管理し、ローカルではZenn CLIで作成・プレビュー、Lint/FormatはlefthookとGitHub Actionsでチェックします。

参考: [Zenn CLI ガイド](https://zenn.dev/zenn/articles/zenn-cli-guide)

## 目次

- 概要
- セットアップ
- 記事/本の作成とプレビュー
- Lint / Format
- Git Hooks (lefthook)
- CI (GitHub Actions)
- Dependabot設定
- 各種設定まとめ
- 補足/既知の注意点

## 概要

- ディレクトリ構成: `articles/` と `books/`（空ディレクトリ維持のため `.keep` あり）
- Nodeで動作（CIでNode.js 22を使用）
- Lint: textlint / markdownlint / secretlint、整形: Prettier
- Git Hooks: pre-commitでLint/Format、commit-msgでcommitlintを実行

## セットアップ

- 必要条件: Node.js 18以上を推奨（CIは22）。
- 依存ライブラリは基本 `npx` で都度実行します（ローカルにdevDependenciesは最小限）。
- Gitフックを有効化（初回のみ）:
  - `npx lefthook install`

VS Code推奨設定は `.vscode/settings.json` に含まれており、Markdownを保存時にPrettierで整形します。

## 記事/本の作成とプレビュー

Zenn CLIをローカルインストールしていない前提で `npx` を使います。

- 記事作成（対話式）:
  - `npx zenn new:article`
- 記事作成（オプション指定例）:
  - `npx zenn new:article --slug my-first-article --title "初めての投稿" --type tech --emoji "📝" --published false`
- 本の作成:
  - `npx zenn new:book`
- 本のチャプター追加（例）:
  - `npx zenn new:chapter --book my-first-book`
- ローカルプレビュー:
  - `npx zenn preview`
  - デフォルトで [http://localhost:8000](http://localhost:8000) が開く

より詳しい使い方はZenn公式ガイドを参照してください。

## Lint / Format

このリポジトリでは以下を使用します。`npx` で都度実行できます。

- textlint（MD/MDXの日本語校正）:
  - `npx textlint "**/*.{md,mdx}"`
- markdownlint（Markdownスタイル）:
  - `npx markdownlint-cli2 "**/*.{md,mdx}"`
- secretlint（シークレット漏えい検知）:
  - `npx secretlint --maskSecrets --secretlintignore .gitignore "**/*"`
- Prettier（整形・保存時整形も有効）:
  - チェック: `npx prettier --check "**/*.{js,ts,jsx,tsx,md,mdx,json,yml}"`
  - 自動整形: `npx prettier --write "**/*.{js,ts,jsx,tsx,md,mdx,json,yml}"`

## Git Hooks (lefthook)

`lefthook.yml` により、以下がGitフックで実行されます。

- pre-commit（並列実行）:
  - secretlint: `npx secretlint --maskSecrets --secretlintignore .gitignore "{staged_files}"`
  - prettier: `npx prettier --write "{staged_files}"`
  - textlint: `npx textlint --cache {staged_files}`
  - markdownlint: `npx markdownlint-cli2 {staged_files}`
- commit-msg:
  - commitlint: `npx commitlint -e`

初回は `npx lefthook install` を忘れずに実行してください。

## CI (GitHub Actions)

`.github/workflows/lint.yml`（ワークフロー名: `reviewdog`）でPRに対してチェックを行ないます。

- トリガー: `main` ブランチへのPull Request（`**/*.md` が変更対象）
- 環境: `ubuntu-latest` / Node.js 22（`actions/setup-node@v4`）
- キャッシュ: `~/.npm` と `.textlintcache`
- 実行内容:
  - textlint: `tsuyoshicho/action-textlint@v3`（`github-pr-review` reporter、warningレベル、`**/*.{md,mdx}`）
  - markdownlint: `npm run markdownlint` を実行（注: 現状 `package.json` に該当スクリプトは未定義）

## Dependabot 設定

`.github/dependabot.yml` により依存更新PRを自動作成します。

- GitHub Actions: 週次、JST 10:00、最大10 PR、ラベル `dependencies`、コミットメッセージ接頭辞 `Common`
- npm: 日次、JST 10:00、最大10 PR、ラベル `dependencies`、コミットメッセージ接頭辞 `Common`

## 各種設定まとめ

- Prettier: `.prettierrc`
  - 主な設定: `printWidth: 90`, `semi: false`, `singleQuote: true` など
- textlint: `.textlintrc.yml`
  - 使用: `preset-smarthr`（句点混在許容）、`preset-ja-spacing`（半全角間スペース無効）、`preset-ja-technical-writing`（文長/句点混在/漢字連続/感嘆符疑問符の制限無効）
  - `spellcheck-tech-word: true`
  - `prh` のカスタム辞書: `rules.yml`（例: `サーバ` → `サーバー`）
- markdownlint: `.markdownlint-cli2.jsonc`
  - MD013/MD010/MD026/MD033/MD034/MD037/MD041/MD024/MD051を無効化、`.git` と `node_modules` を除外
- secretlint: `.secretlintrc.yml`
  - `@secretlint/secretlint-rule-preset-recommend`
- commitlint: `.commitlintrc.yml`
  - 規約: `@commitlint/config-conventional`
  - ルール: `subject-case` は `start-case`/`sentence-case`/`pascal-case` を禁止
- VS Code: `.vscode/settings.json`
  - Markdown保存/貼り付け時にPrettierで整形、textlint対象に `mdx/markdown/plaintext/html`
