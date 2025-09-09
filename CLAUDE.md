# Agent Guide

## 1. 編集後は Lint/Format を実行（重要）

変更を保存したら、以下の順で Lint/Format を実行してください。エラー/警告が出た場合は修正してから再実行します。

```bash
# 2-1. 整形（差分だけ整形したい場合はパスを絞ってください）
npx prettier --write "**/*.{js,ts,jsx,tsx,md,mdx,json,yml}"

# 2-2. 文章校正（日本語/MD/MDX）
npx textlint --cache "**/*.{md,mdx}"

# 2-3. Markdown スタイル
npx markdownlint-cli2 "**/*.{md,mdx}"

# 2-4. シークレット検知（誤コミット防止）
npx secretlint --maskSecrets --secretlintignore .gitignore "**/*"
```
