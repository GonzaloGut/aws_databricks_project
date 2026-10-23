# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

An Apache Airflow 3.2 (Docker Compose, CeleryExecutor) project that orchestrates a medallion-style pipeline on AWS + Databricks. There is no test suite, linter, or README.

## Commands

```bash
docker compose up airflow-init        # first run: DB migration + admin user (airflow/airflow)
docker compose up -d --build          # build custom image (Dockerfile + requirements.txt) and start
docker compose down                   # stop
docker compose run --rm airflow-cli dags list
docker compose run --rm airflow-cli dags test aws_project   # run the whole DAG once, locally
docker compose run --rm airflow-cli tasks test aws_project extract_load_s3 2026-01-01   # single task
```

UI / API server: http://localhost:8080. Flower is behind a profile: `docker compose --profile flower up -d`.

After changing `requirements.txt` the image must be rebuilt (`--build`); `_PIP_ADDITIONAL_REQUIREMENTS` is not used.

## Architecture

- [dags/aws_project.py](dags/aws_project.py) defines the single `aws_project` DAG (TaskFlow API via `airflow.sdk`, `@daily`, paused on creation). Task flow:
  `extract_load_s3 >> [transform_load_s3, transform_load_s3_parquet] >> trigger_crawler >> trigger_databricks_job`
- Business logic lives in `utils/` as one class per layer (`BronzeLayer`, `SilverLayer`, `GoldLayer`). DAG tasks instantiate these classes and call their methods; keep heavy logic out of the DAG file.
  - **Bronze** ([utils/bronze_layer.py](utils/bronze_layer.py)): downloads CSVs (bookings, passengers, airports) over HTTP and writes them to S3 at `bronze/<YYYY-MM-DD>/<file>.csv`. The DAG uses `put_data_s3_hook` (Airflow connection `aws_s3_conn`); `put_data_s3` is a boto3 alternative using env credentials.
  - **Silver** ([utils/silver_layer.py](utils/silver_layer.py)): starts AWS Glue jobs (`silver_layer`, `silver_layer_athena`) passing `--load_date`, then the Glue crawler `crawler_silver`. Methods poll every 30s and block until success, raising on FAILED/STOPPED. The Glue Spark scripts themselves are not in this repo.
  - **Gold** ([utils/gold_layer.py](utils/gold_layer.py)): triggers a Databricks job with `databricks-sdk` (`WorkspaceClient` using `Databricks_HOST` / `Databricks_PAT`) via `jobs.run_now`. It returns right after triggering and does not wait for the run to finish.
- The bronze task returns the date folder name; silver tasks pull it via XCom (`ti.xcom_pull(task_ids='extract_load_s3')`) as the Glue `--load_date`.
- `utils/` is mounted into containers at `/opt/airflow/utils`; the DAG makes it importable with `sys.path.append(<dags>/..)`. New utility modules must be placed in `utils/` to be visible.
- Each `utils` module has an `if __name__ == "__main__":` block used for ad-hoc manual testing outside Airflow.

## Configuration

- `.env` (gitignored, loaded by Compose `env_file` and by `load_dotenv()` in utils) holds `AIRFLOW_UID`, `aws_access_key_id`, `aws_secret_access_key` (lowercase names), `Databricks_PAT`, `Databricks_HOST`.
- AWS region is hardcoded to `ca-central-1`; the S3 bucket name (`aws-s3-bucket-airflow`) and Databricks job ID are hardcoded in the DAG.
- `config/airflow.cfg` is mounted to `/opt/airflow/config`; most settings are overridden by `AIRFLOW__*` env vars in `docker-compose.yaml`.
- `gonzalo-vs_accessKeys.csv` in the repo root holds AWS credentials and is gitignored — never commit or read it into output.
