# AIRFLOW & KAFKA PIPELINES

+ APACHE AIRFLOW
+ APACHE KAFKA 

## Database Connection Commands

Connect to db & copy data file to the table 'access_log' through command pipeline:
```bash
echo "\c <db_name>;\COPY access_log FROM '/home/project/transformed-data.csv' DELIMITERS ',' CSV HEADER;" | psql --username=postgres --host=localhost
```

Connect db & copy data file to the table 'users' through command pipeline:
```bash
echo "\c <db_name>;\COPY users FROM '/home/project/transformed-data.csv' DELIMITERS ',' CSV;" | psql --username=postgres --host=localhost
```

## 1. APACHE AIRFLOW

### INSTALLATION

Install Airflow. Extract the version of Python installed:
```bash
AIRFLOW_VERSION=2.6.1
PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

Create home var and add to env PATH:
```bash
export AIRFLOW_HOME=~/airflow
```

Check airflow config file for default location of dags.  
Default log path: `logs/dag_id/task_id/execution_date/try_number.log`

Init db, create user, then start all components:
```bash
airflow standalone
localhost:8080
```

### CLI Commands

```bash
airflow dags list
airflow tasks list <dag_name>
airflow dags unpause <dag_name>
airflow dags pause <dag_name>
airflow dags delete <dag_name>
```

### DAGs with Python API

```python
from datetime import timedelta
from airflow import DAG 
from airflow.operators.bash import BashOperator 
from airflow.utils.dates import days_ago

# DAG arguments
default_args = {
    'owner': 'user_name',
    'start_date': days_ago(0),
    'email': ['user@mail.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

# DAG definition/instanciation
dag = DAG(
    'dag_ID',
    default_args=default_args,
    description='docstring',
    schedule_interval=timedelta(days=1),
)

# Tasks definition 
task_1 = BashOperator(
    task_id='task_1',
    bash_command='<bash command>',
    dag=dag,
)

# Pipeline
task_1 >> task2 >> task_3 
```

### Submit DAG

```bash
cp <dag_script.py> $AIRFLOW_HOME/dags
cp <dag_script.sh> $AIRFLOW_HOME/dags
cd airflow/dags
chmod 777 <dag_script.*>
```

## 2. APACHE KAFKA

### Installation

CLI API:
```bash
wget https://archive.apache.org/dist/kafka/2.8.0/kafka_2.12-2.8.0.tgz
tar -xzf kafka_2.12-2.8.0.tgz
```

Python API:
```bash
pip install kafka-python
```

Dependencies:
- `/tmp/kakfa-logs` to store the messages
- Kafka uses `/home/project/kafka_2.12-2.8.0` to store shell scripts (bin), config files (config), logs (logs)

### Start ZooKeeper

Open new terminal and keep it running:
```bash
cd kafka_2.12-2.8.0
bin/zookeeper-server-start.sh config/zookeeper.properties
```

### Start Broker Service

Open new terminal and keep it running:
```bash
cd kafka_2.12-2.8.0
bin/kafka-server-start.sh config/server.properties
```

### CLI Commands

Create topic (open new terminal):
```bash
cd kafka_2.12-2.8.0
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic news
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic bankbranch --partitions 2
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic bankbranch
```

Start producer = post messages to topic (DO NOT open new terminal):
```bash
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic news
> Hello
> 1280 

bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic bankbranch 
> {"key1": val1a, "key2": val2a}
> {"key1": val1b, "key2": val2b}

bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic bankbranch --property parse.key=true --property key.separator=:
> 1:{"atmid": 1, "transid": 103}
> 1:{"atmid": 1, "transid": 103}
> 2:{"atmid": 2, "transid": 202}
> 2:{"atmid": 2, "transid": 203}
```

Start consumer = read messages from topic (open new terminal):
```bash
cd kafka_2.12-2.8.0

bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic news --from-beginning
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bankbranch --from-beginning
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bankbranch --from-beginning --property print.key=true --property key.separator=:
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bankbranch --group atm-app  # read messages 

bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group atm-app
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --topic bankbranch --group atm-app --reset-offsets --to-earliest --execute  # reset offset 
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --topic bankbranch --group atm-app --reset-offsets --shift-by -2 --execute  # set offset=2
```

### Python API

Create KafkaAdminClient object:
```python
admin_client = KafkaAdminClient(bootstrap_servers="localhost:9092", client_id='test')
```

Create topics:
```python
topic_list = []
new_topic = NewTopic(name="bankbranch", num_partitions=2, replication_factor=1)
topic_list.append(new_topic)
admin_client.create_topics(new_topics=topic_list)  # equiv. to: "kafka-topics.sh --bootstrap-server localhost:9092 --create --topic bankbranch --partitions 2 --replication_factor 1"
```

Describe topic:
```python
configs = admin_client.describe_configs(
    config_resources=[ConfigResource(ConfigResourceType.TOPIC, "bankbranch")])  # equiv. to: kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic bankbranch
```

Producer:
```python
producer = KafkaProducer(value_serializer=lambda v: json.dumps(v).encode('utf-8'))  # equiv. to: kafka-console-producer.sh --bootstrap-server localhost:9092 --topic bankbranch
producer.send("bankbranch", {'atmid':1, 'transid':100})
producer.send("bankbranch", {'atmid':2, 'transid':101})
```

Consumer:
```python
consumer = KafkaConsumer('bankbranch')
for msg in consumer:
    print(msg.value.decode("utf-8"))  # equiv. to: kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bankbranch
```

### Stream Example with MySQL

Prepare environment:
```bash
wget https://archive.apache.org/dist/kafka/2.8.0/kafka_2.12-2.8.0.tgz
tar -xzf kafka_2.12-2.8.0.tgz
start_mysql
mysql --host=127.0.0.1 --port=3306 --user=root --password=MTg4NDMtbWljaGFl
create database tolldata;
use tolldata;
create table livetolldata(timestamp datetime,vehicle_id int,vehicle_type char(15),toll_plaza_id smallint);
exit
python3 -m pip install kafka-python
python3 -m pip install mysql-connector-python==8.0.31
```

Start Kafka:
```bash
cd kafka_2.12-2.8.0
bin/zookeeper-server-start.sh config/zookeeper.properties
```

Configure & run scripts:
```bash
python3 toll_traffic_generator.py
python3 streaming_data_reader.py
```

Check tolldata database:
```bash
mysql --host=127.0.0.1 --port=3306 --user=root --password=MTg4NDMtbWljaGFl
use tolldata;
select * from livetolldata limit 10;
```

Similar code found with 4 license types