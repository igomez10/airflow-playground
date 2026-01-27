# Distributed Databases - Airflow Setup

This project provides a production-ready Apache Airflow 2.8.1 setup using Docker Compose with CeleryExecutor for distributed task execution.

## Architecture

| Service | Description | Port |
|---------|-------------|------|
| PostgreSQL 15 | Metadata database | 5432 |
| Redis 7 | Celery message broker | 6379 |
| Airflow Webserver | Web UI | 8080 |
| Airflow Scheduler | DAG scheduling | 8974 |
| Airflow Worker | Task execution | - |
| Airflow Triggerer | Async triggers | - |
| Flower (optional) | Celery monitoring | 5555 |

## Prerequisites

- Docker and Docker Compose installed
- At least 4GB of RAM allocated to Docker

## Getting Started

### 1. Set up environment

The `.env` file is already configured with:

```
AIRFLOW_UID=50000
```

### 2. Start services

```bash
# Initialize the database and create admin user
docker compose up airflow-init

# Start all services
docker compose up -d
```

### 3. Access the UI

- **Airflow Web UI**: http://localhost:8080
  - Username: `airflow`
  - Password: `airflow`

- **Flower (Celery Monitor)**: http://localhost:5555 (if enabled)

### 4. Enable optional services

```bash
# Start with Flower (Celery monitoring)
docker compose --profile flower up -d

# Start with debug CLI
docker compose --profile debug up -d
```

### 5. Stop services

```bash
docker compose down

# To also remove volumes (resets all data)
docker compose down -v
```

## Project Structure

```
.
├── docker-compose.yml    # Service definitions
├── .env                  # Environment variables
├── dags/                 # DAG definitions (mounted to container)
├── logs/                 # Execution logs
├── plugins/              # Custom operators and hooks
└── config/               # Airflow configuration files
```

## Creating a New DAG

DAGs are Python files placed in the `dags/` directory. They are automatically detected by the scheduler.

### Existing Example

See `dags/example_dag.py` for a working example with Python and Bash operators.

### Basic DAG Template

Create a new file in `dags/`, replacing `<your_dag_name>` with your DAG's name:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator

# Default arguments for all tasks
default_args = {
    'owner': 'airflow',
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

def my_python_function():
    """Your Python logic here."""
    print("Running my task!")
    return "Task completed"

# Define the DAG
with DAG(
    dag_id='<your_dag_name>',  # Change this to your DAG name
    default_args=default_args,
    description='Your DAG description',
    start_date=datetime(2024, 1, 1),
    schedule=timedelta(days=1),  # Run daily
    catchup=False,
    tags=['custom'],
) as dag:

    # Python task
    task_1 = PythonOperator(
        task_id='python_task',
        python_callable=my_python_function,
    )

    # Bash task
    task_2 = BashOperator(
        task_id='bash_task',
        bash_command='echo "Hello from bash!"',
    )

    # Set task dependencies
    task_1 >> task_2  # task_1 runs before task_2
```

### DAG Best Practices

1. **Unique DAG ID**: Each DAG must have a unique `dag_id`
2. **Idempotent tasks**: Tasks should produce the same result if run multiple times
3. **No heavy computation at top level**: Keep DAG file parsing lightweight
4. **Use `catchup=False`**: Unless you need to backfill historical runs

### Common Operators

| Operator | Use Case |
|----------|----------|
| `PythonOperator` | Execute Python functions |
| `BashOperator` | Run shell commands |
| `PostgresOperator` | Execute SQL on PostgreSQL |
| `EmailOperator` | Send email notifications |
| `BranchPythonOperator` | Conditional branching |

### Task Dependencies

```python
# Sequential
task_1 >> task_2 >> task_3

# Parallel then join
[task_1, task_2] >> task_3

# Fan out
task_1 >> [task_2, task_3]
```

## Useful Commands

```bash
# View logs
docker compose logs -f webserver
docker compose logs -f scheduler

# Run Airflow CLI commands
docker compose exec airflow-webserver airflow dags list
docker compose exec airflow-webserver airflow tasks list example_dag

# Trigger a DAG manually
docker compose exec airflow-webserver airflow dags trigger example_dag

# Trigger your own DAG (replace <dag_id> with your DAG name)
docker compose exec airflow-webserver airflow dags trigger <dag_id>

# Check service health
docker compose ps
```

## Troubleshooting

### Services not starting

Check that Docker has enough resources:
```bash
docker compose logs airflow-init
```

### DAG not appearing in UI

1. Check for syntax errors in your DAG file
2. View scheduler logs: `docker compose logs scheduler`
3. Ensure the file is in the `dags/` directory

### Task failures

Check task logs in the Airflow UI or in `logs/dag_id=<dag_id>/`

## Configuration

Key environment variables (set in docker-compose.yml):

| Variable | Value | Description |
|----------|-------|-------------|
| `AIRFLOW__CORE__EXECUTOR` | CeleryExecutor | Distributed execution |
| `AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION` | true | New DAGs start paused |
| `AIRFLOW__CORE__LOAD_EXAMPLES` | false | Don't load example DAGs |
| `AIRFLOW__API__AUTH_BACKENDS` | basic_auth,session | API authentication |
