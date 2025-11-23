# 🚀 ローカル Airflow（Docker Compose）ハンズオン

※ このまま README に貼れる形式

---

# タスク分割（1タスク = 1PR）

| Task | 内容 | ブランチ名 | コミットメッセージ |
|------|------|------------|-------------------|
| 1 | リポジトリ構成の作成（docker-compose.yaml, requirements.txt） | `feature/setup-docker-compose` | `feat: add docker-compose and requirements for local Airflow` |
| 2 | 最初のDAG作成（hello_composer.py） | `feature/add-hello-dag` | `feat: add hello_composer DAG for basic test` |
| 3 | GCS連携DAG作成（fetch_to_gcs.py） | `feature/add-gcs-dag` | `feat: add fetch_to_gcs DAG for GCS integration` |
| 4 | Composerデプロイ手順のドキュメント追加 | `feature/add-composer-deploy-docs` | `docs: add Composer deployment instructions` |

---

# 1. ゴール

* ローカルで Airflow を Docker で立ち上げる
* Composer と同じ構成（Python 3・Airflow 2.x）で DAG を作る
* ローカルで動作確認 → GCS にアップして Composer で実行できる形にする

---

# 2. 必要なもの

* Docker / Docker Compose
* Python 3.9+（オプション）
* gcloud CLI（Composer デプロイ用）

---

# 3. リポジトリ構成（Composer 互換）

```
airflow-handson/
│
├── dags/               # ← Composer へアップロードするのもここ
│   ├── hello_composer.py
│   └── fetch_to_gcs.py
│
├── docker-compose.yaml
├── requirements.txt     # ← 使う Python ライブラリ (Composer と共通)
└── README.md
```

Composer もローカル Airflow も
**DAG は Python ファイルで構成されるためほぼ同じ**。

---

# 4. Docker Compose で Airflow を立ち上げる

## (1) `docker-compose.yaml`

```yaml
x-airflow-common:
  &airflow-common
  image: apache/airflow:2.8.1
  environment:
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth'
  volumes:
    - ./dags:/opt/airflow/dags
    - ./requirements.txt:/requirements.txt
  command: >
    bash -c "
    pip install --no-cache-dir -r /requirements.txt &&
    airflow db init &&
    airflow users create --username airflow --password airflow \
      --firstname Admin --lastname User --role Admin \
      --email admin@example.com || true &&
    airflow webserver &
    airflow scheduler
    "

services:
  airflow:
    <<: *airflow-common
    ports:
      - "8080:8080"
```

---

# 5. requirements.txt

※ Composer へアップロードしても使える構成

```
apache-airflow-providers-google
google-cloud-storage
requests
```

※ 必要ならここに追加（Composer 側にもそのまま反映される）

---

# 6. ローカル Airflow 起動

```bash
docker compose up -d
```

Airflow UI
👉 [http://localhost:8080](http://localhost:8080)
ユーザー名・パスワードはデフォルトで

* user: `airflow`
* pass: `airflow`

---

# 7. 最初の DAG を作成（Composer 対応版）

`dags/hello_composer.py`

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def hello_world():
    print("Hello! This DAG works in both local Airflow and Cloud Composer!")

default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "start_date": datetime(2024, 1, 1),
    "retries": 1,
    "retry_delay": timedelta(minutes=3),
}

with DAG(
    dag_id="hello_composer",
    schedule_interval="@daily",
    default_args=default_args,
    catchup=False,
) as dag:

    hello_task = PythonOperator(
        task_id="hello_task",
        python_callable=hello_world,
    )
```

Airflow UI で DAG をオン → Trigger で実行確認。

---

# 8. GCS に保存する DAG（こちらも Composer 対応）

`dags/fetch_to_gcs.py`

```python
import os
import json
import requests
from datetime import datetime, timezone
from airflow import DAG
from airflow.operators.python import PythonOperator
from google.cloud import storage


def fetch_data():
    url = "https://api.coindesk.com/v1/bpi/currentprice.json"
    res = requests.get(url)
    res.raise_for_status()
    return res.json()


def upload_to_gcs(**context):
    # XComから前タスクの結果を取得
    data = context["ti"].xcom_pull(task_ids="fetch")

    client = storage.Client()
    # 環境変数から取得（ローカル/Composer共通で使える）
    bucket_name = os.environ.get("GCS_BUCKET", "YOUR_BUCKET")
    bucket = client.bucket(bucket_name)

    file_name = f"airflow-output/price-{datetime.now(timezone.utc).isoformat()}.json"
    blob = bucket.blob(file_name)
    blob.upload_from_string(json.dumps(data), content_type="application/json")

    print("Uploaded:", file_name)


default_args = {"owner": "airflow", "start_date": datetime(2024, 1, 1)}

with DAG(
    "fetch_and_upload",
    default_args=default_args,
    schedule_interval="@daily",
    catchup=False,
) as dag:

    fetch = PythonOperator(
        task_id="fetch",
        python_callable=fetch_data,
    )

    upload = PythonOperator(
        task_id="upload",
        python_callable=upload_to_gcs,
    )

    fetch >> upload
```

---

# 9. ローカルで動作確認後、Composer にアップロード

## (1) Composer の DAG GCS パスを取得

```bash
gcloud composer environments describe <env-name> \
  --location <region> \
  --format="value(config.dagGcsPrefix)"
```

例:
`gs://asia-northeast1-my-composer-bucket/dags`

## (2) アップロード

```bash
gsutil cp dags/*.py gs://asia-northeast1-my-composer-bucket/dags/
```

## (3) Composer → Airflow UI で DAG を確認し実行

---

# 10. Composer 対応で重要なポイント

| ポイント    | ローカル Airflow                                | Composer                        |
| ------- | ------------------------------------------- | ------------------------------- |
| ライブラリ追加 | requirements.txt                            | Composer UI の PyPI packages に登録 |
| 認証      | ADC (gcloud auth application-default login) | 自動（Service Account）             |
| ストレージ   | ローカル FS                                     | GCS                             |

※ つまり、DAG の Python コードはほぼ共通で OK
