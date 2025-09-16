---
title: 'uvで始めるモダンPythonプロジェクトセットアップ'
emoji: '🐕'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['Python', 'uv']
published: false
publication_name: medurance
---

## 0.はじめに

初めまして。Meduranceエンジニアの深田翔(ふかしょ)です。本記事ではuvを使ったモダンなPythonプロジェクトセットアップを紹介していきます。

- uvの基本的な使い方とプロジェクト初期化
- モダンなPython開発環境（linter/formatter/型チェック）の構築
- テスト環境の設定とCI(precommit, Github Actions)の導入
- VSCode設定

## 1. uvについて

uvは、Astral社が開発したRust製のPythonパッケージ・プロジェクトマネージャーです。従来のpip + venvやpoetryに代わるツールとして多くのプロジェクトで採用されはじめています。

### 1.1. uvの主なメリット

- **高速な依存関係解決**: Rust製のため、pipやpoetryよりも圧倒的に高速
- **統一されたツール**: Pythonバージョン管理、依存関係管理、仮想環境管理を一元化
- **豊富な機能**: ユニバーサルロックファイル、クロスプラットフォームサポートなど

詳しくはこちらの記事を参考にしてください: [uvの詳細なメリット・デメリット解説](https://zenn.dev/tabayashi/articles/52389e0d6c353a) 個人的には早さだけで使う価値があると思います。大きめのOSSなどをuv使う場合とそうでない場合で実際にinstallしてみると違いが実感できると思います。

## 2. uvのインストール

公式ドキュメント: https://docs.astral.sh/uv/#getting-started

:::message
すでにインストール済みの方はスキップしてください
:::

### macOS / Linux

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Windows

```powershell
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

インストール後、バージョンを確認しましょう：

```bash
uv --version
```

## 3. プロジェクトの初期化

### 3.1. Pythonバージョンの指定とプロジェクト作成

```bash
uv python pin 3.12
uv init --package my-python-project
cd my-python-project
```

Pythonのバージョンについては、[Python support schedule](https://devguide.python.org/versions/)を確認し、securityとなっているバージョンのうち最新の3.12を使うことが僕は多いです。ただ、使うライブラリによっては前のバージョンになることもあるかと思います。自分の開発環境や使用ライブラリなどを考えて設定してください。僕は以前faiss(meta社の近傍探索ライブラリ)を使おうとしたときは3.10までしか対応していなかったとかはありました。

:::message
3.9以下のバージョンについてはAWS Lambdaをはじめ多くの実行環境でサポート終了が近づいているためおすすめしません。
:::

**`--package`オプションについて**:

- `src layout`構造でパッケージとして配布可能なプロジェクトを作成
- 自動的にpyproject.tomlにbuild-systemが設定される

詳細については以下を参照してください：

- [src layout vs flat layout（公式）](https://packaging.python.org/ja/latest/discussions/src-layout-vs-flat-layout/)
- [Pythonプロジェクト構造のベストプラクティス](https://zenn.dev/zenn_tkc/articles/b412ad1b537c82)

### 3.2. 依存関係の同期

```bash
uv sync
```

**このコマンドで行なわれること**:

1. `.python-version`ファイルに基づいてCPython 3.12をインストール
2. `.venv`ディレクトリに仮想環境を作成
3. `uv.lock`ファイル（ロックファイル）を生成
4. プロジェクト自体を開発モードでインストール(詳しくは3.1参考ドキュメントを参照してください)

:::message
3.1にてpackage指定しているため、build-systemがpyproject.tomlに自動追記されておりプロジェクト自体を開発モードでインストールされます。
:::

**実行方法**:

```bash
# 仮想環境を使ってコマンド実行
uv run your_script.py

# または仮想環境をアクティベート
source .venv/bin/activate  # macOS/Linux
# .venv\Scripts\activate     # Windows
```

これだけで仮想環境にてPythonが実行できます。簡単ですね。

## 4. コードフォーマット・リント（Ruff）

次にコード品質担保のため、Formatter,Linterを導入します。Ruffは高速なPython用Linter/Formatterです。Black、isort、flake8などの機能を統合し、Rust製で非常に高速です。

### 4.1. インストール

```bash
uv add --dev ruff
```

:::message
`uvx ruff`でインストール不要で実行することも可能ですが、開発環境をメンバーで共有するため、プロジェクトの開発環境依存関係として追加します。
:::

### 4.2. 基本的な使い方

とりあえず以下2つが使えれば大丈夫です。

```bash
# リント実行（自動修正あり）
uv run ruff check --fix

# フォーマット実行
uv run ruff format
```

ただし、自動fixが困る場合は --fixを外すか、後述の設定unfixableに自動fixしないものを追加しましょう。その他のコマンド詳細については[公式サイト](https://docs.astral.sh/ruff/)を参考にしてください。

### 4.3. 設定ファイル（pyproject.toml）

以下の設定を`pyproject.toml`に追加します：

```toml
[tool.ruff]
line-length = 120

[tool.ruff.lint]
select = ["ALL"]

ignore = [
    # Documentation
    "D1",     # Missing docstrings
    "D203",   # One blank line before class
    "D213",   # Multi-line summary second line
    "D400",   # First line should end with period
    "D415",   # First line should end with punctuation

    # Development
    "TD",     # todo tags allowed
    "T201",   # print() allowed for debugging

    # Exception handling
    "TRY003", # Long exception messages allowed
    "EM101",  # Exception string literals allowed
    "COM812", # Trailing comma missing (conflicts with formatter)
]

unfixable = [
]

[tool.ruff.lint.per-file-ignores]
"tests/*" = [
    "S101",    # assert allowed in tests
    "PLR2004", # Magic values allowed in tests
    "ANN",     # Type annotations optional in tests
    "D",       # Docstrings optional in tests
]
```

設定についてですが、[Ruff Rules](https://docs.astral.sh/ruff/rules/)に記載のルールのうち、Deprecateのもの+いらないものをignoreしてそれ以外はselect allで入れるのが良いと思います。

## 5. 型チェック（Pyright）

大規模なプロジェクトでは、Pythonでも型チェックがないと辛くなってきます。そのため、Microsoft製の静的型チェッカーであるPyrightを使用します。

:::message
Pythonなんて型チェックなしで大丈夫やって人はスキップしてください（笑）。
:::

### 5.1. インストールと設定

```bash
uv add --dev pyright
```

`pyproject.toml`に設定を追加：

```toml
[tool.pyright]
typeCheckingMode = "standard"  # basic/standard/strict から選択
pythonVersion = "3.12"
```

### 5.2. 実行

```bash
uv run pyright
```

**typeCheckingModeの選択**:

- `basic`: 最小限の型チェック
- `standard`: 推奨レベル（バランスの取れた設定）
- `strict`: 最も厳しい型チェック

詳細は[公式ドキュメント](https://microsoft.github.io/pyright/#/)を参照してください。

## 6. テスト環境（Pytest）

PythonのテストフレームワークとしてPytestを使用します。カバレッジ測定のためpytest-covも同時にインストールします。

### 6.1. インストール

```bash
uv add --dev pytest pytest-cov
```

### 6.2. サンプルコード

動作確認のためのサンプルコードとテストを作成しましょう。

#### src/my_python_project/calculator.py

```python
"""シンプルな計算機モジュール"""


def add(a: float, b: float) -> int | float:
    """2つの数値を加算"""
    return a + b


def divide(a: float, b: float) -> float:
    """2つの数値を除算"""
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b


class Calculator:
    """計算機クラス"""

    def __init__(self) -> None:
        self.result: float = 0

    def add(self, value: float) -> float:
        """値を加算"""
        self.result += value
        return self.result

    def reset(self) -> None:
        """結果をリセット"""
        self.result = 0
```

#### tests/test_calculator.py

```python
"""calculator.pyのテスト"""

import pytest

from my_python_project.calculator import Calculator, add, divide


def test_add():
    """add関数のテスト"""
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0.5, 0.5) == 1.0


def test_divide():
    """divide関数のテスト"""
    assert divide(10, 2) == 5.0
    assert divide(7, 2) == 3.5

    # ゼロ除算のテスト
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)


class TestCalculator:
    """Calculatorクラスのテスト"""

    def test_add_method(self):
        """加算メソッドのテスト"""
        calc = Calculator()
        assert calc.add(5) == 5
        assert calc.add(3) == 8
        assert calc.add(-2) == 6

    def test_reset(self):
        """リセットメソッドのテスト"""
        calc = Calculator()
        calc.add(10)
        calc.reset()
        assert calc.result == 0
```

### 6.3. テストの実行

```bash
# 基本的なテスト実行
uv run pytest

# カバレッジ付きでテスト実行
uv run pytest --cov=src --cov-report=term-missing

# HTMLレポート付き
uv run pytest --cov=src --cov-report=html
```

## 7. VSCode設定

効率的な開発のため、VSCodeの設定を行ないます。

### 7.1. 必要な拡張機能

1. Python（ms-python.python）
2. Ruff（charliermarsh.ruff）

### 7.2. 設定ファイル

`.vscode/settings.json`を作成し、以下を記載：

```json
{
  // Pylance/Pyright設定
  "python.analysis.autoImportCompletions": true,
  "python.analysis.diagnosticMode": "workspace",
  "python.analysis.inlayHints.functionReturnTypes": true,
  "python.analysis.inlayHints.variableTypes": true,
  "python.analysis.inlayHints.pytestParameters": true,

  // フォーマッター設定
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  }
}
```

## 8. Pre-commit設定

コミット時に自動的にフォーマット・リント・型チェックを実行し、コード品質を維持するためPrecommitを導入します。

### 8.1. インストールと初期設定

```bash
uv add --dev pre-commit
uv run pre-commit install
```

### 8.2. 設定ファイル

`.pre-commit-config.yaml`を作成し以下の内容を記載してください

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.7
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: local
    hooks:
      - id: pyright
        name: pyright
        entry: uv run pyright
        language: system
        types: [python]
        pass_filenames: false
```

この設定により、コミット時に以下の処理が自動実行されます：

1. **Ruff (astral-sh/ruff-pre-commit)**
   - `ruff`: Pythonコードのリントを実行し、`--fix`オプションで自動修正可能な問題を修正
   - `ruff-format`: Pythonコードを自動フォーマット（Black互換のフォーマッタ）

2. **Pyright (local hook)**
   - Pythonの静的型チェックを実行
   - `language: system`により、プロジェクトのPython環境を使用
   - `pass_filenames: false`により、変更されたファイルではなくプロジェクト全体をチェック
   - `types: [python]`により、Pythonファイルのみを対象に実行

### 8.3. 動作確認

```bash
# 全ファイルに対してpre-commitを実行
uv run pre-commit run --all-files
```

## 9. GitHub Actions CI設定

プルリクエスト・プッシュ時に自動的にテスト・品質チェックを実行するCI設定を行ないます。僕はGithub Actionsを使うことが多いです。

### 9.1. ワークフローファイル

`.github/workflows/ci.yml`を作成：

```yaml
name: CI

permissions:
  contents: read
  pull-requests: write
  issues: write

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  pre-commit:
    name: Pre-commit checks
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install uv
        uses: astral-sh/setup-uv@v3

      - name: Install dependencies
        run: |
          uv sync --dev

      - name: Run pre-commit
        uses: pre-commit/action@v3.0.1
  pytest:
    name: Run tests with pytest
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install uv
        uses: astral-sh/setup-uv@v3

      - name: Install dependencies
        run: |
          uv sync --dev

      - name: Run tests with coverage
        run: |
          uv run pytest --cov=src --cov-report=term-missing --cov-report=xml --cov-report=json
          echo "## Coverage Report" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
          uv run coverage report >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY

      - name: Coverage comment on PR
        if: github.event_name == 'pull_request'
        uses: py-cov-action/python-coverage-comment-action@v3
        with:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          MINIMUM_GREEN: 80
          MINIMUM_ORANGE: 60

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: |
            coverage.xml
            coverage.json
            htmlcov/
```

このGitHub Actions設定により、以下の処理がPR作成時やmainブランチへのプッシュ時に自動実行されます。

- pre-commitジョブ - 8章で設定したpre-commit(lint/format/typecheck)を走らせます。

:::message
pre-commitで設定してるから不要じゃないの？　と思う方もいるかも知れませんが、pre-commitはローカルrepo側の品質管理色が強く、また、無効化してpush等も可能であるため、リモートリポジトリ側での品質管理として、GitHub actionsのciでpre-commitのコマンドを走らせます。(また、actions側もlint/format/typecheckとして定義する方が本当は望ましいですがここでは簡単のためpre-commitを流用する形にしています。)
:::

- pytestジョブ
  - `pytest --cov=src`でsrcディレクトリのカバレッジを測定
  - py-cov-action/Python-coverage-comment-action@v3を使用して、PR作成時にcommentでカバレッジレポート表示
  - actions/upload-artifact@v4を使用してactions上でカバレッジレポート表示。

うまくいっていれば以下のように表示されます。

![GitHub Actionsでのテスト結果とカバレッジレポート](/images/uv-python-starter/actions_report.png)

## 10.その他僕がよく使う設定・tips

### 10.1. 環境変数管理（Pydantic Settings）

型安全な環境変数の管理をするために、loaddotenvではなくpydantic-settingsを使うことが多いです。

```bash
uv add pydantic-settings
```

#### .env

```env
OPENAI_API_KEY=your_openai_api_key_here
DEBUG=true
```

#### src/my_python_project/config.py

```python
import os
from pathlib import Path
from typing import Any

from pydantic_settings import BaseSettings, SettingsConfigDict


def find_env_file() -> str | None:
    """プロジェクトルートの.envファイルを再帰的に探す"""
    current_path: Path = Path(__file__).resolve()

    for parent in current_path.parents:
        env_file: Path = parent / ".env"
        if env_file.exists():
            return str(env_file)

    return None


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=find_env_file(),
        env_file_encoding="utf-8",
    )

    OPENAI_API_KEY: str | None = None
    DEBUG: bool = False

    def __init__(self, **data: Any) -> None:  # noqa: ANN401
        super().__init__(**data)
        for field_name, field_value in self.model_dump().items():
            if field_value is not None:
                os.environ[field_name] = str(field_value)
```

**init**メソッドをオーバライドしてる部分については、例えばOPENAI_API_KEYなどはos.environに登録しておくと、OpenAIモジュールが自動で読み込んで受け渡しが不要になるためそのような処理を追加してます。(他にもAWS_ACCESS_KEY_IDなど) ただし、一時的に実行環境において環境変数が汚れてしまうのが嫌な人はオーバライドをせずに各種インスタンス作成時に明示的に渡すようにしましょう。

### 10.2. 開発効率化のTips

エンジニアごと諸説あるとは思いますが、僕はプロジェクト直下にmemo.mdを作成&gitignoreして、個人的なメモとして使うことが多いです。例えば、シークレット値も含めたコマンドや、ちょっとしたtodo管理、aiに投げるプロンプトのメモ書きに使用しています。

## まとめ

uvを使ったモダンなPython開発環境の構築方法を紹介しました。ぜひお試しください！　今後もPython, TypeScript, AWS関連の記事を中心に投稿していく予定ですので、フォローいただけると嬉しいです！
