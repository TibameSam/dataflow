# dataflow

這是一個「資料工作流排程系統」的教學專案，使用 **Apache Airflow** 來統一管理、排程、監控 [crawler](../crawler) 的爬蟲任務。學完這個專案，你會了解業界怎麼把多個爬蟲、ETL、上傳任務串成一個有依賴關係、可重跑、可監控的 pipeline。

## 這個專案在做什麼？

如果說 `crawler` 是「工人」，那 `dataflow` 就是「工廠的調度中心」：

```
Airflow Scheduler  →  讀取 DAG (用 Python 寫的工作流)  →  按時間觸發 Task
                                                              ↓
                                                  DockerOperator / KubernetesPodOperator
                                                              ↓
                                                    啟動 crawler 容器執行爬蟲
                                                              ↓
                                                  資料寫進 MySQL / BigQuery
```

- **DAG（Directed Acyclic Graph）**：用 Python 描述「哪些任務、誰先誰後、什麼時候跑」
- **Airflow Scheduler**：時間到了就把 task 丟給 worker 執行
- **Airflow Worker**：實際執行任務的工人（透過 Celery + Redis 派工）
- **Webserver**：Airflow 的 Web UI（port 5000），可以看 DAG、看 log、手動觸發
- **DockerOperator / KubernetesPodOperator**：讓 Airflow 直接啟動 crawler 的 Docker 容器或 K8s Pod 執行任務

## 為什麼要用 Airflow？

`crawler` 專案已經有 `scheduler.py`（用 APScheduler）可以定時派任務，為什麼還要 Airflow？

當任務複雜起來，你會遇到這些需求：
- **任務之間有依賴**：要先抓股價、再算技術指標、最後寄報表，APScheduler 沒辦法表達
- **任務失敗要能重跑單一節點**：APScheduler 失敗整個 job 都重來，Airflow 可以只 retry 失敗的 task
- **要看歷史執行狀況**：什麼時候跑過、跑了多久、log 在哪，APScheduler 都不記錄
- **要動態切換環境**：本地、GCP Composer、自架 Swarm 之間切換，需要抽象層

Airflow 用 **DAG（有向無環圖）** 描述任務之間的關係，配上 Web UI、metadata DB、retry 機制，是業界資料工程的標配。

## 使用的技術

| 技術 | 用途 | 為什麼用它 |
| --- | --- | --- |
| Python 3.11 | 主要開發語言 | Airflow 本身就是 Python 寫的 |
| [uv](https://docs.astral.sh/uv/) | 套件管理 | 比 pip 快很多 |
| [Apache Airflow](https://airflow.apache.org/) | 工作流排程 | 業界資料工程標配 |
| Celery + Redis | Airflow worker 派工機制 | 讓任務能分散到多台 worker |
| DockerOperator | 從 Airflow 啟動 crawler 容器 | 讓 DAG 跟 crawler 解耦 |
| KubernetesPodOperator | 從 Airflow 啟動 K8s Pod | 用於 GCP Cloud Composer 環境 |
| Docker Swarm | 多機部署 Airflow stack | 比裸 docker compose 多了高可用 |
| Google Cloud Composer | 託管版 Airflow | 不用自己維運 Airflow 伺服器 |
| MySQL | Airflow metadata DB | 儲存 DAG 狀態、執行歷史 |

## 資料夾結構速覽

```
dataflow/
├── dataflow/
│   ├── config.py                # 環境變數集中管理（DB、RabbitMQ 帳密）
│   ├── constant.py              # DAG 的預設參數（owner、retries、timeout）
│   ├── dags/                    # Airflow 讀取的 DAG 定義（被 Airflow 自動掃描）
│   │   ├── hello_world.py       # 入門範例
│   │   ├── bash_operator.py     # 用 BashOperator 跑 shell command
│   │   ├── python_operator.py   # 用 PythonOperator 跑 Python function
│   │   ├── branch_python_operator.py # 條件分支
│   │   ├── dummy_operator.py    # 空任務（常用於分支匯流）
│   │   ├── docker_operator.py   # 用 DockerOperator 跑容器
│   │   ├── producer.py          # 派送 crawler 任務 (Docker)
│   │   ├── producer_twse.py     # 派送 twse queue 任務
│   │   ├── producer_scheduler.py # 排程版 producer
│   │   └── producer_k8s_scheduler.py # K8s 版（給 Cloud Composer 用）
│   └── etl/                     # 任務實作（被 dags/ import 使用）
│       └── ...（與 dags/ 同名，職責分離：dag 定義 vs task 實作）
├── airflow.cfg                  # 本地/Swarm 環境的 Airflow 設定
├── airflow-gce.cfg              # GCP Composer 環境的 Airflow 設定
├── genenv.py                    # 根據 local.ini 產生 .env
├── local.ini                    # DEV/DOCKER/PRODUCTION 三種環境設定
└── docker-compose-airflow*.yml  # Airflow stack 部署設定
```

**為什麼 `dags/` 跟 `etl/` 要分開？**
`dags/` 只放「DAG 結構」（誰先誰後、何時跑），`etl/` 放「task 怎麼做」（實際邏輯）。這樣修改 DAG 拓樸不會動到 task 程式碼，反之亦然，職責更清楚。

## 學習順序建議

如果你是第一次接觸 Airflow，建議依序閱讀：

1. `dataflow/constant.py` — 認識 DAG 的預設參數（owner、retries、start_date）
2. `dataflow/dags/hello_world.py` — 最簡單的 DAG 長什麼樣
3. `dataflow/dags/bash_operator.py` / `python_operator.py` — 兩種最常用的 operator
4. `dataflow/dags/branch_python_operator.py` — 學會條件分支
5. `dataflow/dags/docker_operator.py` + `dataflow/etl/docker_operator.py` — 看 DAG 怎麼跟 task 實作分離
6. `dataflow/dags/producer_scheduler.py` — 把所有概念串起來：定時觸發 crawler 容器

## Docker Compose 檔案說明

### Airflow Stack

| 檔案 | 用途 |
| --- | --- |
| `docker-compose-airflow.yml` | 本機 / 自架 Swarm 用的 Airflow stack |
| `docker-compose-airflow-gce.yml` | 部署到 GCE 主機（GCP Compute Engine）的版本 |

`docker-compose-airflow.yml` 裡面包含：
- **initdb**：第一次啟動時初始化 Airflow metadata DB
- **create-user**：建立預設管理員帳號 `admin / admin`
- **redis**：Celery 的 broker，讓 worker 領任務
- **webserver**：Airflow Web UI（port 5000）
- **scheduler**：負責讀 DAG、依排程觸發任務
- **flower**：Celery 監控介面（port 5556）
- **worker / crawler_twse / crawler_tpex**：實際執行任務的 worker，分別監聽不同 queue

**注意**：這份 compose 用的是 **Docker Swarm** 模式（`docker stack deploy`），不是單機 `docker compose up`。差別在 Swarm 可以跨多台機器部署、有 `placement.constraints` 控制服務跑在哪個節點。

### Airflow 設定檔

| 檔案 | 用途 |
| --- | --- |
| `airflow.cfg` | 本地 / Swarm 環境的 Airflow 設定（executor、DB 連線、log 路徑等） |
| `airflow-gce.cfg` | 部署到 GCP Composer 時用的版本（DB host、log storage 不同） |

## Dockerfile 說明

| 檔案 | 用途 | 差別 |
| --- | --- | --- |
| `with.env.Dockerfile` | 本機 / Swarm 用 | base image 是 `linsamtw/tibame_dataflow:0.0.5`，使用 `airflow.cfg`，build 時跑 `ENV=DOCKER genenv.py` |
| `gce.with.env.Dockerfile` | 部署到 GCE 用（從零 build） | base image 是乾淨的 `ubuntu:22.04`，改用 `airflow-gce.cfg` |
| `gce.with.env.cache.Dockerfile` | 部署到 GCE 用（複用 cache 加速） | base image 是 `linsamtw/tibame_dataflow:0.0.7`，只複製檔案、不重裝套件，build 很快 |

**為什麼有 cache 版本？**
重新跑 `apt install` + `uv sync` 動輒 10 分鐘，但其實大部分時間套件根本沒變。`gce.with.env.cache.Dockerfile` 用「上一版 image 當 base」的技巧，只蓋掉新改的程式碼，build 通常 30 秒內完成，適合每日小改動。

**為什麼要分本地版 / GCE 版？**
本地用的 `airflow.cfg` 連的是 Swarm 內的 MySQL；GCE 版的 `airflow-gce.cfg` 連的是 Cloud SQL，DB host、log 儲存位置都不同，所以 build image 時就要選對設定檔。

## .gitignore 說明

| 項目 | 為什麼要忽略 |
| --- | --- |
| `*__pycache__/`、`*.pyc` | Python 編譯產生的暫存檔 |
| `.vscode/` | 編輯器個人設定 |
| `.env` | **最重要！** 裡面有 DB 帳密、API key |
| `.venv/` | uv 建立的虛擬環境，每台機器自己產生 |
| `airflow.db`、`logs/` | Airflow 跑起來會產生的本地 metadata、log，不該推上 git |

---

# 環境設定

#### 安裝 uv

    curl -LsSf https://astral.sh/uv/install.sh | sh

#### 安裝 Python 3.11

    uv python install 3.11

#### set uv 虛擬環境

    uv venv --python 3.11

#### 安裝 repo 套件

    uv sync

#### 建立環境變數

    ENV=DEV python genenv.py
    ENV=DOCKER python genenv.py
    ENV=PRODUCTION python genenv.py

#### 排版

    black -l 80 dataflow/

# Docker

#### build docker image

    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.1 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.1.arm64 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.2 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.2.arm64 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.3 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.3.arm64 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.4 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.4.arm64 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.5 .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.5.arm64 .
    docker build -f gce.with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.6.gce .
    docker build -f with.env.Dockerfile -t linsamtw/tibame_dataflow:0.0.7 .
    docker build -f gce.with.env.cache.Dockerfile -t linsamtw/tibame_dataflow:0.0.8 .

#### push docker image

    docker push linsamtw/tibame_dataflow:0.0.1
    docker push linsamtw/tibame_dataflow:0.0.1.arm64
    docker push linsamtw/tibame_dataflow:0.0.2
    docker push linsamtw/tibame_dataflow:0.0.2.arm64
    docker push linsamtw/tibame_dataflow:0.0.3
    docker push linsamtw/tibame_dataflow:0.0.3.arm64
    docker push linsamtw/tibame_dataflow:0.0.4
    docker push linsamtw/tibame_dataflow:0.0.4.arm64
    docker push linsamtw/tibame_dataflow:0.0.5
    docker push linsamtw/tibame_dataflow:0.0.5.arm64
    docker push linsamtw/tibame_dataflow:0.0.6.gce
    docker push linsamtw/tibame_dataflow:0.0.7
    docker push linsamtw/tibame_dataflow:0.0.8

#### pull docker image

    docker pull linsamtw/tibame_dataflow:0.0.1
    docker pull linsamtw/tibame_dataflow:0.0.2

# Airflow 部署 (Docker Swarm)

#### deploy airflow stack

    DOCKER_IMAGE_VERSION=0.0.1 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.1.arm64 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.2 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.2.arm64 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.3 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.3.arm64 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.4 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.4.arm64 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.5 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.5.arm64 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.6.gce docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow
    DOCKER_IMAGE_VERSION=0.0.7 docker stack deploy --with-registry-auth -c docker-compose-airflow.yml airflow

#### 移除 airflow stack

    docker stack rm airflow

# GCP Cloud Composer

Cloud Composer 是 GCP 提供的「託管版 Airflow」，不用自己跑 webserver / scheduler / DB，把 DAG 上傳上去就會自動執行。

#### 調整筆電 gcloud project

    gcloud config set project airflow-466005

#### 上傳程式碼到 Composer

    gcloud composer \
    environments storage \
    dags import --environment airflow  \
    --location us-central1 \
    --source "dataflow"
