---
title: 'S3 Vectors × Knowledge Bases でRAGチャットボット (boto3使用)'
emoji: '🖐️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'bedrock', 'llm', 'python', 'rag']
published: true
publication_name: medurance
---

## 0. はじめに

Meduranceエンジニアの深田翔です。本記事は、AWSからPreview発表されたAmazon S3 VectorsとKnowledge Baseを連携してRAG構築をしてみた記事になります。
なお、各種リソースの作成と削除は、それぞれのリソース・概念の理解を深めるため、コンソールからではなく、Python SDK(boto3)を使って作成しています。
また、metadata検索についてもサポートされる型すべてで実験しています。
:::message
Preview版なこともあり、terraform,cdkで正式にサポートされていないです、早くterraformかcdkでIaCしたいです。
:::

なお、今回の記事の内容を入れたリポジトリについても公開してるので、あわせてご確認ください。
https://github.com/ShoFukada/s3_vectors_rag_hands_on

## 1. Amazon S3 Vectorsとは？

Amazon S3 Vectorsは、S3上にベクトルデータを保存して検索に利用できる機能です。2025年7月15日にプレビュー版としてリリースされました。詳細は[公式サイト](https://aws.amazon.com/jp/blogs/news/introducing-amazon-s3-vectors-first-cloud-storage-with-native-vector-support-at-scale/)を確認してください。

ベクトルデータベースは、埋め込みベクトルの型や次元数によってデータサイズが大きくなりがちです(数千次元のfloatベクトルとかになるため)。そのため専用のベクトルデータベースを使うと、運用コストがかなり高くなってしまいます。

S3 Vectorsを使えば、従来のベクトルデータベースと比べて最大90％も低いコストでベクトルデータを管理できます。検索レイテンシは1秒未満と、他のベクトルデータベース(数十msレベル)と比べると若干遅めですが、普通の業務アプリケーションであればms単位の速さが求められることはあまりないので、十分実用的だと思います。

ベクトル生成にはAmazon Titan EmbeddingsやCohereなどBedrockで提供されるモデルを利用でき、Knowledge Basesの検索バックエンドとしても使用できます。

他のサービスとの費用などの違いについては[xthxsl_mlさんの記事](https://zenn.dev/fusic/articles/14a98be48d9266)が参考になると思いますのであわせてご確認ください。

## 2. 各種インフラ作成

ではまずAWS上に必要なリソースをデプロイします。

### 2.1. 全体構成と設定

#### 2.1.1. インフラ全体構成

今回構築するシステムのリソース構成を以下の図に示します。

![リソース全体像](/images/s3-vectors-rag-chatbot/infra.png)

システムは主に以下のリソースで構成されています。

- S3バケット（ローカルからドキュメントをアップロード）
- S3ベクトルバケットとインデックス
- ナレッジベース用IAMロール
- ナレッジベース
- データソース

#### 2.1.2. 環境変数設定

環境変数の設定を行ないます。

##### 事前準備

以下の準備を事前に済ませてください。

- AWS Bedrockでモデルを有効化
  - 例えば[こちらの記事](https://zenn.dev/progdence/articles/02559735d488d4)などを参考にしてください
- AWS認証情報の設定（Access Key IDとSecret Access Key）
  - `AWS_ACCESS_KEY_ID`と`AWS_SECRET_ACCESS_KEY`を環境変数として設定してください
- S3、Bedrock、IAMへのアクセス権限

:::message
筆者はサンドボックスアカウントにて、Adminアクセス権限を持つSSOユーザーで作業を行なっています。
:::

`KNOWLEDGE_BASE_ID`と`DATA_SOURCE_ID`については、リソースをデプロイした後に記載するため、まずは空欄のままで問題ありません。

##### .envファイル

```bash
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION="us-east-1"
AWS_PROFILE=
BEDROCK_ROLE_NAME="BedrockKnowledgeBaseRole"
KNOWLEDGE_BASE_ID=
DATA_SOURCE_ID=
```

##### config.py

```python
import os
from pathlib import Path
from typing import Any

import boto3
from pydantic_settings import BaseSettings, SettingsConfigDict


def _get_aws_account_id() -> str:
    """Get AWS account ID from STS."""
    try:
        sts = boto3.client("sts")
        return sts.get_caller_identity()["Account"]
    except Exception:  # noqa: BLE001
        return "unknown"


def find_env_file() -> str | None:
    """プロジェクトルートの.envファイルを再帰的に探す"""
    current_path: Path = Path(__file__).resolve()

    for parent in current_path.parents:
        env_file: Path = parent / ".env"
        if env_file.exists():
            return str(env_file)

    return None


class Settings(BaseSettings):
    """Project settings with editable in-repo defaults.

    Except for ``OPENAI_API_KEY`` (keep it outside version control), you can hard-code
    values here for local work. Environment variables still override these defaults
    when present.
    """

    model_config = SettingsConfigDict(
        env_file=find_env_file(),
        env_file_encoding="utf-8",
    )

    OPENAI_API_KEY: str | None = None
    AWS_REGION: str
    AWS_PROFILE: str | None = None
    AWS_ACCESS_KEY_ID: str | None = None
    AWS_SECRET_ACCESS_KEY: str | None = None
    AWS_SESSION_TOKEN: str | None = None
    DOCUMENT_S3_BUCKET: str | None = None
    DOCUMENT_S3_PREFIX: str = "knowledge-base/documents/"
    VECTOR_BUCKET_NAME: str | None = None
    VECTOR_INDEX_NAME: str | None = None
    BEDROCK_EMBEDDING_MODEL_ARN: str = "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0"
    BEDROCK_EMBEDDING_DIMENSION: int = 1024
    KNOWLEDGE_BASE_NAME: str = "s3-vectors-rag-hands-on"
    BEDROCK_RESPONSE_MODEL_ARN: str = (
        "arn:aws:bedrock:us-east-1:<ACCOUNT_ID>:inference-profile/global.anthropic.claude-sonnet-4-5-20250929-v1:0"
    )
    LOCAL_DATA_DIR: str = "data/input"
    BEDROCK_ROLE_NAME: str
    KNOWLEDGE_BASE_ID: str | None  # infra.provision_all() の出力をenvから渡す
    DATA_SOURCE_ID: str | None  # infra.provision_all() の出力をenvから渡す

    def __init__(self, **data: Any) -> None:  # noqa: ANN401
        super().__init__(**data)

        account_id = _get_aws_account_id()
        if self.DOCUMENT_S3_BUCKET is None:
            self.DOCUMENT_S3_BUCKET = f"s3-vectors-rag-hands-on-documents-{account_id}"
        if self.VECTOR_BUCKET_NAME is None:
            self.VECTOR_BUCKET_NAME = f"s3-vectors-rag-hands-on-vectors-{account_id}"
        if self.VECTOR_INDEX_NAME is None:
            self.VECTOR_INDEX_NAME = f"s3-vectors-rag-hands-on-index-{account_id}"

        for field_name, field_value in self.model_dump().items():
            if field_value is not None:
                os.environ[field_name] = str(field_value)

```

pydantic-settingsを使った環境変数管理については、筆者が以前執筆した以下の記事を参考にしてください。

https://zenn.dev/medurance/articles/uv-python-starter#10.1.-%E7%92%B0%E5%A2%83%E5%A4%89%E6%95%B0%E7%AE%A1%E7%90%86%EF%BC%88pydantic-settings%EF%BC%89

なお、S3バケット名はグローバルで一意である必要があるため、AWS Account IDをサフィックスとして付与しています。その他の設定値については、後述の各種リソース作成コードで順次使用していきますので、対応関係を適宜ご確認ください。

#### 2.1.3. 使用するモデル

##### Embeddingモデル

S3 Vectorsで使用できる埋め込みモデルは以下の3種類のみです。

- `amazon.titan-embed-text-v2:0` - テキストベースの埋め込み用
- `amazon.titan-embed-image-v1` - イメージおよびマルチ画面埋め込み用
- `cohere.embed-english-v3` - 多言語および特殊なテキスト埋め込み用

今回はEmbeddingモデルとして[Amazon Titan Text Embeddings V2](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/titan-embedding-models.html)を使用します。

##### 応答生成モデル

チャットボットの応答生成には、Claude Sonnet 4.5をクロスリージョン推論で使用します。クロスリージョン推論とは、複数のAWSリージョンでモデルリソースを共有する機能で、スループットの向上や呼び出し制限の緩和が見込めます。詳細は[公式ドキュメント](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)をご覧ください。

### 2.2. S3バケット作成

ここからリソース作成のコードを順番に解説していきます。

まず、Knowledge Baseのデータソースとなるドキュメントを格納するS3バケットを作成し、サンプルドキュメントをアップロードします。`DOCUMENT_S3_BUCKET`で指定したバケット名でS3バケットを作成します。

```python
def ensure_document_bucket() -> str:
    """ドキュメント用のS3バケットが存在しない場合は作成する。

    データソースの作成時に必要なバケットARNを返す。
    """
    s3 = boto3.client("s3", region_name=settings.AWS_REGION)
    try:
        s3.head_bucket(Bucket=settings.DOCUMENT_S3_BUCKET)
    except ClientError as error:
        error_code = error.response.get("Error", {}).get("Code")
        if error_code not in {"404", "NoSuchBucket", "NotFound"}:
            raise
        create_kwargs: dict[str, Any] = {"Bucket": settings.DOCUMENT_S3_BUCKET}
        if settings.AWS_REGION != "us-east-1":
            create_kwargs["CreateBucketConfiguration"] = {
                "LocationConstraint": settings.AWS_REGION,
            }
        s3.create_bucket(**create_kwargs)

    return f"arn:aws:s3:::{settings.DOCUMENT_S3_BUCKET}"
```

次に、`LOCAL_DATA_DIR`（`data/input`）内に配置したドキュメントとメタデータファイルをS3にアップロードします。

```python
def upload_sample_documents() -> None:
    """ローカルのサンプルコーパス(ドキュメント + サイドカーメタデータファイル)をアップロードする。"""
    local_dir = Path(settings.LOCAL_DATA_DIR)
    if not local_dir.exists():
        msg = f"Local data directory not found: {local_dir}"
        raise FileNotFoundError(msg)

    s3 = boto3.client("s3", region_name=settings.AWS_REGION)
    for path in local_dir.rglob("*"):
        if not path.is_file():
            continue

        relative_key = path.relative_to(local_dir).as_posix()
        s3_key = f"{settings.DOCUMENT_S3_PREFIX}{relative_key}".replace("//", "/")
        s3.upload_file(str(path), settings.DOCUMENT_S3_BUCKET, s3_key)
```

#### メタデータフィルターについて

`<ドキュメントパス>.metadata.json`に指定形式でメタデータを記述すると、メタデータフィルターを使用できます。

https://aws.amazon.com/jp/blogs/machine-learning/amazon-bedrock-knowledge-bases-now-supports-metadata-filtering-to-improve-retrieval-accuracy/

サポートされている型は`int`、`string`、`boolean`、`string-list`の4種類のみです。`data/input`内のサンプルファイルを見るとわかりますが、日付フィルターを利用するために`published_at`をUNIXタイムスタンプで保存し、`int`型として扱う工夫をしています。

### 2.3. S3ベクトルバケット・インデックス作成

S3 Vectorsのバケットとインデックスを作成します。

Embeddingモデルに合わせてベクトルの次元数を指定する必要があります。Titan Embedding V2の場合は1024を指定します。ベクトルバケットは複数のインデックスを格納するコンテナで、インデックスは実際のベクトルデータを保持します。まずベクトルバケットを作成し、その中にインデックスを1つ作成します。

S3 Vectorsにおける検索制約について[t-motoyamaさんの記事](https://zenn.dev/mtaerohand/articles/58e74ca0cd5146)が面白かったのであわせて見てみるといいと思います。

```python
def ensure_vector_bucket_and_index() -> tuple[str, str]:
    """S3 Vectorsバケットとインデックスが存在することを確認する。

    タプル ``(vector_bucket_arn, vector_index_arn)`` を返す。
    """
    vectors = boto3.client("s3vectors", region_name=settings.AWS_REGION)

    try:
        bucket_response = vectors.get_vector_bucket(
            vectorBucketName=settings.VECTOR_BUCKET_NAME,
        )
    except vectors.exceptions.NotFoundException:
        vectors.create_vector_bucket(
            vectorBucketName=settings.VECTOR_BUCKET_NAME,
        )
        bucket_response = vectors.get_vector_bucket(
            vectorBucketName=settings.VECTOR_BUCKET_NAME,
        )

    vector_bucket = bucket_response["vectorBucket"]
    vector_bucket_arn: str = vector_bucket["vectorBucketArn"]

    try:
        index_response = vectors.get_index(
            vectorBucketName=settings.VECTOR_BUCKET_NAME,
            indexName=settings.VECTOR_INDEX_NAME,
        )
    except vectors.exceptions.NotFoundException:
        vectors.create_index(
            vectorBucketName=settings.VECTOR_BUCKET_NAME,
            indexName=settings.VECTOR_INDEX_NAME,
            dataType="float32",
            dimension=settings.BEDROCK_EMBEDDING_DIMENSION,
            distanceMetric="cosine",
        )
        index_response = vectors.get_index(
            vectorBucketName=settings.VECTOR_BUCKET_NAME,
            indexName=settings.VECTOR_INDEX_NAME,
        )

    return vector_bucket_arn, index_response["index"]["indexArn"]
```

### 2.4. ナレッジベース用IAMロール作成

ナレッジベースにアタッチするIAMロールを作成します。

このロールには以下のポリシーを設定します。

- **Identity-based policies**
  - ドキュメントバケットとベクトルバケットへのアクセスポリシー
  - Bedrockの各種モデルへのアクセスポリシー
- **Resource-based policies（信頼ポリシー）**
  - `bedrock.amazonaws.com`をプリンシパルとして指定

IAMポリシーの詳細については[公式ドキュメント](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)を参照してください。

```python
def ensure_bedrock_kb_role(
    document_bucket_arn: str,
    vector_bucket_arn: str,
    vector_index_arn: str,
) -> str:
    """Bedrock Knowledge Base用のIAMロールを作成または取得する。

    ロールを最新のポリシーに更新し、ARNを返す。
    """
    iam = boto3.client("iam")
    role_name = settings.BEDROCK_ROLE_NAME

    try:
        response = iam.get_role(RoleName=role_name)
        role_arn = response["Role"]["Arn"]
    except iam.exceptions.NoSuchEntityException:
        trust_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {"Service": "bedrock.amazonaws.com"},
                    "Action": "sts:AssumeRole",
                }
            ],
        }

        role_response = iam.create_role(
            RoleName=role_name,
            AssumeRolePolicyDocument=json.dumps(trust_policy),
            Description="Role for Bedrock Knowledge Base to access S3 and models",
        )

        role_arn = role_response["Role"]["Arn"]

    s3_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": ["s3:GetObject", "s3:ListBucket"],
                "Resource": [
                    document_bucket_arn,
                    f"{document_bucket_arn}/*",
                ],
            }
        ],
    }
    iam.put_role_policy(
        RoleName=role_name,
        PolicyName="S3Access",
        PolicyDocument=json.dumps(s3_policy),
    )

    s3vectors_resources = [
        vector_bucket_arn,
        f"{vector_bucket_arn}/*",
    ]
    if vector_index_arn:
        s3vectors_resources.append(vector_index_arn)
        s3vectors_resources.append(f"{vector_bucket_arn}/index/*")

    s3vectors_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": ["s3vectors:*"],
                "Resource": s3vectors_resources,
            }
        ],
    }
    iam.put_role_policy(
        RoleName=role_name,
        PolicyName="S3VectorsAccess",
        PolicyDocument=json.dumps(s3vectors_policy),
    )

    bedrock_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": ["bedrock:InvokeModel"],
                "Resource": "*",
            }
        ],
    }
    iam.put_role_policy(
        RoleName=role_name,
        PolicyName="BedrockModelAccess",
        PolicyDocument=json.dumps(bedrock_policy),
    )

    return role_arn
```

### 2.5. ナレッジベースの作成

ベクトルバケットとインデックスのARNを指定してBedrock Knowledge Baseを作成します。

Embeddingモデルはこのステップで指定します。ナレッジベースは、ドキュメントバケットから文書を取得し、Embeddingモデルでベクトル化してベクトルバケットに同期する役割を担います。

```python
def get_or_create_knowledge_base(
    vector_bucket_arn: str,
    vector_index_arn: str,
    role_arn: str,
) -> str:
    """ナレッジベースが存在しない場合は作成する。"""
    bedrock_agents = boto3.client("bedrock-agent", region_name=settings.AWS_REGION)

    paginator = bedrock_agents.get_paginator("list_knowledge_bases")
    for page in paginator.paginate():
        for kb in page.get("knowledgeBaseSummaries", []):
            if kb.get("name") == settings.KNOWLEDGE_BASE_NAME:
                return kb["knowledgeBaseId"]

    response = bedrock_agents.create_knowledge_base(
        name=settings.KNOWLEDGE_BASE_NAME,
        roleArn=role_arn,
        knowledgeBaseConfiguration={
            "type": "VECTOR",
            "vectorKnowledgeBaseConfiguration": {
                "embeddingModelArn": settings.BEDROCK_EMBEDDING_MODEL_ARN,
            },
        },
        storageConfiguration={
            "type": "S3_VECTORS",
            "s3VectorsConfiguration": {
                "vectorBucketArn": vector_bucket_arn,
                "indexArn": vector_index_arn,
            },
        },
    )
    return response["knowledgeBase"]["knowledgeBaseId"]
```

### 2.6. データソースの作成

ナレッジベースに対してS3バケットをデータソースとしてアタッチします。

チャンキング戦略やパーサーはこのステップで指定できます（今回はデフォルトを使用）。デフォルトでは、Amazon Bedrockのデフォルトパーサーとデフォルトチャンキング（約300トークンのチャンクサイズにBedrockが自動分割）が採用されます。

なお、S3 Vectorsには以下の制約があります。

- ハイブリッド検索は未サポート
- メタデータは最大2KB
- チャンクは最大500トークン
- float型ベクトルのみサポート

詳細は[公式APIリファレンス](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent_CreateDataSource.html)を参照してください。

```python

def get_or_create_data_source(knowledge_base_id: str, document_bucket_arn: str) -> str:
    """必要に応じてS3データソースをナレッジベースにアタッチする。"""
    bedrock_agents = boto3.client("bedrock-agent", region_name=settings.AWS_REGION)
    paginator = bedrock_agents.get_paginator("list_data_sources")
    for page in paginator.paginate(knowledgeBaseId=knowledge_base_id):
        for data_source in page.get("dataSourceSummaries", []):
            if data_source.get("name") == "s3-sample-documents":
                return data_source["dataSourceId"]

    response = bedrock_agents.create_data_source(
        knowledgeBaseId=knowledge_base_id,
        name="s3-sample-documents",
        description="Sample documents uploaded from data/input",
        dataSourceConfiguration={
            "type": "S3",
            "s3Configuration": {
                "bucketArn": document_bucket_arn,
                "inclusionPrefixes": [settings.DOCUMENT_S3_PREFIX],
            },
        },
    )
    return response["dataSource"]["dataSourceId"]
```

### 2.7. 実行

それでは、これまでのコードを統合したスクリプト全体を実行してみます。事前に`uv sync`でパッケージをインストールしておいてください。

コード全体は以下のリポジトリで確認できます。

https://github.com/ShoFukada/s3_vectors_rag_hands_on/blob/main/src/s3_vectors_rag_hands_on/infra.py

#### 実行例

```bash
❯ uv run -m s3_vectors_rag_hands_on.infra
[SUCCESS] document bucket
[SUCCESS] sample document upload
[SUCCESS] S3 Vectors bucket and index
[SUCCESS] Bedrock knowledge base role
[SUCCESS] knowledge base
[SUCCESS] data source
{
  "knowledge_base_id": "<KNOWLEDGE_BASE_ID>",
  "data_source_id": "<DATA_SOURCE_ID>",
  "vector_bucket_arn": "arn:aws:s3vectors:us-east-1:<ACCOUNT_ID>:bucket/s3-vectors-rag-hands-on-vectors-<ACCOUNT_ID>",
  "vector_index_arn": "arn:aws:s3vectors:us-east-1:<ACCOUNT_ID>:bucket/s3-vectors-rag-hands-on-vectors-<ACCOUNT_ID>/index/s3-vectors-rag-hands-on-index-<ACCOUNT_ID>"
}
```

実行後に出力された`knowledge_base_id`と`data_source_id`は`.env`ファイルに反映しておきましょう（3章以降で使用します）。

## 3. ドキュメントの同期

現段階では、ドキュメントの内容はまだベクトルバケットに保存されていません（Embeddingしてベクトルとして保存されていない状態）。以下のコマンドで確認できます（`{account_id}`は自分のAWS Account IDに置き換えてください）。

```bash
❯ aws s3vectors list-vectors \
  --vector-bucket-name "s3-vectors-rag-hands-on-vectors-{account_id}" \
  --index-name "s3-vectors-rag-hands-on-index-{account_id}" \
  --segment-count 2 \
  --segment-index 0 \
  --return-data \
  --return-metadata
{
    "vectors": []
}
```

### 3.1. 同期処理の実装

データソースとナレッジベースを同期することで、ドキュメントバケットの内容がベクトルバケットのインデックスにEmbeddingされて格納され、ナレッジベースから検索可能になります。

同期ジョブはポーリングで実行状態を確認する必要があるため、以下のような流れで実装します。

1. 同期ジョブを開始
2. ジョブIDを使って実行状態を取得
3. 完了するまで定期的にポーリング
4. 完了フラグが返ってきたら終了

デバッグ用のログ出力などを含むため、コードは少し長めですが、基本的な流れはシンプルです。

```python
from __future__ import annotations

import time
import uuid
from typing import TYPE_CHECKING, Any, Literal

import boto3

if TYPE_CHECKING:
    from collections.abc import Callable

from .config import Settings

settings = Settings()

IngestionStatus = Literal[
    "STARTING",
    "IN_PROGRESS",
    "COMPLETE",
    "FAILED",
    "STOPPING",
    "STOPPED",
]


def start_sync(knowledge_base_id: str, data_source_id: str) -> str:
    """インジェストジョブを開始し、その識別子を返す。"""
    client = boto3.client("bedrock-agent", region_name=settings.AWS_REGION)
    response = client.start_ingestion_job(
        knowledgeBaseId=knowledge_base_id,
        dataSourceId=data_source_id,
        clientToken=str(uuid.uuid4()),
    )
    return response["ingestionJob"]["ingestionJobId"]


def wait_for_sync(
    knowledge_base_id: str,
    data_source_id: str,
    ingestion_job_id: str,
    *,
    poll_seconds: float = 20.0,
    timeout_seconds: float = 3600.0,
    on_update: Callable[[IngestionStatus, dict[str, Any]], None] | None = None,
) -> dict[str, Any]:
    """インジェストジョブが完了またはタイムアウトするまでポーリングする。"""
    client = boto3.client("bedrock-agent", region_name=settings.AWS_REGION)
    waited = 0.0

    while waited <= timeout_seconds:
        response = client.get_ingestion_job(
            knowledgeBaseId=knowledge_base_id,
            dataSourceId=data_source_id,
            ingestionJobId=ingestion_job_id,
        )
        job = response["ingestionJob"]
        status: IngestionStatus = job["status"]
        if on_update:
            on_update(status, job)
        if status in {"COMPLETE", "FAILED", "STOPPED"}:
            return job
        time.sleep(poll_seconds)
        waited += poll_seconds

    msg = f"Ingestion job {ingestion_job_id} did not finish within {timeout_seconds} seconds"
    raise TimeoutError(
        msg,
    )


def sync_data_source(knowledge_base_id: str, data_source_id: str) -> dict[str, Any]:
    """インジェストジョブを開始し、完了を待つ。"""
    ingestion_job_id = start_sync(knowledge_base_id, data_source_id)
    return wait_for_sync(knowledge_base_id, data_source_id, ingestion_job_id)


def _format_stat(value: int | None) -> str:
    """統計値をフォーマットする。値がNoneの場合は'?'を返す。"""
    return "?" if value is None else str(value)


def main() -> None:
    """メインエントリーポイント。インジェストジョブを開始し、完了まで監視する。"""
    knowledge_base_id = settings.KNOWLEDGE_BASE_ID
    data_source_id = settings.DATA_SOURCE_ID

    if not knowledge_base_id or not data_source_id:
        raise SystemExit(
            "Knowledge base ID and data source ID must be configured in Settings before running sync.",
        )

    print(
        f"Starting ingestion job for knowledge base {knowledge_base_id} and data source {data_source_id}…",
        flush=True,
    )
    ingestion_job_id = start_sync(knowledge_base_id, data_source_id)
    print(
        f"Started ingestion job {ingestion_job_id}. Polling every 20.0 seconds…",
        flush=True,
    )

    last_status: IngestionStatus | None = None

    def on_update(status: IngestionStatus, job: dict[str, Any]) -> None:
        nonlocal last_status
        if status != last_status:
            stats = job.get("statistics", {})
            scanned = _format_stat(stats.get("numberOfDocumentsScanned"))
            failed = _format_stat(stats.get("numberOfDocumentsFailed"))
            timestamp = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
            print(
                f"[{timestamp}] status={status} scanned={scanned} failed={failed}",
                flush=True,
            )
            last_status = status

    try:
        final_job = wait_for_sync(
            knowledge_base_id,
            data_source_id,
            ingestion_job_id,
            on_update=on_update,
        )
    except TimeoutError as exc:
        print(str(exc), flush=True)
        raise SystemExit(1) from exc

    status: IngestionStatus = final_job["status"]
    stats = final_job.get("statistics", {})
    failures = final_job.get("failureReasons", []) or []

    print("Ingestion summary:")
    print(f"  status: {status}")
    print(
        "  documents: "
        f"scanned={_format_stat(stats.get('numberOfDocumentsScanned'))} "
        f"indexed_new={_format_stat(stats.get('numberOfNewDocumentsIndexed'))} "
        f"indexed_modified={_format_stat(stats.get('numberOfModifiedDocumentsIndexed'))} "
        f"failed={_format_stat(stats.get('numberOfDocumentsFailed'))}"
    )

    exit_code = 0 if status == "COMPLETE" and not failures else 1
    raise SystemExit(exit_code)


if __name__ == "__main__":
    main()
```

### 3.2. 同期の実行

それでは実際に同期処理を実行してみます。

#### 実行例

```bash
❯ uv run -m s3_vectors_rag_hands_on.sync
Starting ingestion job for knowledge base <KNOWLEDGE_BASE_ID> and data source <DATA_SOURCE_ID>…
Started ingestion job <INGESTION_JOB_ID>. Polling every 20.0 seconds…
[2025-10-03 22:52:25] status=IN_PROGRESS scanned=5 failed=0
[2025-10-03 22:52:45] status=COMPLETE scanned=5 failed=0
Ingestion summary:
  status: COMPLETE
  documents: scanned=5 indexed_new=5 indexed_modified=0 failed=0
```

### 3.3. 同期結果の確認

同期が正常に完了しました。次に、ベクトルが実際にバケットに格納されているかをCLIで確認してみます（`{account_id}`は自分のAWS Account IDに置き換えてください）。

```bash
❯ aws s3vectors list-vectors \
  --vector-bucket-name "s3-vectors-rag-hands-on-vectors-{account_id}" \
  --index-name "s3-vectors-rag-hands-on-index-{account_id}" \
  --segment-count 2 \
  --segment-index 0 \
  --return-data \
  --return-metadata
{
    "vectors": [
        {
            "key": "de17f656-4326-4977-a51e-48a985e8787a",
            "data": {
                "float32": [
                    -0.018628833815455437,
                    0.0012931098463013768,
                    0.04644904285669327,
                    -0.024917058646678925,
                    ...（中略）
```

同期が完了し、ベクトルがバケットに反映されていることが確認できました。

なお、今回はSDKから同期を実行しましたが、AWS Management Consoleの同期ボタンを使用することももちろん可能です。

## 4. RAG

ここからは実際にRAGを実行してみます。

### 4.1. 検索処理の実装

まず、ナレッジベースに対して質問を投げて回答を得るための関数を実装します。

```python
def ask_knowledge_base(
    question: str,
    knowledge_base_id: str,
    *,
    metadata_filter: dict[str, Any] | None = None,
    number_of_results: int = 4,
    search_type: str | None = None,
) -> dict[str, Any]:
    """Run a single RetrieveAndGenerate request and return the raw payload."""
    runtime = boto3.client("bedrock-agent-runtime", region_name=settings.AWS_REGION)

    vector_config: dict[str, Any] = {"numberOfResults": number_of_results}
    if metadata_filter:
        vector_config["filter"] = metadata_filter
    if search_type:
        vector_config["overrideSearchType"] = search_type

    retrieval_configuration = {"vectorSearchConfiguration": vector_config}

    return runtime.retrieve_and_generate(
        input={"text": question},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {
                "knowledgeBaseId": knowledge_base_id,
                "modelArn": settings.BEDROCK_RESPONSE_MODEL_ARN,
                "retrievalConfiguration": retrieval_configuration,
            },
        },
    )
```

検索のコア部分の関数は上記のようになります。処理の流れは以下のとおりです。

1. `bedrock-agent-runtime`クライアントを起動
2. ベクトル検索設定として取得件数とメタデータフィルターを指定
3. ナレッジベースIDと応答生成用のモデルARN（前述のクロスリージョンSonnet 4.5）を指定
4. `retrieve_and_generate`を実行して回答を取得

### 4.2. メタデータフィルターの実装

次に、サンプルドキュメントに合わせてメタデータフィルターを適用した検索を実装します。

```python
from __future__ import annotations

import json
from dataclasses import dataclass
from typing import Any

import boto3

from .config import Settings

settings = Settings()


def equals_filter(key: str, value: str | float) -> dict[str, dict[str, str | bool | int | float]]:
    """Create an ``equals`` metadata filter clause for a knowledge base query."""
    return {"equals": {"key": key, "value": value}}


def equals_filter_bool(key: str, *, value: bool) -> dict[str, dict[str, str | bool | int | float]]:
    """Create an ``equals`` metadata filter clause with a boolean value."""
    return {"equals": {"key": key, "value": value}}


def greater_or_equals_filter(key: str, value: float) -> dict[str, dict[str, str | int | float]]:
    """Create a ``greaterThanOrEquals`` metadata filter clause."""
    return {"greaterThanOrEquals": {"key": key, "value": value}}


def and_all(*conditions: dict[str, Any]) -> dict[str, Any]:
    """Combine multiple conditions using ``andAll`` semantics."""
    return {"andAll": [condition for condition in conditions if condition]}


def ask_knowledge_base(
    question: str,
    knowledge_base_id: str,
    *,
    metadata_filter: dict[str, Any] | None = None,
    number_of_results: int = 4,
    search_type: str | None = None,
) -> dict[str, Any]:
    """Run a single RetrieveAndGenerate request and return the raw payload."""
    runtime = boto3.client("bedrock-agent-runtime", region_name=settings.AWS_REGION)

    vector_config: dict[str, Any] = {"numberOfResults": number_of_results}
    if metadata_filter:
        vector_config["filter"] = metadata_filter
    if search_type:
        vector_config["overrideSearchType"] = search_type

    retrieval_configuration = {"vectorSearchConfiguration": vector_config}

    return runtime.retrieve_and_generate(
        input={"text": question},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {
                "knowledgeBaseId": knowledge_base_id,
                "modelArn": settings.BEDROCK_RESPONSE_MODEL_ARN,
                "retrievalConfiguration": retrieval_configuration,
            },
        },
    )


MAX_REFERENCE_PREVIEW_LENGTH = 120


@dataclass(slots=True)
class QueryScenario:
    """Simple container for scripted RAG test cases."""

    label: str
    question: str
    metadata_filter: dict[str, Any] | None = None
    number_of_results: int = 4


def _print_response(label: str, response: dict[str, Any]) -> None:
    answer = response.get("output", {}).get("text", "(no answer)")
    print(f"=== {label} ===")
    print(answer)

    citations = response.get("citations", [])
    if not citations:
        print("(no citations returned)")
        print()
        return

    print("Sources:")
    for citation in citations:
        references = citation.get("retrievedReferences", [])
        for reference in references:
            ref = reference.get("content", {}).get("text", "")
            location = reference.get("location", {}).get("s3Location", {})
            uri = location.get("uri", "")
            preview = ref[:MAX_REFERENCE_PREVIEW_LENGTH]
            ellipsis = "…" if len(ref) > MAX_REFERENCE_PREVIEW_LENGTH else ""
            print(f"  - {uri}: {preview}{ellipsis}")
    print()


def run_scenarios(knowledge_base_id: str) -> None:
    """Execute a curated set of queries to validate metadata filtering."""
    base_filters = {
        "domain": equals_filter("domain", "auroradynamics.com"),
        "is_internal_true": equals_filter_bool("is_internal", value=True),
        "is_internal_false": equals_filter_bool("is_internal", value=False),
        "security_tags": equals_filter("tags", "security,governance"),
        "press_tags": equals_filter("tags", "press,announcement"),
        "published_after_2025": greater_or_equals_filter("published_at", 1_750_000_000),
        "metrics_tags": equals_filter("tags", "metrics,operations"),
        "catalog_tags": equals_filter("tags", "services,catalog"),
    }

    scenarios = [
        QueryScenario(
            label="No filter overview",
            question="Give me a broad overview of Aurora Dynamics as described in our knowledge base.",
        ),
        QueryScenario(
            label="Domain filter",
            question="Summarize the public description of Aurora Dynamics.",
            metadata_filter=base_filters["domain"],
        ),
        QueryScenario(
            label="Internal security docs",
            question="What security governance guidance is available?",
            metadata_filter=and_all(
                base_filters["is_internal_true"],
                base_filters["security_tags"],
            ),
        ),
        QueryScenario(
            label="Press announcement",
            question="What recent announcement did Aurora make?",
            metadata_filter=and_all(
                base_filters["is_internal_false"],
                base_filters["press_tags"],
                base_filters["published_after_2025"],
            ),
        ),
        QueryScenario(
            label="Metrics spotlight",
            question="Share key operational metrics for Aurora Dynamics.",
            metadata_filter=and_all(
                base_filters["is_internal_true"],
                base_filters["metrics_tags"],
            ),
        ),
        QueryScenario(
            label="Catalog lookup (hybrid search)",
            question="List the solution offerings Aurora provides to customers.",
            metadata_filter=base_filters["catalog_tags"],
        ),
    ]

    print("Running scripted knowledge base checks…", flush=True)

    for scenario in scenarios:
        try:
            response = ask_knowledge_base(
                scenario.question,
                knowledge_base_id,
                metadata_filter=scenario.metadata_filter,
                number_of_results=scenario.number_of_results,
            )
        except Exception as exc:  # noqa: BLE001
            print(f"=== {scenario.label} ===")
            print(f"Query failed: {exc}")
            if scenario.metadata_filter:
                print("Filter used:", json.dumps(scenario.metadata_filter, indent=2))
            print()
            continue

        _print_response(scenario.label, response)


if __name__ == "__main__":
    kb_id = settings.KNOWLEDGE_BASE_ID
    if not kb_id:
        raise SystemExit("Knowledge base ID must be configured in Settings before running scripted checks.")

    run_scenarios(kb_id)

```

メタデータフィルターの詳しい書き方については、以下のAWS公式ブログが参考になります。

https://aws.amazon.com/jp/blogs/news/knowledge-bases-for-amazon-bedrock-now-supports-metadata-filtering-to-improve-retrieval-accuracy/

### 4.3. 実行と検証

それでは実際にRAGを実行してみます。

#### 実行例

```bash
❯ uv run -m s3_vectors_rag_hands_on.chatbot
Running scripted knowledge base checks…
=== No filter overview ===
Aurora Dynamics is a consulting firm that specializes in architecting secure AWS-native platforms for data-intensive industries. The company is headquartered in Seattle with delivery centers in Dublin, Singapore, and Toronto. Their consulting services span strategy, migration, and managed services with 24/7 response capabilities. The company serves key industries including biopharma, retail operations, and climate-tech logistics. They have recently expanded their climate-tech practice to help logistics and energy enterprises meet decarbonization targets through geospatial analytics, IoT telemetry mapping, and AI-assisted forecasting. Aurora Dynamics offers modular solution plays including LaunchPad Migration for phased migrations using Control Tower, a Data Fabric Suite built with AWS Glue and Lake Formation, and Intelligent Ops featuring AI copilots with Amazon Bedrock integration. Their engagement model includes discovery framing, shared delivery pods, and customer enablement sprints. Key differentiators include governed lakehouse blueprints, responsible AI accelerators, and FinOps maturity frameworks. The company maintains strong security controls with a zero-trust reference architecture aligned with AWS Well-Architected best practices, continuous compliance monitoring, and quarterly penetration testing for customer environments.
Sources:
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_company_profile.pdf: Aurora Dynamics Company Profile Aurora Dynamics architects secure AWS-native platforms for data-intensive industries. Ou…
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_press_announcement.txt: Aurora Dynamics Expands Climate-Tech Practice SEATTLE – Aurora Dynamics today announced a dedicated climate-tech innovat…
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_company_profile.pdf: Aurora Dynamics Company Profile Aurora Dynamics architects secure AWS-native platforms for data-intensive industries. Ou…
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_solution_catalog.docx: Aurora Dynamics curates modular solution plays to accelerate AWS adoption for regulated enterprises.  LaunchPad Migratio…
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_security_brief.pdf: Aurora Dynamics Security Controls Brief Security guild maintains a zero-trust reference aligned with AWS Well-Architecte…
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_company_profile.pdf: Aurora Dynamics Company Profile Aurora Dynamics architects secure AWS-native platforms for data-intensive industries. Ou…

=== Domain filter ===
Aurora Dynamics is a consulting firm that architects secure AWS-native platforms for data-intensive industries. The company provides services across strategy, migration, and managed services with 24/7 response capabilities. They serve key industries including biopharma, retail operations, and climate-tech logistics. Their differentiators include governed lakehouse blueprints, responsible AI accelerators, and FinOps maturity frameworks. The company is headquartered in Seattle with delivery centers in Dublin, Singapore, and Toronto.
Sources:
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_company_profile.pdf: Aurora Dynamics Company Profile Aurora Dynamics architects secure AWS-native platforms for data-intensive industries. Ou…

=== Internal security docs ===
Aurora Dynamics provides security governance guidance through a security guild that maintains a zero-trust reference architecture aligned with AWS Well-Architected best practices. The governance framework includes continuous compliance monitoring using AWS Config conformance packs and GuardDuty delegated administration. Additionally, customer vault environments undergo quarterly penetration testing and resilience game-days. For incident response, there are playbooks that integrate with Security Lake and cross-account Detective graphing capabilities.
Sources:
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_security_brief.pdf: Aurora Dynamics Security Controls Brief Security guild maintains a zero-trust reference aligned with AWS Well-Architecte…

=== Press announcement ===
Sorry, I am unable to assist you with this request.
(no citations returned)

=== Metrics spotlight ===
Aurora Dynamics has established revenue targets across four major regions for FY25. North America leads with a target of $86M USD and a projected compound annual growth rate of 21%, focusing on LaunchPad Migration and FinOps Studio offerings. EMEA follows with a $54M USD target and 18% CAGR, emphasizing Data Fabric Suite and Sovereign Cloud Advisory services. The APAC region has set a $47M USD target with the highest growth rate at 24% CAGR, concentrating on Smart Factory Edge and Intelligent Ops solutions. LATAM has a $22M USD revenue target with 15% CAGR, prioritizing Climate Analytics Hub and Managed Services.
Sources:
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_operational_metrics.xlsx: Regional Forecast             Region          FY25 Revenue Target (USD M)                Projected CAGR          Primary Offerings               North America           86              21%             L…

=== Catalog lookup (hybrid search) ===
Aurora Dynamics provides three main solution offerings to customers:

1. LaunchPad Migration - A phased migration factory that uses Control Tower guardrails and Infrastructure as Code accelerators to help customers migrate to AWS.

2. Data Fabric Suite - A multi-zone lakehouse reference architecture built using AWS Glue, Lake Formation, and Redshift Serverless.

3. Intelligent Ops - An event-driven runbook solution with AI copilots that integrates Amazon Bedrock-based assistants to support engineers.

These solutions are designed to accelerate AWS adoption for regulated enterprises through modular solution plays.
Sources:
  - s3://s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>/knowledge-base/documents/aurora_solution_catalog.docx: Aurora Dynamics curates modular solution plays to accelerate AWS adoption for regulated enterprises.  LaunchPad Migratio…
```

### 4.4. 検証結果の確認

それぞれのフィルターに応じたドキュメントが正しく取得できていることが確認できました。各シナリオの検証結果は以下のとおりです。

1. **No filter overview**
   - クエリ: "Give me a broad overview of Aurora Dynamics as described in our knowledge base."
   - フィルター: なし
   - 主な引用: `aurora_company_profile.pdf`ほかプレスリリース・カタログ・セキュリティ資料
   - 判定: ✔ データ全体を横断した総合サマリーが生成され、複数ドキュメントが引用された

2. **Domain filter**
   - クエリ: "Summarize the public description of Aurora Dynamics."
   - フィルター: `equals(domain, "auroradynamics.com")`
   - 主な引用: `aurora_company_profile.pdf`
   - 判定: ✔ 指定ドメインの公開資料のみから回答が生成された

3. **Internal security docs**
   - クエリ: "What security governance guidance is available?"
   - フィルター: `equals(is_internal, true)` AND `equals(tags, "security,governance")`
   - 主な引用: `aurora_security_brief.pdf`
   - 判定: ✔ 社内向けセキュリティ資料のみがヒットし、想定どおりの回答が返った

4. **Press announcement**
   - クエリ: "What recent announcement did Aurora make?"
   - フィルター: `equals(is_internal, false)` AND `equals(tags, "press,announcement")` AND `greaterThanOrEquals(published_at, 1750000000)`
   - 主な引用: `aurora_press_announcement.txt`
   - 判定: ✔ 公開プレスアナウンスのみを対象に、日付条件を満たす内容が返された

5. **Metrics spotlight**
   - クエリ: "Share key operational metrics for Aurora Dynamics."
   - フィルター: `equals(is_internal, true)` AND `equals(tags, "metrics,operations")`
   - 主な引用: `aurora_operational_metrics.xlsx`
   - 判定: ✔ メトリクス資料から数値指標が返され、他ドキュメントは引用されなかった

6. **Catalog lookup**
   - クエリ: "List the solution offerings Aurora provides to customers."
   - フィルター: `equals(tags, "services,catalog")`
   - 主な引用: `aurora_solution_catalog.docx`
   - 判定: ✔ ソリューションカタログのみが参照され、想定の3つのモジュラー提供内容を回答

## 5. リソースのクリーンアップ

最後に、作成したリソースを削除します。クリーンアップもコード化しているため、実行するだけで完了します。

### 5.1. 削除時の注意事項

Bedrockはデフォルトでベクトルストアをパージしようとしますが、S3 Vectorsリソースがすでに存在しない場合でもデータソース削除が成功するように、`dataDeletionPolicy`を`RETAIN`に上書きしています。

詳細については以下の記事が参考になります。

https://qiita.com/Ichiyama_san/items/25c50a67bddbe63ab567

### 5.2. クリーンアップコード

クリーンアップコードは、infraで作成したリソースを順番に削除していきます。

```python
from __future__ import annotations

import sys
from dataclasses import dataclass
from typing import TYPE_CHECKING

import boto3
from botocore.exceptions import ClientError

from .config import Settings

if TYPE_CHECKING:
    from collections.abc import Iterable

    from botocore.client import BaseClient

settings = Settings()


def _client(service_name: str) -> BaseClient:
    """設定されたリージョンを使用してboto3クライアントを作成する。"""
    return boto3.client(service_name, region_name=settings.AWS_REGION)


@dataclass
class CleanupSummary:
    """最終レポートと終了コードのための結果フラグを収集する。"""

    knowledge_base_deleted: bool | None = None
    data_source_deleted: bool | None = None
    vector_index_deleted: bool | None = None
    vector_bucket_deleted: bool | None = None
    documents_deleted: int = 0
    document_bucket_deleted: bool | None = None
    iam_role_deleted: bool | None = None

    def failures(self) -> list[str]:
        """失敗を表すキーのリストを返す。"""
        failure_keys: list[str] = []
        for key, value in self.__dict__.items():
            if key == "documents_deleted":
                continue
            if value is False:
                failure_keys.append(key)
        return failure_keys


@dataclass
class DataSourceInfo:
    """更新/削除呼び出しに必要なデータソースの情報。"""

    knowledge_base_id: str
    data_source_id: str
    name: str
    configuration: dict
    description: str | None = None


def resolve_knowledge_base_id() -> str | None:
    """設定で提供されたKnowledge Base IDが存在する場合はそれを返す。"""
    kb_id = settings.KNOWLEDGE_BASE_ID
    if not kb_id:
        raise ValueError("Settings.KNOWLEDGE_BASE_ID が未設定です。config.py または .env に ID を記載してください。")

    client = _client("bedrock-agent")
    try:
        client.get_knowledge_base(knowledgeBaseId=kb_id)
    except client.exceptions.ResourceNotFoundException:
        print("[cleanup] WARN  設定済みの Knowledge Base ID は既に削除済みのようです")
        return None

    print(f"[cleanup] 設定値の Knowledge Base ID を利用します: {kb_id}")
    return kb_id


def resolve_data_source(knowledge_base_id: str) -> DataSourceInfo | None:
    """設定のIDを使用してデータソースメタデータを返す(存在する場合)。"""
    data_source_id = settings.DATA_SOURCE_ID
    if not data_source_id:
        raise ValueError("Settings.DATA_SOURCE_ID が未設定です。config.py または .env に ID を記載してください。")

    client = _client("bedrock-agent")
    try:
        response = client.get_data_source(
            knowledgeBaseId=knowledge_base_id,
            dataSourceId=data_source_id,
        )
    except client.exceptions.ResourceNotFoundException:
        print("[cleanup] WARN  設定済みの Data Source ID は既に削除済みのようです")
        return None

    data = response["dataSource"]
    info = DataSourceInfo(
        knowledge_base_id=knowledge_base_id,
        data_source_id=data_source_id,
        name=data["name"],
        configuration=data["dataSourceConfiguration"],
        description=data.get("description"),
    )
    print(f"[cleanup] Data Source を特定しました: {info.data_source_id} (name={info.name})")
    return info


def delete_data_source(info: DataSourceInfo) -> bool | None:
    """ベクトルデータをRETAINモードに設定した後、データソースを削除する。"""
    client = _client("bedrock-agent")
    print(
        f"[cleanup] データソース削除を開始します "
        f"(knowledge_base_id={info.knowledge_base_id}, data_source_id={info.data_source_id})",
        flush=True,
    )

    # Bedrockはデフォルトでベクトルストアをパージしようとする。
    # S3 Vectorsリソースが既に存在しない場合でも削除が成功するようにRETAINに上書きする。
    update_kwargs = {
        "knowledgeBaseId": info.knowledge_base_id,
        "dataSourceId": info.data_source_id,
        "name": info.name,
        "dataSourceConfiguration": info.configuration,
        "dataDeletionPolicy": "RETAIN",
    }
    if info.description is not None:
        update_kwargs["description"] = info.description

    try:
        client.update_data_source(**update_kwargs)
        print("[cleanup] dataDeletionPolicy を RETAIN に更新しました")
    except client.exceptions.ResourceNotFoundException:
        print("[cleanup] WARN  データソースが見つからなかったため更新をスキップします")
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        print(f"[cleanup] WARN  dataDeletionPolicy の更新に失敗しましたが削除を試みます: {code}")
        print(f"[cleanup] WARN  {error}")

    try:
        client.delete_data_source(
            knowledgeBaseId=info.knowledge_base_id,
            dataSourceId=info.data_source_id,
        )
    except client.exceptions.ResourceNotFoundException:
        print("[cleanup] WARN  データソースは既に削除済みでした")
        return None
    except client.exceptions.ConflictException:
        print("[cleanup] WARN  データソースの削除処理が既に進行中です")
        return True
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        print(f"[cleanup] ERROR データソース削除に失敗しました: {code}")
        print(f"[cleanup] ERROR {error}")
        return False
    else:
        print("[cleanup] データソース削除完了")
        return True


def delete_knowledge_base(knowledge_base_id: str) -> bool | None:
    """Knowledge Baseを削除する。"""
    client = _client("bedrock-agent")
    print(f"[cleanup] Knowledge Base 削除を開始します (id={knowledge_base_id})")
    try:
        client.delete_knowledge_base(knowledgeBaseId=knowledge_base_id)
    except client.exceptions.ResourceNotFoundException:
        print("[cleanup] WARN  Knowledge Base は既に削除済みでした")
        return None
    except client.exceptions.ConflictException:
        print("[cleanup] WARN  Knowledge Base の削除処理が既に進行中です")
        return True
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        print(f"[cleanup] ERROR Knowledge Base 削除に失敗しました: {code}")
        print(f"[cleanup] ERROR {error!s}")
        return False
    else:
        print("[cleanup] Knowledge Base 削除完了")
        return True


def delete_vector_index() -> bool | None:
    """S3 Vectorsインデックスを削除する。"""
    client = _client("s3vectors")
    print(
        f"[cleanup] S3 Vectors インデックス削除を開始します "
        f"(bucket={settings.VECTOR_BUCKET_NAME}, index={settings.VECTOR_INDEX_NAME})",
        flush=True,
    )
    try:
        client.delete_index(
            vectorBucketName=settings.VECTOR_BUCKET_NAME,
            indexName=settings.VECTOR_INDEX_NAME,
        )
    except client.exceptions.NotFoundException:
        print("[cleanup] WARN  S3 Vectors インデックスは既に存在しませんでした")
        return None
    except client.exceptions.ConflictException:
        print("[cleanup] WARN  S3 Vectors インデックス削除が競合しました (再試行で解消する場合があります)")
        return False
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        print(f"[cleanup] ERROR S3 Vectors インデックス削除に失敗しました: {code}")
        print(f"[cleanup] ERROR {error!s}")
        return False
    else:
        print("[cleanup] S3 Vectors インデックス削除完了")
        return True


def delete_vector_bucket() -> bool | None:
    """S3 Vectorsバケットを削除する。"""
    client = _client("s3vectors")
    print(f"[cleanup] S3 Vectors バケット削除を開始します (name={settings.VECTOR_BUCKET_NAME})")
    try:
        client.delete_vector_bucket(vectorBucketName=settings.VECTOR_BUCKET_NAME)
    except client.exceptions.NotFoundException:
        print("[cleanup] WARN  S3 Vectors バケットは既に存在しませんでした")
        return None
    except client.exceptions.ConflictException:
        print("[cleanup] WARN  S3 Vectors バケット削除が競合しました。残っているインデックスがないか確認してください")
        return False
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        print(f"[cleanup] ERROR S3 Vectors バケット削除に失敗しました: {code}")
        print(f"[cleanup] ERROR {error!s}")
        return False
    else:
        print("[cleanup] S3 Vectors バケット削除完了")
        return True


def _chunk_delete(keys: Iterable[str]) -> None:
    """S3オブジェクトを一括削除する。"""
    s3 = _client("s3")
    objects = [{"Key": key} for key in keys]
    if objects:
        s3.delete_objects(Bucket=settings.DOCUMENT_S3_BUCKET, Delete={"Objects": objects})


def empty_document_bucket() -> int:
    """ドキュメントバケット内の全オブジェクトを削除する。"""
    s3 = _client("s3")
    print(f"[cleanup] S3 ドキュメントバケット内のオブジェクトを削除します: {settings.DOCUMENT_S3_BUCKET}")

    deleted = 0
    try:
        paginator = s3.get_paginator("list_objects_v2")
        for page in paginator.paginate(Bucket=settings.DOCUMENT_S3_BUCKET):
            keys = [obj["Key"] for obj in page.get("Contents", [])]
            if not keys:
                continue
            _chunk_delete(keys)
            deleted += len(keys)
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        if code == "NoSuchBucket":
            print("[cleanup] WARN  S3 ドキュメントバケットは既に存在しませんでした")
            return 0
        print(f"[cleanup] ERROR S3 ドキュメントバケット内の削除に失敗しました: {code}")
        print(f"[cleanup] ERROR {error!s}")
        return deleted

    if deleted == 0:
        print("[cleanup] 削除対象のオブジェクトはありませんでした")
    else:
        print(f"[cleanup] {deleted} 件のオブジェクトを削除しました")
    return deleted


def delete_document_bucket() -> bool | None:
    """ドキュメントバケットを削除する。"""
    s3 = _client("s3")
    print(f"[cleanup] S3 ドキュメントバケット削除を開始します: {settings.DOCUMENT_S3_BUCKET}")
    try:
        s3.delete_bucket(Bucket=settings.DOCUMENT_S3_BUCKET)
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        if code == "NoSuchBucket":
            print("[cleanup] WARN  S3 ドキュメントバケットは既に存在しませんでした")
            return None
        if code == "BucketNotEmpty":
            print("[cleanup] WARN  S3 ドキュメントバケットにオブジェクトが残っているため削除できませんでした")
            return False
        print(f"[cleanup] ERROR S3 ドキュメントバケット削除に失敗しました: {code}")
        print(f"[cleanup] ERROR {error!s}")
        return False
    else:
        print("[cleanup] S3 ドキュメントバケット削除完了")
        return True


def delete_iam_role() -> bool | None:
    """Bedrock Knowledge Base用のIAMロールを削除する。"""
    iam = _client("iam")
    role_name = settings.BEDROCK_ROLE_NAME
    print(f"[cleanup] IAMロール削除を開始します (role={role_name})")

    try:
        # まず、インラインポリシーを全て削除する
        try:
            policy_response = iam.list_role_policies(RoleName=role_name)
            for policy_name in policy_response.get("PolicyNames", []):
                iam.delete_role_policy(RoleName=role_name, PolicyName=policy_name)
                print(f"[cleanup] インラインポリシー削除: {policy_name}")
        except iam.exceptions.NoSuchEntityException:
            print("[cleanup] WARN  IAMロールは既に削除済みでした")
            return None

        # 次に、アタッチされたマネージドポリシーを全てデタッチする
        attached_policies = iam.list_attached_role_policies(RoleName=role_name)
        for policy in attached_policies.get("AttachedPolicies", []):
            iam.detach_role_policy(RoleName=role_name, PolicyArn=policy["PolicyArn"])
            print(f"[cleanup] マネージドポリシーをデタッチ: {policy['PolicyName']}")

        # 最後にロールを削除する
        iam.delete_role(RoleName=role_name)
    except iam.exceptions.NoSuchEntityException:
        print("[cleanup] WARN  IAMロールは既に削除済みでした")
        return None
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code")
        print(f"[cleanup] ERROR IAMロール削除に失敗しました: {code}")
        print(f"[cleanup] ERROR {error!s}")
        return False
    else:
        print("[cleanup] IAMロール削除完了")
        return True


def cleanup_all() -> CleanupSummary:
    """全てのリソースをクリーンアップする。"""
    print("[cleanup] クリーンアップを開始します (全リソース削除モード / ドキュメント含む)")

    summary = CleanupSummary()

    knowledge_base_id: str | None
    try:
        knowledge_base_id = resolve_knowledge_base_id()
    except ValueError as exc:
        print(f"[cleanup] ERROR {exc!s}")
        knowledge_base_id = None
        summary.knowledge_base_deleted = False

    if knowledge_base_id:
        data_source_info: DataSourceInfo | None
        try:
            data_source_info = resolve_data_source(knowledge_base_id)
        except ValueError as exc:
            print(f"[cleanup] ERROR {exc!s}")
            data_source_info = None
            summary.data_source_deleted = False

        if data_source_info:
            summary.data_source_deleted = delete_data_source(data_source_info)
        else:
            print("[cleanup] Data Source: 設定された ID は既に削除済みとみなします")

        summary.knowledge_base_deleted = delete_knowledge_base(knowledge_base_id)
    else:
        print("[cleanup] Knowledge Base: 設定された ID は既に削除済みとみなします")

    summary.vector_index_deleted = delete_vector_index()
    summary.vector_bucket_deleted = delete_vector_bucket()
    summary.documents_deleted = empty_document_bucket()
    summary.document_bucket_deleted = delete_document_bucket()
    summary.iam_role_deleted = delete_iam_role()

    label_map = {
        "knowledge_base_deleted": "Knowledge Base",
        "data_source_deleted": "Data Source",
        "vector_index_deleted": "S3 Vectors インデックス",
        "vector_bucket_deleted": "S3 Vectors バケット",
        "document_bucket_deleted": "S3 ドキュメントバケット",
        "iam_role_deleted": "IAMロール",
    }

    for key, label in label_map.items():
        value = getattr(summary, key)
        if value is True:
            print(f"[cleanup] {label}: 削除済み")
        elif value is False:
            print(f"[cleanup] WARN  {label}: 削除に失敗しました")
        else:
            print(f"[cleanup] {label}: スキップまたは不要でした")

    print(f"[cleanup] 削除したドキュメント数: {summary.documents_deleted}")

    print("[cleanup] クリーンアップ完了")
    return summary


def main(argv: list[str]) -> int:  # noqa: ARG001
    """クリーンアップのメインエントリーポイント。"""
    summary = cleanup_all()

    failures = summary.failures()
    if failures:
        print("[cleanup] WARN  一部のリソースで削除が失敗しました: " + ", ".join(sorted(failures)))
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1:]))
```

### 5.3. 実行

それではクリーンアップスクリプトを実行します。

#### 実行例

```bash
❯ uv run -m s3_vectors_rag_hands_on.cleanup
[cleanup] クリーンアップを開始します (全リソース削除モード / ドキュメント含む)
[cleanup] 設定値の Knowledge Base ID を利用します: <KNOWLEDGE_BASE_ID>
[cleanup] Data Source を特定しました: <DATA_SOURCE_ID> (name=s3-sample-documents)
[cleanup] データソース削除を開始します (knowledge_base_id=<KNOWLEDGE_BASE_ID>, data_source_id=<DATA_SOURCE_ID>)
[cleanup] dataDeletionPolicy を RETAIN に更新しました
[cleanup] データソース削除完了
[cleanup] Knowledge Base 削除を開始します (id=<KNOWLEDGE_BASE_ID>)
[cleanup] Knowledge Base 削除完了
[cleanup] S3 Vectors インデックス削除を開始します (bucket=s3-vectors-rag-hands-on-vectors-<ACCOUNT_ID>, index=s3-vectors-rag-hands-on-index-<ACCOUNT_ID>)
[cleanup] S3 Vectors インデックス削除完了
[cleanup] S3 Vectors バケット削除を開始します (name=s3-vectors-rag-hands-on-vectors-<ACCOUNT_ID>)
[cleanup] S3 Vectors バケット削除完了
[cleanup] S3 ドキュメントバケット内のオブジェクトを削除します: s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>
[cleanup] 10 件のオブジェクトを削除しました
[cleanup] S3 ドキュメントバケット削除を開始します: s3-vectors-rag-hands-on-documents-<ACCOUNT_ID>
[cleanup] S3 ドキュメントバケット削除完了
[cleanup] IAMロール削除を開始します (role=BedrockKnowledgeBaseRole)
[cleanup] インラインポリシー削除: BedrockModelAccess
[cleanup] インラインポリシー削除: S3Access
[cleanup] インラインポリシー削除: S3VectorsAccess
[cleanup] IAMロール削除完了
[cleanup] Knowledge Base: 削除済み
[cleanup] Data Source: 削除済み
[cleanup] S3 Vectors インデックス: 削除済み
[cleanup] S3 Vectors バケット: 削除済み
[cleanup] S3 ドキュメントバケット: 削除済み
[cleanup] IAMロール: 削除済み
[cleanup] 削除したドキュメント数: 10
[cleanup] クリーンアップ完了
```

実行完了しリソースがすべて削除されました。
マネージドコンソールも合わせて確認しリソースが削除されているか確認すると良いと思います。

## まとめ

本記事では、Python SDK（boto3）を使ってAmazon S3 VectorsとKnowledge Baseを組み合わせたRAGシステムを実装しました。

質問や気になる点があれば、お気軽にコメントでお知らせください！
今後もPython, TypeScript, AWS関連の記事を中心に投稿していく予定ですので、フォローいただけると嬉しいです！

https://github.com/ShoFukada/s3_vectors_rag_hands_on
