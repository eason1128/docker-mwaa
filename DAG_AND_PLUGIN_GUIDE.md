# DAG and Plugin Development Guide for AWS MWAA Local Runner

This guide provides comprehensive instructions for adding DAGs and plugins to your AWS MWAA Local Runner environment.

## Table of Contents

1. [Adding New DAGs](#adding-new-dags)
2. [Adding New Plugins](#adding-new-plugins)
3. [Testing Workflow](#testing-workflow)
4. [Best Practices](#best-practices)
5. [Examples](#examples)
6. [Troubleshooting](#troubleshooting)

## Adding New DAGs

### Directory Structure
```
aws-mwaa-local-runner/
├── dags/
│   ├── example_dag_with_taskflow_api.py    # Existing example
│   ├── your_new_dag.py                     # Your new DAG
│   └── my_etl_pipeline.py                  # Another DAG
└── ...
```

### Step-by-Step DAG Creation

#### 1. Create DAG File
Create a new Python file in the `dags/` directory:

```bash
# Navigate to dags directory
cd dags/

# Create new DAG file
touch my_first_dag.py
```

#### 2. Basic DAG Template
```python
from datetime import datetime, timedelta
from airflow.decorators import dag, task
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

# Default arguments for all tasks
default_args = {
    'owner': 'your-name',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

@dag(
    dag_id='my_first_dag',
    default_args=default_args,
    description='My first DAG',
    schedule_interval=timedelta(days=1),
    catchup=False,
    tags=['example', 'tutorial'],
)
def my_first_dag():
    """
    ### My First DAG
    This is a simple example DAG demonstrating basic concepts.
    """
    
    @task()
    def extract_data():
        """Extract data from source"""
        return {"data": "sample_data", "timestamp": str(datetime.now())}
    
    @task()
    def transform_data(data):
        """Transform the extracted data"""
        transformed = data.copy()
        transformed["processed"] = True
        return transformed
    
    @task()
    def load_data(data):
        """Load data to destination"""
        print(f"Loading data: {data}")
        return "Data loaded successfully"
    
    # Define task dependencies
    extracted = extract_data()
    transformed = transform_data(extracted)
    loaded = load_data(transformed)

# Instantiate the DAG
dag_instance = my_first_dag()
```

#### 3. Advanced DAG Examples

**ETL Pipeline with External Dependencies:**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta

def extract_from_api(**context):
    """Extract data from external API"""
    import requests
    # Your extraction logic here
    response = requests.get('https://api.example.com/data')
    return response.json()

def transform_data(**context):
    """Transform extracted data"""
    ti = context['task_instance']
    raw_data = ti.xcom_pull(task_ids='extract_task')
    # Your transformation logic here
    return processed_data

dag = DAG(
    'etl_pipeline',
    default_args={
        'owner': 'data-team',
        'start_date': datetime(2025, 1, 1),
        'retries': 2,
    },
    schedule_interval='@daily',
    catchup=False,
    description='ETL Pipeline with external data source'
)

extract_task = PythonOperator(
    task_id='extract_task',
    python_callable=extract_from_api,
    dag=dag
)

transform_task = PythonOperator(
    task_id='transform_task',
    python_callable=transform_data,
    dag=dag
)

load_task = PostgresOperator(
    task_id='load_task',
    postgres_conn_id='postgres_default',
    sql='INSERT INTO processed_data SELECT * FROM staging_data;',
    dag=dag
)

# Set dependencies
extract_task >> transform_task >> load_task
```

#### 4. Deploy DAG
```bash
# DAGs are automatically detected when placed in dags/ directory
# Restart MWAA Local Runner to pick up changes
./mwaa-local-env start
```

## Adding New Plugins

### Plugin Directory Structure
```
aws-mwaa-local-runner/
├── plugins/
│   ├── __init__.py
│   ├── operators/           # Custom operators
│   │   ├── __init__.py
│   │   ├── my_custom_operator.py
│   │   └── api_operator.py
│   ├── hooks/              # External system connections
│   │   ├── __init__.py
│   │   ├── my_api_hook.py
│   │   └── database_hook.py
│   ├── sensors/            # Event detection
│   │   ├── __init__.py
│   │   └── file_sensor.py
│   ├── utility/            # Helper functions
│   │   ├── __init__.py
│   │   └── common_utils.py
│   └── config/             # Configuration
│       └── settings.py
```

### Step-by-Step Plugin Creation

#### 1. Create Custom Operator

**File: plugins/operators/my_custom_operator.py**
```python
from airflow.models import BaseOperator
from airflow.utils.decorators import apply_defaults
from typing import Any, Dict, Optional

class MyCustomOperator(BaseOperator):
    """
    Custom operator that performs specific business logic.
    
    :param my_parameter: Custom parameter for the operator
    :param connection_id: Connection ID for external service
    """
    
    template_fields = ['my_parameter']
    ui_color = '#e8f4fd'
    ui_fgcolor = '#000000'
    
    @apply_defaults
    def __init__(
        self,
        my_parameter: str,
        connection_id: Optional[str] = None,
        *args,
        **kwargs
    ):
        super().__init__(*args, **kwargs)
        self.my_parameter = my_parameter
        self.connection_id = connection_id
    
    def execute(self, context: Dict[str, Any]) -> Any:
        """Execute the operator logic"""
        self.log.info(f"Executing with parameter: {self.my_parameter}")
        
        # Your custom logic here
        result = self._perform_custom_operation()
        
        self.log.info(f"Operation completed with result: {result}")
        return result
    
    def _perform_custom_operation(self) -> str:
        """Perform the actual custom operation"""
        # Implement your business logic
        return f"Processed: {self.my_parameter}"
```

#### 2. Create Custom Hook

**File: plugins/hooks/my_api_hook.py**
```python
from airflow.hooks.base import BaseHook
from typing import Dict, Any, Optional
import requests

class MyApiHook(BaseHook):
    """
    Custom hook for interacting with external API.
    
    :param api_conn_id: Connection ID containing API credentials
    """
    
    conn_name_attr = 'api_conn_id'
    default_conn_name = 'my_api_default'
    
    def __init__(self, api_conn_id: str = default_conn_name):
        super().__init__()
        self.api_conn_id = api_conn_id
        self._base_url = None
        self._session = None
    
    def get_conn(self) -> requests.Session:
        """Get connection session"""
        if self._session is None:
            connection = self.get_connection(self.api_conn_id)
            self._base_url = f"https://{connection.host}"
            
            self._session = requests.Session()
            if connection.login and connection.password:
                self._session.auth = (connection.login, connection.password)
            
            # Add headers if specified in connection extra
            if connection.extra:
                import json
                extra = json.loads(connection.extra)
                if 'headers' in extra:
                    self._session.headers.update(extra['headers'])
        
        return self._session
    
    def make_request(
        self, 
        endpoint: str, 
        method: str = 'GET', 
        data: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """Make API request"""
        session = self.get_conn()
        url = f"{self._base_url}/{endpoint.lstrip('/')}"
        
        response = session.request(method=method, url=url, json=data)
        response.raise_for_status()
        
        return response.json()
```

#### 3. Create Custom Sensor

**File: plugins/sensors/file_sensor.py**
```python
from airflow.sensors.base import BaseSensorOperator
from airflow.utils.decorators import apply_defaults
import os
from typing import Any, Dict

class FileSensor(BaseSensorOperator):
    """
    Sensor that waits for a file to appear in the filesystem.
    
    :param filepath: Path to the file to check
    :param recursive: Whether to check subdirectories
    """
    
    template_fields = ['filepath']
    
    @apply_defaults
    def __init__(
        self,
        filepath: str,
        recursive: bool = False,
        *args,
        **kwargs
    ):
        super().__init__(*args, **kwargs)
        self.filepath = filepath
        self.recursive = recursive
    
    def poke(self, context: Dict[str, Any]) -> bool:
        """Check if file exists"""
        self.log.info(f"Checking for file: {self.filepath}")
        
        if self.recursive:
            # Check subdirectories
            for root, dirs, files in os.walk(os.path.dirname(self.filepath)):
                if os.path.basename(self.filepath) in files:
                    return True
            return False
        else:
            return os.path.exists(self.filepath)
```

#### 4. Create Utility Module

**File: plugins/utility/data_utils.py**
```python
"""Common utility functions for data processing"""

import pandas as pd
from typing import Dict, List, Any
import json

def validate_data_schema(data: Dict[str, Any], required_fields: List[str]) -> bool:
    """Validate that data contains required fields"""
    return all(field in data for field in required_fields)

def clean_data(df: pd.DataFrame) -> pd.DataFrame:
    """Common data cleaning operations"""
    # Remove duplicates
    df = df.drop_duplicates()
    
    # Handle missing values
    df = df.fillna('')
    
    # Standardize column names
    df.columns = df.columns.str.lower().str.replace(' ', '_')
    
    return df

def format_json_response(data: Any) -> str:
    """Format data as JSON string"""
    return json.dumps(data, indent=2, default=str)

class DataProcessor:
    """Reusable data processing class"""
    
    def __init__(self, config: Dict[str, Any]):
        self.config = config
    
    def process(self, data: Any) -> Any:
        """Process data according to configuration"""
        # Your processing logic here
        return data
```

### Using Custom Plugins in DAGs

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

# Import your custom plugins
from plugins.operators.my_custom_operator import MyCustomOperator
from plugins.hooks.my_api_hook import MyApiHook
from plugins.sensors.file_sensor import FileSensor
from plugins.utility.data_utils import validate_data_schema, DataProcessor

dag = DAG(
    'dag_with_custom_plugins',
    default_args={
        'owner': 'data-team',
        'start_date': datetime(2025, 1, 1),
    },
    schedule_interval='@daily',
    catchup=False
)

# Use custom sensor
wait_for_file = FileSensor(
    task_id='wait_for_input_file',
    filepath='/tmp/input_data.csv',
    timeout=300,
    poke_interval=30,
    dag=dag
)

# Use custom operator
process_data = MyCustomOperator(
    task_id='process_with_custom_operator',
    my_parameter='example_value',
    connection_id='my_api_conn',
    dag=dag
)

# Use custom hook in Python operator
def extract_from_api(**context):
    hook = MyApiHook(api_conn_id='my_api_conn')
    data = hook.make_request('data/endpoint')
    return data

extract_task = PythonOperator(
    task_id='extract_from_api',
    python_callable=extract_from_api,
    dag=dag
)

# Set dependencies
wait_for_file >> extract_task >> process_data
```

## Testing Workflow

### 1. Test DAGs Locally

```bash
# Test DAG syntax
python dags/my_dag.py

# List all DAGs
./mwaa-local-env start
# Then in Airflow UI, check if DAG appears

# Test specific task
docker exec -it aws-mwaa-local-runner-2_10_3-local-runner-1 airflow tasks test my_dag task_id 2025-01-01
```

### 2. Test Plugins

```bash
# Test plugin import
docker exec -it aws-mwaa-local-runner-2_10_3-local-runner-1 python -c "from plugins.operators.my_custom_operator import MyCustomOperator; print('Import successful')"

# Test plugin functionality
docker exec -it aws-mwaa-local-runner-2_10_3-local-runner-1 python -c "
from plugins.utility.data_utils import validate_data_schema
result = validate_data_schema({'field1': 'value'}, ['field1'])
print(f'Validation result: {result}')
"
```

### 3. Debug Common Issues

```bash
# Check plugin loading
docker exec -it aws-mwaa-local-runner-2_10_3-local-runner-1 airflow plugins

# Check DAG parsing errors
docker logs aws-mwaa-local-runner-2_10_3-local-runner-1 | grep -i error

# Access Airflow CLI
docker exec -it aws-mwaa-local-runner-2_10_3-local-runner-1 bash
```

## Best Practices

### DAG Best Practices

1. **Use Descriptive Names**
   ```python
   dag_id='user_data_etl_pipeline'  # Good
   dag_id='dag1'                    # Bad
   ```

2. **Set Appropriate Timeouts**
   ```python
   default_args = {
       'execution_timeout': timedelta(hours=2),
       'retry_delay': timedelta(minutes=5),
   }
   ```

3. **Use TaskFlow API When Possible**
   ```python
   @dag(schedule_interval='@daily')
   def modern_dag():
       @task()
       def my_task():
           return "result"
   ```

4. **Implement Proper Error Handling**
   ```python
   @task()
   def robust_task():
       try:
           # Your logic
           return result
       except Exception as e:
           logging.error(f"Task failed: {e}")
           raise
   ```

### Plugin Best Practices

1. **Follow Airflow Conventions**
   - Operators inherit from `BaseOperator`
   - Hooks inherit from `BaseHook`
   - Sensors inherit from `BaseSensorOperator`

2. **Implement Proper Logging**
   ```python
   def execute(self, context):
       self.log.info("Starting operation")
       # Your logic
       self.log.info("Operation completed")
   ```

3. **Use Template Fields**
   ```python
   template_fields = ['parameter_name']  # Enable Jinja templating
   ```

4. **Add Type Hints**
   ```python
   def execute(self, context: Dict[str, Any]) -> Any:
       pass
   ```

## Advanced Examples

### Complex ETL DAG with Custom Components

```python
from datetime import datetime, timedelta
from airflow.decorators import dag, task
from plugins.operators.my_custom_operator import MyCustomOperator
from plugins.hooks.my_api_hook import MyApiHook
from plugins.utility.data_utils import DataProcessor

@dag(
    dag_id='advanced_etl_pipeline',
    start_date=datetime(2025, 1, 1),
    schedule_interval='@daily',
    catchup=False,
    tags=['production', 'etl']
)
def advanced_etl_pipeline():
    """Advanced ETL pipeline using custom components"""
    
    @task()
    def extract_multiple_sources():
        """Extract from multiple data sources"""
        hook = MyApiHook()
        
        # Extract from API
        api_data = hook.make_request('daily-data')
        
        # Extract from database (using built-in hooks)
        from airflow.providers.postgres.hooks.postgres import PostgresHook
        pg_hook = PostgresHook(postgres_conn_id='postgres_default')
        db_data = pg_hook.get_records("SELECT * FROM source_table")
        
        return {
            'api_data': api_data,
            'db_data': db_data
        }
    
    @task()
    def transform_data(raw_data):
        """Transform extracted data"""
        processor = DataProcessor({'mode': 'production'})
        
        # Process API data
        api_processed = processor.process(raw_data['api_data'])
        
        # Process DB data
        db_processed = processor.process(raw_data['db_data'])
        
        return {
            'api_processed': api_processed,
            'db_processed': db_processed
        }
    
    # Use custom operator for specialized processing
    custom_processing = MyCustomOperator(
        task_id='custom_business_logic',
        my_parameter="{{ ds }}",  # Use Airflow macros
        connection_id='business_api'
    )
    
    @task()
    def load_to_warehouse(processed_data):
        """Load to data warehouse"""
        # Your loading logic
        return "Data loaded successfully"
    
    # Define workflow
    raw = extract_multiple_sources()
    processed = transform_data(raw)
    custom_processing >> load_to_warehouse(processed)

# Instantiate the DAG
etl_dag = advanced_etl_pipeline()
```

## Troubleshooting

### Common DAG Issues

1. **DAG Not Appearing**
   - Check syntax: `python dags/your_dag.py`
   - Verify DAG is in `dags/` directory
   - Check Airflow logs for parsing errors

2. **Import Errors**
   - Ensure all dependencies are in `requirements/requirements.txt`
   - Check Python path and module structure

3. **Task Failures**
   - Check task logs in Airflow UI
   - Verify connections and variables
   - Test tasks individually

### Common Plugin Issues

1. **Plugin Not Loading**
   - Check `__init__.py` files exist
   - Verify plugin structure
   - Check import statements

2. **Custom Operator Issues**
   - Inherit from correct base class
   - Implement required methods
   - Check template fields

## Reference Links

- [Airflow DAG Documentation](https://airflow.apache.org/docs/apache-airflow/2.10.3/authoring-and-scheduling/dags.html)
- [Airflow Plugin Documentation](https://airflow.apache.org/docs/apache-airflow/2.10.3/authoring-and-scheduling/plugins.html)
- [TaskFlow API Tutorial](https://airflow.apache.org/docs/apache-airflow/2.10.3/tutorial/taskflow.html)

---

**Last Updated**: July 23, 2025  
**Compatible With**: AWS MWAA Local Runner v2.10.3