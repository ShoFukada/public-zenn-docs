# public-zenn-docs

Zenn 向けの公開用ドキュメントリポジトリです。`articles/` と `books/` を管理し、ローカルでは Zenn CLI で作成・プレビュー、Lint/Format は lefthook と GitHub Actions でチェックします。

参考: [Zenn CLI ガイド](https://zenn.dev/zenn/articles/zenn-cli-guide)

## 目次

- 概要
- セットアップ
- 記事/本の作成とプレビュー
- Lint / Format
- Git Hooks (lefthook)
- CI (GitHub Actions)
- Dependabot 設定
- 各種設定まとめ
- 補足/既知の注意点

## 概要

- ディレクトリ構成: `articles/` と `books/`（空ディレクトリ維持のため `.keep` あり）
- Node で動作（CI では Node.js 22 を使用）
- Lint: textlint / markdownlint / secretlint、整形: Prettier
- Git Hooks: pre-commit で Lint/Format、commit-msg で commitlint を実行

## セットアップ

- 必要条件: Node.js 18 以上を推奨（CI は 22）。
- 依存ライブラリは基本 `npx` で都度実行します（ローカルに devDependencies は最小限）。
- Git フックを有効化（初回のみ）:
  - `npx lefthook install`

VS Code 推奨設定は `.vscode/settings.json` に含まれており、Markdown を保存時に Prettier で整形します。

## 記事/本の作成とプレビュー

Zenn CLI をローカルインストールしていない前提で `npx` を使います。

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
  - 既定で [http://localhost:8000](http://localhost:8000) が開きます

より詳しい使い方は Zenn 公式ガイドを参照してください。

## Lint / Format

このリポジトリでは以下を使用します。`npx` で都度実行できます。

- textlint（MD/MDX の日本語校正）:
  - `npx textlint "**/*.{md,mdx}"`
- markdownlint（Markdown スタイル）:
  - `npx markdownlint-cli2 "**/*.{md,mdx}"`
- secretlint（シークレット漏えい検知）:
  - `npx secretlint --maskSecrets --secretlintignore .gitignore "**/*"`
- Prettier（整形・保存時整形も有効）:
  - チェック: `npx prettier --check "**/*.{js,ts,jsx,tsx,md,mdx,json,yml}"`
  - 自動整形: `npx prettier --write "**/*.{js,ts,jsx,tsx,md,mdx,json,yml}"`

## Git Hooks (lefthook)

`lefthook.yml` により、以下が Git フックで実行されます。

- pre-commit（並列実行）:
  - secretlint: `npx secretlint --maskSecrets --secretlintignore .gitignore "{staged_files}"`
  - prettier: `npx prettier --write "{staged_files}"`
  - textlint: `npx textlint --cache {staged_files}`
  - markdownlint: `npx markdownlint-cli2 {staged_files}`
- commit-msg:
  - commitlint: `npx commitlint -e`

初回は `npx lefthook install` を忘れずに実行してください。

## CI (GitHub Actions)

`.github/workflows/lint.yml`（ワークフロー名: `reviewdog`）で PR に対してチェックを行います。

- トリガー: `main` ブランチへの Pull Request（`**/*.md` が変更対象）
- 環境: `ubuntu-latest` / Node.js 22（`actions/setup-node@v4`）
- キャッシュ: `~/.npm` と `.textlintcache`
- 実行内容:
  - textlint: `tsuyoshicho/action-textlint@v3`（`github-pr-review` reporter、warning レベル、`**/*.{md,mdx}`）
  - markdownlint: `npm run markdownlint` を実行（注: 現状 `package.json` に該当スクリプトは未定義）

## Dependabot 設定

`.github/dependabot.yml` により依存更新 PR を自動作成します。

- GitHub Actions: 週次、JST 10:00、最大 10 PR、ラベル `dependencies`、コミットメッセージ接頭辞 `Common`
- npm: 日次、JST 10:00、最大 10 PR、ラベル `dependencies`、コミットメッセージ接頭辞 `Common`

## 各種設定まとめ

- Prettier: `.prettierrc`
  - 主な設定: `printWidth: 90`, `semi: false`, `singleQuote: true`, `trailingComma: none`, `proseWrap: never` など
- textlint: `.textlintrc.yml`
  - 使用: `preset-smarthr`（句点混在許容）、`preset-ja-spacing`（半全角間スペース無効）、`preset-ja-technical-writing`（文長/句点混在/漢字連続/!?, の制限無効）
  - `spellcheck-tech-word: true`
  - `prh` のカスタム辞書: `rules.yml`（例: `サーバ` → `サーバー`）
- markdownlint: `.markdownlint-cli2.jsonc`
  - MD013/MD010/MD026/MD033/MD034/MD037/MD041/MD024/MD051 を無効化、`.git` と `node_modules` を除外
- secretlint: `.secretlintrc.yml`
  - `@secretlint/secretlint-rule-preset-recommend`
- commitlint: `.commitlintrc.yml`
  - 規約: `@commitlint/config-conventional`
  - ルール: `subject-case` は `start-case`/`sentence-case`/`pascal-case` を禁止
- VS Code: `.vscode/settings.json`
  - Markdown 保存/貼り付け時に Prettier で整形、textlint 対象に `mdx/markdown/plaintext/html`
