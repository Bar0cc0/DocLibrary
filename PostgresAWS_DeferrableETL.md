
# Deferrable ETL with Airflow, S3, and dbt

### Orchestration Flow
| Step | Description |
|------|-------------|
|1. **Sensor**	|Airflow Deferrable HTTP Sensor checks if the API’s data is available |
|2. **Fetch**	|Run jobs to fetch data from the API and uploads it to S3 |
|3. **Validate**	|Validates the schema with dbt |
|4. **Partition Creation**	|Dynamically creates a partition for the current execution date |
|5. **Load**		|Upserts into the partitioned tables |

---
### Features
| Feature | Description |
|---------|-------------|
| **Deferrable Sensor** | Asynchronous execution allowing workers to yield resources while waiting for external data availability, preventing worker pool exhaustion during API polling |
| **Dynamic partition creation** | Automatically generates PostgreSQL table partitions for each execution date, optimizing query performance and enabling efficient data lifecycle management |
| **Incremental dbt model** | Implements efficient change data capture with upsert capabilities to handle both new records and updates to existing data, reducing processing time and resource usage |
| **Retry logic, SLA miss callback, max_active_runs=1** | Comprehensive resilience framework ensuring reliable execution through automated retry policies, SLA violation notifications, and sequential run enforcement to prevent concurrency issues |
| **Failure alerts** | Multi-channel notification system delivering formatted error reports via email and Slack with contextual information including task details, timestamps, and direct log links |
| **Secret management** | Secure credential storage and access using Airflow's native Variables and Connections, enabling safe integration with external systems without hardcoding sensitive information |
| **Lineage & Docs** | Automated documentation generation and data lineage tracking through dbt docs integration, providing visibility into data transformations and dependencies |
| **Monitoring** | End-to-end observability through Airflow's integrated monitoring tools, including visual pipeline state tracking, detailed execution logs, and customizable SLA definitions |

---


### File: dags/deferrable_etl_dag.py
```python
from airflow import DAG
from airflow.utils.dates import days_ago
from airflow.providers.http.sensors.http import HttpSensorAsync
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator
from airflow.operators.slack import SlackAPIPostOperator
from airflow.models import Variable
from datetime import timedelta, datetime
import logging

# Default DAG arguments
default_args = {
    'owner': 'data-engineer',
    'depends_on_past': False,
    'email': ['data-ops@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=10),
    'on_failure_callback': lambda context: SlackAPIPostOperator(
        task_id='notify_failure',
        channel='#alerts',
        text=f":red_circle: DAG *{context['dag'].dag_id'}* failed on task *{context['task'].task_id}* at {datetime.utcnow().isoformat()} UTC",
        token=Variable.get('SLACK_TOKEN')
    ).execute(context)
}

dag = DAG(
    dag_id='deferrable_etl_partition_dbt',
    default_args=default_args,
    description='Deferrable sensor ETL -> S3 -> dbt incremental -> dynamic partitions -> docs',
    schedule_interval='@hourly',
    start_date=days_ago(1),
    catchup=False,
    max_active_runs=1,
    sla_miss_callback=lambda context: SlackAPIPostOperator(
        task_id='sla_missed',
        channel='#alerts',
        text=f":warning: SLA missed for *{context['task_instance'].task_id}* in DAG *{context['dag'].dag_id}*",
        token=Variable.get('SLACK_TOKEN')
    ).execute(context)
)

start = EmptyOperator(task_id='start', dag=dag)

# 1. Deferrable HTTP Sensor
sensor_check = HttpSensorAsync(
    task_id='sensor_check_api',
    http_conn_id='source_api',
    endpoint='data/{{ ds }}.csv',
    method='HEAD',
    poke_interval=60,
    timeout=3600,
    soft_fail=False,
    dag=dag,
)

# 2. Fetch & upload to S3
def fetch_and_upload_to_s3(ds, **kwargs):
    import requests
    date = ds
    url = f"https://api.example.com/data/{date}.csv"
    resp = requests.get(url, timeout=30)
    resp.raise_for_status()
    s3 = S3Hook(aws_conn_id='aws_default')
    key = f"raw/daily/{date}.csv"
    s3.load_string(resp.text, key, bucket_name='my-etl-bucket', replace=True)

fetch_task = PythonOperator(
    task_id='fetch_and_upload_to_s3',
    python_callable=fetch_and_upload_to_s3,
    provide_context=True,
    dag=dag,
)

# 3. Validate & load staging
def validate_and_load_staging(ds, **kwargs):
    import pandas as pd
    from io import StringIO
    date = ds
    s3 = S3Hook(aws_conn_id='aws_default')
    data = s3.read_key(f"raw/daily/{date}.csv", bucket_name='my-etl-bucket')
    df = pd.read_csv(StringIO(data))
    required = {'order_id','customer_id','amount','updated_at'}
    if not required.issubset(df.columns):
        raise ValueError(f"Missing cols: {required - set(df.columns)}")
    pg = PostgresHook(postgres_conn_id='postgres_default')
    engine = pg.get_sqlalchemy_engine()
    df.to_sql('staging_orders', engine, if_exists='replace', index=False)

validate_task = PythonOperator(
    task_id='validate_and_load_staging',
    python_callable=validate_and_load_staging,
    provide_context=True,
    dag=dag,
)

# 4. Dynamic Partition Creation
def create_partition(ds, **kwargs):
    pg = PostgresHook(postgres_conn_id='postgres_default')
    conn = pg.get_conn()
    cursor = conn.cursor()
    date = ds
    next_day = (kwargs['execution_date'] + timedelta(days=1)).strftime('%Y-%m-%d')
    partition = date.replace('-', '_')
    create_sql = f"""
    CREATE TABLE IF NOT EXISTS public.orders_{partition} 
      PARTITION OF public.orders 
      FOR VALUES FROM ('{date}') TO ('{next_day}');
    """
    cursor.execute(create_sql)
    conn.commit()

partition_task = PythonOperator(
    task_id='create_partition',
    python_callable=create_partition,
    provide_context=True,
    dag=dag,
)

# 5. Run dbt models
dbt_run = BashOperator(
    task_id='run_dbt',
    bash_command=(
        'cd /opt/airflow/dbt_project && '
        'dbt deps && '
        'dbt seed --profiles-dir . && '
        'dbt run --models staging_orders orders_incremental'
    ),
    dag=dag,
)

# 6. Generate & publish dbt docs
dbt_docs = BashOperator(
    task_id='generate_dbt_docs',
    bash_command=(
        'cd /opt/airflow/dbt_project && '
        'dbt docs generate && '
        'dbt docs serve --port 8001 &'
    ),
    dag=dag,
)

end = EmptyOperator(task_id='end', dag=dag)

# Task dependencies
start >> sensor_check >> fetch_task >> validate_task >> partition_task >> dbt_run >> dbt_docs >> end
```  

---
### PostgreSQL Partitioning (dynamic)
```sql
CREATE TABLE public.orders (
  order_id   BIGINT PRIMARY KEY,
  customer_id INT NOT NULL,
  amount     NUMERIC,
  updated_at TIMESTAMP NOT NULL,
  load_date  DATE NOT NULL
) PARTITION BY RANGE (load_date);
```
Partitions auto-created by the DAG’s `create_partition` task.

---
### dbt Configuration Files

#### dbt_project.yml
```yaml
name: 'etl_project'
version: '1.0'
config-version: 2
profile: 'etl_profile'

model-paths: ['models']
seed-paths: ['seeds']

models:
  etl_project:
    staging_orders:
      materialized: table
    orders_incremental:
      materialized: incremental
      unique_key: order_id
```

#### profiles.yml
```yaml
etl_profile:
  target: dev
  outputs:
    dev:
      type: postgres
      threads: 4
      host: '{{ env_var('DB_HOST') }}'
      port: 5432
      user: '{{ env_var('DB_USER') }}'
      pass: '{{ env_var('DB_PASS') }}'
      dbname: '{{ env_var('DB_NAME') }}'
      schema: public
```  

---
### Alert Templates

#### Slack Message (on failure)
```json
{
  "attachments": [
    {
      "fallback": "DAG failure alert",
      "color": "danger",
      "title": "Airflow DAG Failure: deferrable_etl_partition_dbt",
      "fields": [
        {"title": "Task", "value": "{{ task_instance.task_id }}", "short": true},
        {"title": "Execution Time", "value": "{{ ts }}", "short": true},
        {"title": "Log", "value": "{{ task_instance.log_url }}"}
      ]
    }
  ]
}
```

#### Email Template (on failure)
```html
<h3>Airflow DAG Failure: deferrable_etl_partition_dbt</h3>
<p><strong>Task:</strong> {{ task_instance.task_id }}</p>
<p><strong>Execution Time:</strong> {{ ts }}</p>
<p>See logs: <a href="{{ task_instance.log_url }}">link</a></p>
```

---
### Terraform Modules: AWS Infrastructure

#### Module: S3 Bucket (modules/s3/main.tf)
```hcl
resource "aws_s3_bucket" "etl_raw" {
  bucket = var.bucket_name
  acl    = "private"

  versioning {
    enabled = true
  }

  lifecycle_rule {
    id      = "expire-raw"
    enabled = true
    expiration {
      days = 30
    }
  }
}
```

```hcl
# modules/s3/variables.tf
variable "bucket_name" {
  type = string
}
```

#### Module: EC2 PostgreSQL (modules/ec2_db/main.tf)
```hcl
resource "aws_instance" "postgres" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  key_name                    = var.key_pair
  subnet_id                   = var.subnet_id
  vpc_security_group_ids      = [var.db_security_group]
  associate_public_ip_address = false

  user_data = templatefile("${path.module}/init-postgres.sh.tpl", {
    db_name     = var.db_name
    db_user     = var.db_user
    db_password = var.db_password
  })

  tags = {
    Name = "etl-postgres-db"
  }
}
```

```hcl
# modules/ec2_db/variables.tf
variable "ami_id"             { type = string }
variable "instance_type"      { type = string }
variable "key_pair"           { type = string }
variable "subnet_id"          { type = string }
variable "db_security_group"  { type = string }
variable "db_name"            { type = string }
variable "db_user"            { type = string }
variable "db_password"        { type = string
  sensitive = true }
```

```bash
def contents of modules/ec2_db/init-postgres.sh.tpl
#!/bin/bash
# install PostgreSQL and configure database
apt-get update && apt-get install -y postgresql postgresql-contrib
sudo -u postgres psql -c "CREATE USER {{db_user}} WITH PASSWORD '{{db_password}}';"
sudo -u postgres psql -c "CREATE DATABASE {{db_name}} OWNER {{db_user}};"
```

#### Module: Alerting (modules/alerting/main.tf)
```hcl
resource "aws_sns_topic" "alerts" {
  name = "etl-alerts"
}

resource "aws_sns_topic_subscription" "ops_email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.ops_email
}

# existing Lambda role and function assumed external
resource "aws_sns_topic_subscription" "lambda_route" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "lambda"
  endpoint  = var.lambda_arn
}
```
```hcl
# modules/alerting/variables.tf
variable "ops_email" { type = string }
variable "lambda_arn" { type = string }
```

#### Module: Metrics (modules/metrics/main.tf)
```hcl
# Prometheus-Grafana deployed separately; define Grafana datasource
resource "grafana_data_source" "prometheus" {
  name = "Prometheus"
  type = "prometheus"
  url  = var.prometheus_url
}
```
```hcl
# modules/metrics/variables.tf
variable "prometheus_url" { type = string }
```

---
### Root Terraform Configuration (main.tf)
```hcl
provider "aws" {
  region = var.aws_region
}

module "s3_bucket" {
  source      = "./modules/s3"
  bucket_name = var.s3_bucket_name
}

module "ec2_db" {
  source            = "./modules/ec2_db"
  ami_id            = var.ec2_ami_id
  instance_type     = var.ec2_instance_type
  key_pair          = var.ec2_key_pair
  subnet_id         = var.ec2_subnet_id
  db_security_group = var.db_security_group_id
  db_name           = var.db_name
  db_user           = var.db_user
  db_password       = var.db_password
}

module "alerting" {
  source     = "./modules/alerting"
  ops_email  = var.ops_email
  lambda_arn = var.alert_lambda_arn
}

module "metrics" {
  source         = "./modules/metrics"
  prometheus_url = var.prometheus_url
}
```

```hcl
# variables.tf
variable "aws_region"           { type = string }
variable "s3_bucket_name"       { type = string }
variable "ec2_ami_id"           { type = string }
variable "ec2_instance_type"    { type = string }
variable "ec2_key_pair"         { type = string }
variable "ec2_subnet_id"        { type = string }
variable "db_security_group_id" { type = string }
variable "db_name"              { type = string }
variable "db_user"              { type = string }
variable "db_password"          { type = string, sensitive = true }
variable "ops_email"            { type = string }
variable "alert_lambda_arn"     { type = string }
variable "prometheus_url"       { type = string }
```

```hcl
# outputs.tf
output "db_instance_public_ip" {
  value = module.ec2_db.aws_instance.postgres.public_ip
}
```

---
### Terraform Parameter Validation
Use Terraform's built-in `terraform validate` and `pre-commit` hooks, plus a simple OPA (Open Policy Agent) policy for additional checks.

#### 1. Terraform Validation Script (scripts/validate.sh)
```bash
echo "## Running terraform fmt"
terraform fmt -check

echo "## Running terraform validate"
terraform init -backend=false
terraform validate -json > validate.json
if ! jq -e '.diagnostics | length == 0' validate.json; then
  echo "Validation errors:" && jq .validate.json
  exit 1
fi

echo "## Running OPA policy checks"
opa test -v policy/
```

#### 2. OPA Policy Example (policy/terraform.rego)
```rego
package terraform.validation

# Ensure all modules require version constraints
deny[msg] {
  input.module_versions[_] == null
  msg = "Module without version constraint detected"
}
```

---
### Terratest in Python
Use `pytest` + `localexec` to verify module provisioning and outputs.

#### Directory: tests/
```
tests/
└── test_terraform_infra.py
```

#### tests/test_terraform_infra.py
```python
import os
import subprocess
import pytest
from python_terraform import Terraform

@pytest.fixture(scope='module')
def tf():
    tf = Terraform(working_dir=os.path.abspath("../"))
    return tf

def test_terraform_init(tf):
    ret, stdout, stderr = tf.init(capture_output=False)
    assert ret == 0

def test_terraform_plan(tf):
    ret, plan, _ = tf.plan(no_color=IsFlagged, capture_output=True)
    assert ret == 0
    assert "Plan: 0 to add, 0 to change, 0 to destroy" in plan

def test_outputs_exist(tf):
    ret, output, _ = tf.output(json=True)
    assert ret == 0
    data = output
    assert "db_instance_public_ip" in data
```

---
### CI/CD Pipeline
#### 1. GitHub Actions Workflow (.github/workflows/terraform.yml)
```yaml
name: "Terraform CI"
on:
  pull_request:
    paths:
      - 'modules/**'
      - 'main.tf'
      - 'variables.tf'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      - name: Install dependencies
        run: |
          pip install python-terraform pytest opa
      - name: Terraform Format & Validate
        run: bash scripts/validate.sh
      - name: Run Terratest
        run: pytest tests/test_terraform_infra.py

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    environment:
      name: 'production'
      url: ${{ steps.deploy.outputs.endpoint }}
    steps:
      - uses: actions/checkout@v3
      - name: Configure Terraform Cloud
        run: |
          terraform login
          terraform init
      - name: Terraform Apply
        run: |
          terraform apply -auto-approve
        env:
          TF_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

#### 2. Terraform Cloud Configuration (versions.tf & .terraformrc)
```hcl
# versions.tf
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "deferrable-etl-infra"
    }
  }
  required_version = ">= 1.2.0"
}
```
```hcl
# .terraformrc (in user home)
credentials "app.terraform.io" {
  token = var.terraform_cloud_token
}
```