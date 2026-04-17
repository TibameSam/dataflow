# dataflow

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

    black -l 80 src/

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

## deploy-airflow:
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

## 調整筆電 gcloud project
    gcloud config set project airflow-466005

## 上傳程式碼到 Composer
	gcloud composer \
	environments storage \
	dags import --environment airflow  \
	--location us-central1 \
	--source "src/dataflow" 

