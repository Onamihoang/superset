# Kiến Trúc Apache Superset - Tài Liệu Chi Tiết

## Tổng Quan Kiến Trúc

Apache Superset là một nền tảng trực quan hóa dữ liệu và Business Intelligence (BI) hiện đại với kiến trúc phân tầng rõ ràng:

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend (React/TypeScript)              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ SQL Lab  │  │ Explore  │  │Dashboard │  │ Charts   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↕ REST API
┌─────────────────────────────────────────────────────────────┐
│                    Backend (Flask/Python)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  REST APIs   │  │  Commands    │  │   Models     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │Query Engine  │  │  DB Engines  │  │Cache Manager │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│              Temporary Storage & Caching Layer              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │    Redis     │  │  Memcached   │  │ File System  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                   Metadata Database                          │
│        (PostgreSQL/MySQL - Lưu config, users, dashboards)   │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│              Data Sources (50+ Database Types)               │
│  PostgreSQL │ MySQL │ BigQuery │ Snowflake │ Redshift ...   │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. TECH STACK CHI TIẾT

### Backend Stack (Python)

| Công Nghệ | Phiên Bản | Mục Đích |
|-----------|-----------|----------|
| **Flask** | 2.2.5+ | Web framework chính |
| **Flask-AppBuilder** | 5.0.2+ | Admin interface, authentication, RBAC |
| **SQLAlchemy** | Latest | ORM để quản lý database models |
| **Marshmallow** | 3.x | Schema validation và serialization |
| **Celery** | 5.3.6+ | Task queue xử lý bất đồng bộ |
| **Flask-Caching** | 2.1.0+ | Multi-backend caching |
| **Pandas** | 2.1.4+ | Xử lý và phân tích dữ liệu |
| **PyArrow** | 16.1.0+ | Columnar data format (hiệu năng cao) |
| **Jinja2** | - | Template engine cho SQL parameterization |
| **pytest** | - | Testing framework |

**File cấu hình chính:**
- `pyproject.toml` - Python tooling config
- `requirements/` - Dependency management

### Frontend Stack (React/TypeScript)

| Công Nghệ | Phiên Bản | Mục Đích |
|-----------|-----------|----------|
| **React** | 18+ | UI library |
| **Redux** | Latest | State management (global state) |
| **TypeScript** | Latest | Type safety |
| **Ant Design (antd)** | Latest | Component library |
| **Webpack** | - | Module bundler |
| **Jest** | - | Unit testing |
| **React Testing Library** | - | Component testing |
| **Playwright** | - | E2E testing (thay thế Cypress) |
| **D3.js / Plotly** | - | Chart rendering libraries |

**File cấu hình chính:**
- `superset-frontend/package.json` - Frontend dependencies và scripts
- `superset-frontend/webpack.config.js` - Build configuration

---

## 2. CÁC MODULE CHÍNH VÀ CHỨC NĂNG

### Backend Modules

#### A. Models (`superset/models/`)

**Chức năng:** Định nghĩa database schema cho metadata

**Các model chính:**

1. **Database** (`core.py:143-300`)
   - Lưu thông tin kết nối đến data sources
   - Fields: `database_name`, `sqlalchemy_uri`, `password` (encrypted)
   - Flags: `allow_run_async`, `allow_file_upload`, `allow_ctas`, `allow_dml`

2. **Dataset/Table** (`core.py`)
   - Metadata của bảng dữ liệu
   - Columns, metrics, filters

3. **Dashboard**
   - Dashboard layout, position_json
   - Relationship với charts

4. **Chart/Slice**
   - Chart configuration (params, viz_type)
   - Datasource relationship

5. **Query** (`sql_lab.py`)
   - SQL Lab query history
   - Status tracking, results caching

**File tham khảo:**
- `/home/user/superset/superset/models/core.py` - Core models
- `/home/user/superset/superset/models/sql_lab.py` - SQL Lab models
- `/home/user/superset/superset/models/dashboard.py` - Dashboard models

#### B. Database Engine Specs (`superset/db_engine_specs/`)

**Chức năng:** Hỗ trợ 50+ loại database khác nhau

**Kiến trúc:**
```python
# Base class - superset/db_engine_specs/base.py
class BaseEngineSpec:
    engine = "base"
    engine_aliases: set[str] = set()
    drivers: dict[str, str] = {}

    # Methods để customize behavior cho từng DB
    def convert_dttm(col, dttm)  # Time conversion
    def get_function_names()      # SQL functions
    def execute_cursor()          # Query execution
```

**Các engine specs có sẵn:**
- PostgreSQL (`postgres.py`)
- MySQL (`mysql.py`)
- BigQuery (`bigquery.py`)
- Snowflake (`snowflake.py`)
- Redshift (`redshift.py`)
- ClickHouse (`clickhouse.py`)
- DuckDB (`duckdb.py`)
- ... và 40+ loại khác

**Đặc điểm:**
- Plugin architecture: có thể thêm database mới qua entry_points
- Lazy loading: chỉ load engine spec khi cần
- Tự động detect driver phù hợp

**File tham khảo:**
- `/home/user/superset/superset/db_engine_specs/base.py`
- `/home/user/superset/superset/db_engine_specs/__init__.py` (load_engine_specs)

#### C. Connectors (`superset/connectors/`)

**Chức năng:** Abstraction layer cho datasources

**SQLA Connector** (`sqla/models.py`):
```python
class TableColumn:
    column_name: str
    is_dttm: bool        # Time dimension
    type: str            # Data type
    expression: str      # Computed columns

class Dataset(BaseDatasource):
    columns: list[TableColumn]
    metrics: list[Metric]
    table_name: str
    database_id: int

    # Query building
    def get_query_str(filters, groupby, metrics) -> str
```

**Chức năng chính:**
- Map database tables → Superset datasets
- Metadata caching (columns, data types)
- Query generation với filters, aggregations

**File tham khảo:**
- `/home/user/superset/superset/connectors/sqla/models.py`

#### D. SQL Lab (`superset/sqllab/`)

**Chức năng:** SQL Editor và query execution engine

**Components:**

1. **API Endpoints** (`api.py`)
   - `POST /sqllab/execute` - Execute SQL query
   - `GET /sqllab/results/{query_id}` - Fetch results
   - `GET /sqllab/export/{client_id}` - Export results

2. **SQL Executors** (`sql_json_executer.py`)
   - **SynchronousSqlJsonExecutor**: Blocking execution với timeout
   - **ASynchronousSqlJsonExecutor**: Celery tasks cho long-running queries

3. **Execution Context** (`sqllab_execution_context.py`)
```python
@dataclass
class SqlJsonExecutionContext:
    database_id: int
    schema: str
    sql: str
    template_params: dict  # Jinja2 variables
    async_flag: bool
    limit: int
    create_table_as_select: CreateTableAsSelect | None
```

**Query Execution Flow:**
```
User submits SQL
    ↓
/sqllab/execute API
    ↓
ExecuteSqlCommand (validation, access check)
    ↓
Jinja2 rendering (template_params)
    ↓
SqlJsonExecutor.execute()
    ↓
Database.get_engine() → SQLAlchemy connection
    ↓
BaseEngineSpec.execute_cursor() → Raw driver
    ↓
Results → SupersetResultSet (PyArrow format)
    ↓
Cache storage (Redis/Memcached)
    ↓
Response to frontend
```

**File tham khảo:**
- `/home/user/superset/superset/sqllab/api.py`
- `/home/user/superset/superset/sqllab/sql_json_executer.py`
- `/home/user/superset/superset/commands/sql_lab/execute.py`

#### E. Chart Data API (`superset/charts/data/api.py`)

**Chức năng:** Endpoint để lấy dữ liệu cho charts

**Flow:**
```
POST /charts/{id}/data/
    ↓
ChartDataCommand.run()
    ↓
QueryContextProcessor.get_data()
    ↓
QueryContext → QueryObject
    ↓
Dataset.get_query_str() → Generate SQL
    ↓
Execute via db_engine_spec
    ↓
Apply post-processing (pivots, rolling windows, etc.)
    ↓
Return formatted data
```

**QueryContext** (`common/query_context.py`):
```python
class QueryContext:
    datasource: BaseDatasource
    queries: list[QueryObject]  # Multiple queries per chart
    form_data: dict            # Chart configuration
    result_type: ChartDataResultType
    cache_values: dict
```

**File tham khảo:**
- `/home/user/superset/superset/charts/data/api.py`
- `/home/user/superset/superset/common/query_context.py`

#### F. Command Pattern (`superset/commands/`)

**Chức năng:** Encapsulate business logic với validation và transactions

**Base Command:**
```python
class BaseCommand:
    @abstractmethod
    def run() -> CommandResult: ...

    def validate() -> None: ...
```

**Ví dụ commands:**
- `ExecuteSqlCommand` - Execute SQL Lab queries
- `ChartDataCommand` - Fetch chart data
- `CreateDatasetCommand` - Create new dataset
- `WarmUpCacheCommand` - Pre-cache results

**Pattern:**
```
API Endpoint
    ↓
Command.validate()  # Access control, param validation
    ↓
Command.run()       # Business logic
    ↓
@transaction()      # Database transaction
    ↓
Return result
```

**File tham khảo:**
- `/home/user/superset/superset/commands/base.py`
- `/home/user/superset/superset/commands/sql_lab/execute.py`

#### G. Cache Manager (`superset/utils/cache_manager.py`)

**Chức năng:** Quản lý multi-layer caching

**Cache Types:**
```python
class CacheManager:
    _cache                      # General metadata cache
    _data_cache                 # Query results, chart data
    _thumbnail_cache            # Chart/dashboard thumbnails
    _filter_state_cache         # Dashboard filter state
    _explore_form_data_cache    # Explore form data
```

**Cache Key Generation:**
```python
def generate_cache_key(values_dict: dict, key_prefix: str = "") -> str:
    hash_str = md5_sha_from_dict(values_dict)
    return f"{key_prefix}{hash_str}"
```

**File tham khảo:**
- `/home/user/superset/superset/utils/cache_manager.py`
- `/home/user/superset/superset/utils/cache.py`

### Frontend Modules

#### A. SQL Lab (`superset-frontend/src/SqlLab/`)

**Chức năng:** SQL Editor với syntax highlighting, autocomplete, query execution

**Structure:**
```
SqlLab/
├── actions/
│   └── sqlLab.js         # Redux actions (runQuery, fetchResults)
├── components/
│   ├── QueryEditor.tsx   # SQL editor component
│   ├── ResultSet.tsx     # Query results display
│   ├── QueryTable.tsx    # Query history
│   └── SqlEditor.tsx     # Ace/Monaco editor wrapper
├── reducers/
│   └── sqlLab.js         # Redux state management
└── types.ts              # TypeScript interfaces
```

**Data Flow:**
```
User types SQL → QueryEditor
    ↓
Click "Run" → dispatch(runQuery)
    ↓
API call: POST /sqllab/execute
    ↓
If async: poll /async-events/
    ↓
dispatch(querySuccess) → Update Redux state
    ↓
ResultSet renders data
```

**File tham khảo:**
- `/home/user/superset/superset-frontend/src/SqlLab/`

#### B. Explore (`superset-frontend/src/explore/`)

**Chức năng:** Chart builder interface (no-code visualization)

**Components:**
- **ExploreViewContainer** - Main container
- **ControlPanelsContainer** - Chart controls (filters, metrics)
- **ExploreChartPanel** - Chart rendering area
- **SaveModal** - Save chart dialog

**Flow:**
```
User selects datasource
    ↓
Update formData (chart config)
    ↓
Redux action → POST /charts/{id}/data/
    ↓
Backend returns query results
    ↓
Chart library (D3/Plotly) renders visualization
```

**File tham khảo:**
- `/home/user/superset/superset-frontend/src/explore/`

#### C. Dashboard (`superset-frontend/src/dashboard/`)

**Chức năng:** Dashboard layout, filters, cross-filtering

**Components:**
- **Dashboard.jsx** - Main container
- **DashboardGrid.jsx** - Drag-and-drop layout
- **DashboardBuilder.jsx** - Edit mode
- **FilterBar** - Native filters

**State Management:**
- Redux store: `dashboardState`, `dashboardFilters`, `charts`
- Layout: `position_json` (grid coordinates)

**File tham khảo:**
- `/home/user/superset/superset-frontend/src/dashboard/`

#### D. Superset UI Core (`superset-frontend/packages/superset-ui-core/`)

**Chức năng:** Reusable UI component library

**Exports:**
- Ant Design wrappers
- Theme tokens
- Common hooks
- Utility functions

**Usage:**
```typescript
import { Button, Modal } from '@superset-ui/core';
// Thay vì import trực tiếp từ 'antd'
```

**File tham khảo:**
- `/home/user/superset/superset-frontend/packages/superset-ui-core/`

---

## 3. CÁCH KẾT NỐI ĐÊN DATABASE

### Kiến Trúc Kết Nối

Superset sử dụng **SQLAlchemy** làm ORM và connection pooling layer.

#### Database Model

```python
# superset/models/core.py
class Database(Model):
    __tablename__ = "dbs"

    # Kết nối
    sqlalchemy_uri: str      # e.g., "postgresql://user:pass@host:5432/db"
    password: str            # Encrypted separately

    # Connection pool settings
    extra: str               # JSON config:
    {
        "engine_params": {
            "pool_size": 10,
            "max_overflow": 20,
            "pool_timeout": 30,
            "pool_recycle": 3600
        }
    }

    # Features
    allow_run_async: bool    # Cho phép async queries
    allow_file_upload: bool  # CSV upload
    allow_ctas: bool        # CREATE TABLE AS SELECT
    allow_dml: bool         # INSERT/UPDATE/DELETE

    # Methods
    def get_engine() -> Engine:
        """Trả về SQLAlchemy engine với connection pooling"""

    @property
    def db_engine_spec() -> BaseEngineSpec:
        """Dynamic engine spec based on database type"""
```

#### Connection Flow

```
Request to query data
    ↓
Database.get_engine()
    ↓
SQLAlchemy creates connection from pool
    ↓
BaseEngineSpec.execute_cursor(cursor, query, params)
    ↓
Raw database driver (psycopg2, pymysql, etc.)
    ↓
Results returned
    ↓
Connection returned to pool
```

#### Database URI Examples

```python
# PostgreSQL
postgresql://username:password@hostname:5432/database_name

# MySQL
mysql://username:password@hostname:3306/database_name

# BigQuery
bigquery://project_id

# Snowflake
snowflake://username:password@account.region/database/schema

# SQLite
sqlite:////path/to/database.db
```

#### Connection Pooling

**Configuration** (trong `extra` field):
```json
{
    "engine_params": {
        "pool_size": 10,          // Số connections thường xuyên
        "max_overflow": 20,       // Thêm connections khi cần
        "pool_timeout": 30,       // Timeout khi chờ connection
        "pool_recycle": 3600,     // Recycle connection sau 1h
        "pool_pre_ping": true     // Test connection trước khi dùng
    }
}
```

**Lợi ích:**
- Tái sử dụng connections
- Giảm overhead tạo connection mới
- Tự động reconnect khi connection bị đứt

#### Database Engine Spec Selection

```python
# superset/db_engine_specs/__init__.py
def load_engine_specs():
    """Load all available engine specs"""
    # Standard specs từ superset/db_engine_specs/
    # External specs từ entry_points

def get_engine_spec(backend: str, driver: str) -> BaseEngineSpec:
    """Select appropriate engine spec based on database type"""
```

**Ví dụ:**
- `postgresql+psycopg2://...` → PostgresEngineSpec
- `mysql+pymysql://...` → MySQLEngineSpec
- `bigquery://...` → BigQueryEngineSpec

**File tham khảo:**
- `/home/user/superset/superset/models/core.py` - Database model
- `/home/user/superset/superset/db_engine_specs/base.py` - Engine spec base

---

## 4. CÓ THỂ QUERY NHIỀU DB CÙNG LÚC KHÔNG?

### Câu Trả Lời: CÓ (với điều kiện)

#### 4.1. Cấp Độ UI/Application

**✅ CÓ THỂ query nhiều databases đồng thời:**

1. **SQL Lab - Multiple Tabs:**
```
Tab 1: PostgreSQL → SELECT * FROM users
Tab 2: MySQL      → SELECT * FROM orders
Tab 3: BigQuery   → SELECT * FROM analytics
```
- Mỗi tab kết nối đến database riêng
- Queries chạy đồng thời (async)
- Results được cache riêng

2. **Dashboards - Multiple Data Sources:**
```
Dashboard "Sales Overview"
├── Chart 1: Data from PostgreSQL (sales DB)
├── Chart 2: Data from BigQuery (analytics DB)
└── Chart 3: Data from Snowflake (warehouse DB)
```
- Mỗi chart có thể dùng datasource khác nhau
- Charts load parallel
- Cache riêng per chart

#### 4.2. Cấp Độ Single Query

**❌ KHÔNG THỂ join cross-database trong 1 query:**

```sql
-- KHÔNG THỂ làm được trong Superset:
SELECT
    pg.user_id,
    bq.events
FROM postgresql.users pg
JOIN bigquery.analytics bq ON pg.user_id = bq.user_id
```

**Lý do:**
- Mỗi query execution chỉ connect đến 1 database
- `database_id` parameter xác định database cho query
- Không có query federation engine

**Code reference:**
```python
# superset/sqllab/sqllab_execution_context.py:75-76
class SqlJsonExecutionContext:
    database_id: int  # Single database per execution

    def __post_init__(self):
        self.database = Database.get(self.database_id)
```

#### 4.3. Workarounds Cho Cross-Database Queries

**Cách 1: Virtual Dataset với UNION**
```sql
-- Tạo virtual dataset kết hợp data từ nhiều sources
-- Phải chạy manually và lưu results
```

**Cách 2: ETL Pipeline**
```
External Tool (Airflow, dbt, etc.)
    ↓
Sync data from multiple sources
    ↓
Into single data warehouse
    ↓
Superset queries warehouse
```

**Cách 3: Database Federation (ở database level)**
```
PostgreSQL với Foreign Data Wrapper (FDW)
    ↓
Connect to MySQL, MongoDB, etc.
    ↓
Superset queries PostgreSQL
    ↓
PostgreSQL federates to other DBs
```

#### 4.4. Parallel Query Execution

**Superset DOES support parallel execution:**

**Async Queries:**
```python
# Multiple queries can run simultaneously via Celery
Query 1 (PostgreSQL) → Celery Worker 1
Query 2 (BigQuery)   → Celery Worker 2
Query 3 (Snowflake)  → Celery Worker 3
```

**Configuration:**
```python
# Database model
allow_run_async: bool = True

# Feature flag
SQLLAB_FORCE_RUN_ASYNC = True
```

**Flow:**
```
POST /sqllab/execute (async=true)
    ↓
Create Celery task
    ↓
Return job_id immediately
    ↓
Frontend polls /async-events/
    ↓
Results ready → notify frontend
```

**File tham khảo:**
- `/home/user/superset/superset/sqllab/sql_json_executer.py`
- `/home/user/superset/superset/async_events/async_query_manager.py`

---

## 5. DỮ LIỆU ĐƯỢC LƯU TẠM THỜI Ở ĐÂU?

### Multi-Layer Storage Architecture

Superset lưu temporary data ở **5 layers** khác nhau:

```
┌─────────────────────────────────────────────┐
│  Layer 1: In-Memory (Python Process)        │
│  - Pandas DataFrames during processing      │
│  - Request/response buffers                 │
└─────────────────────────────────────────────┘
              ↓ (serialize)
┌─────────────────────────────────────────────┐
│  Layer 2: Results Backend (Redis/Memcached) │
│  - Query results                            │
│  - Chart data                               │
│  - Serialized as JSON/MessagePack           │
└─────────────────────────────────────────────┘
              ↓ (fallback)
┌─────────────────────────────────────────────┐
│  Layer 3: Metadata Database Cache           │
│  - CacheKey table                           │
│  - SupersetMetastoreCache                   │
└─────────────────────────────────────────────┘
              ↓ (for frontend state)
┌─────────────────────────────────────────────┐
│  Layer 4: Browser Storage                   │
│  - localStorage: SQL Lab state, tabs        │
│  - sessionStorage: Temporary UI state       │
└─────────────────────────────────────────────┘
              ↓ (for async tasks)
┌─────────────────────────────────────────────┐
│  Layer 5: Celery Results Backend            │
│  - Async task status                        │
│  - Job metadata                             │
└─────────────────────────────────────────────┘
```

### Layer 1: In-Memory Processing

**Location:** Python process memory

**Data:**
- Pandas DataFrames during query execution
- Temporary variables in request handlers
- Request/response buffers

**Lifecycle:**
- Created: During query execution
- Cleared: After response sent or on error

**Example:**
```python
# superset/result_set.py
class SupersetResultSet:
    # Uses PyArrow Table (columnar format)
    # In-memory during processing
    # Serialized to MessagePack for caching
```

### Layer 2: Results Backend (Redis/Memcached)

**Location:** External cache server

**Configuration:**
```python
# superset/config.py
RESULTS_BACKEND = RedisCache(
    host='localhost',
    port=6379,
    key_prefix='superset_results'
)

DATA_CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 86400,  # 24 hours
    'CACHE_KEY_PREFIX': 'superset_'
}
```

**Data Stored:**
1. **Query Results** (SQL Lab)
   - Key: `superset_results:{query_id}`
   - Value: Serialized SupersetResultSet
   - TTL: 7 days (default)

2. **Chart Data**
   - Key: `superset_data:{md5_hash(query_params)}`
   - Value: JSON chart data
   - TTL: 24 hours (configurable)

3. **Dashboard Filter State**
   - Key: `superset_filter_state:{key}`
   - Value: JSON filter values
   - TTL: 24 hours

4. **Explore Form Data**
   - Key: `superset_explore:{key}`
   - Value: Chart configuration
   - TTL: 7 days

**Cache Manager Implementation:**
```python
# superset/utils/cache_manager.py
class CacheManager:
    def __init__(self):
        self._cache = cache  # General cache
        self._data_cache = data_cache  # Query results
        self._thumbnail_cache = thumbnail_cache
        self._filter_state_cache = filter_state_cache

    def data_cache(self):
        return self._data_cache

    def cache(self):
        return self._cache
```

**Cache Key Generation:**
```python
# superset/utils/cache.py
def generate_cache_key(values_dict: dict, key_prefix: str = "") -> str:
    """
    Generate MD5 hash từ query parameters

    Example:
    {
        "datasource": "1__table",
        "viz_type": "bar",
        "metrics": ["count"],
        "filters": [...]
    }
    →
    "superset_data:a1b2c3d4e5f6..."
    """
    json_data = json.dumps(values_dict, sort_keys=True)
    hash_str = hashlib.md5(json_data.encode()).hexdigest()
    return f"{key_prefix}{hash_str}"
```

**File tham khảo:**
- `/home/user/superset/superset/utils/cache_manager.py`
- `/home/user/superset/superset/utils/cache.py`

### Layer 3: Metadata Database (Fallback Cache)

**Location:** PostgreSQL/MySQL metadata database

**Table: `cache_keys`**
```sql
CREATE TABLE cache_keys (
    id INT PRIMARY KEY,
    cache_key VARCHAR(256),
    cache_value BLOB,
    created_on TIMESTAMP,
    datasource_uid VARCHAR(64)
);
```

**Sử dụng:**
- Fallback khi Redis/Memcached không available
- `SupersetMetastoreCache` implementation
- Slower nhưng more persistent

**Configuration:**
```python
# Khi không có Redis
CACHE_CONFIG = {
    'CACHE_TYPE': 'SupersetMetastoreCache',
    'CACHE_DEFAULT_TIMEOUT': 86400
}
```

### Layer 4: Browser Storage (Frontend)

**localStorage:**
```javascript
// SQL Lab state persistence
localStorage.setItem('sqllab', JSON.stringify({
    queries: [...],
    queryEditors: [...],
    tables: [...]
}));

// User preferences
localStorage.setItem('common', JSON.stringify({
    conf: {
        SQLLAB_TIMEOUT: 300,
        SQLLAB_VALIDATION_TIMEOUT: 10
    }
}));
```

**sessionStorage:**
```javascript
// Temporary UI state (cleared on tab close)
sessionStorage.setItem('dashboard_filters', JSON.stringify(filters));
```

**IndexedDB:**
- Không được dùng extensively hiện tại
- Có thể dùng cho offline mode trong tương lai

### Layer 5: Celery Results Backend

**Location:** Redis/RabbitMQ

**Data:**
- Async task status
- Job metadata
- Task results (temporary)

**Configuration:**
```python
# superset/config.py
CELERY_CONFIG = {
    'broker_url': 'redis://localhost:6379/0',
    'result_backend': 'redis://localhost:6379/0',
    'task_track_started': True,
    'result_expires': 86400  # 24 hours
}
```

**Flow:**
```
Submit async query
    ↓
Celery task created
    ↓
Task status: PENDING → RUNNING → SUCCESS
    ↓
Results stored in Celery backend
    ↓
Frontend polls for status
    ↓
Results moved to DATA_CACHE
    ↓
Celery result expired
```

**File tham khảo:**
- `/home/user/superset/superset/async_events/async_query_manager.py`

### Temporary Cache API

Superset cung cấp REST API để manage temporary cache:

```python
# superset/temporary_cache/api.py
class TemporaryCacheRestApi:
    POST   /api/v1/temp-cache/{resource}/{key}   # Create
    PUT    /api/v1/temp-cache/{resource}/{key}   # Update
    GET    /api/v1/temp-cache/{resource}/{key}   # Retrieve
    DELETE /api/v1/temp-cache/{resource}/{key}   # Delete
```

**Resources:**
- `explore_form_data` - Chart configuration
- `filter_state` - Dashboard filters
- `permalink` - Shareable links

**Example:**
```javascript
// Frontend stores explore state
POST /api/v1/temp-cache/explore_form_data/abc123
{
    "datasource": "1__table",
    "viz_type": "bar",
    "metrics": ["count"]
}

// Retrieve later
GET /api/v1/temp-cache/explore_form_data/abc123
→ Returns chart config
```

**File tham khảo:**
- `/home/user/superset/superset/temporary_cache/api.py`
- `/home/user/superset/superset/temporary_cache/commands/`

### Cache Warming (Pre-caching)

**Purpose:** Pre-cache slow queries để improve UX

**Commands:**
```python
# superset/commands/chart/warm_up_cache.py
class ChartWarmUpCacheCommand(BaseCommand):
    """Pre-cache chart data"""

# superset/commands/dataset/warm_up_cache.py
class DatasetWarmUpCacheCommand(BaseCommand):
    """Pre-cache dataset metadata"""
```

**CLI Usage:**
```bash
# Warm up all charts in dashboard
superset warm-up-cache --dashboard-id 1

# Warm up specific chart
superset warm-up-cache --chart-id 5
```

**Scheduled Warming:**
- Celery beat tasks
- Run during off-peak hours
- Cache expires theo TTL settings

**File tham khảo:**
- `/home/user/superset/superset/commands/chart/warm_up_cache.py`

---

## 6. QUERY EXECUTION PIPELINE CHI TIẾT

### SQL Lab Query Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Frontend: User submits SQL                               │
│    QueryEditor.tsx → runQuery() action                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. API: POST /api/v1/sqllab/execute                         │
│    ExecuteSqlCommand.run()                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Validation                                                │
│    - User has access to database?                           │
│    - SQL contains malicious code?                           │
│    - Database allows DML?                                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Template Rendering (Jinja2)                              │
│    SELECT * FROM table WHERE date = '{{ start_date }}'      │
│    → SELECT * FROM table WHERE date = '2024-01-01'          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Create Query Record                                       │
│    Query model persisted to metadata DB                     │
│    status = "running", start_time = now()                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
       ┌────────────────────┴────────────────────┐
       │                                         │
       ▼                                         ▼
┌──────────────────┐                  ┌──────────────────┐
│ Sync Execution   │                  │ Async Execution  │
│ (Blocking)       │                  │ (Celery Task)    │
└──────────────────┘                  └──────────────────┘
       │                                         │
       └────────────────────┬────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. Get Database Engine                                       │
│    Database.get_engine() → SQLAlchemy Engine                │
│    Connection pool → Get connection                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. Execute via Engine Spec                                   │
│    PostgresEngineSpec.execute_cursor(cursor, sql, params)   │
│    → psycopg2 driver executes raw SQL                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. Fetch Results                                             │
│    cursor.fetchall() → List of tuples                       │
│    Convert to SupersetResultSet (PyArrow Table)             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 9. Store Results                                             │
│    Serialize to MessagePack                                 │
│    Store in Redis with key: superset_results:{query_id}    │
│    TTL: 7 days                                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 10. Update Query Record                                      │
│     status = "success", end_time = now()                    │
│     rows = 1000, result_rows = Redis key                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 11. Return Response                                          │
│     {query_id, status, data}                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 12. Frontend: Render Results                                 │
│     ResultSet.tsx displays table                            │
│     Download/export options available                       │
└─────────────────────────────────────────────────────────────┘
```

### Chart Data Query Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. User configures chart in Explore                         │
│    Select datasource, metrics, filters, dimensions          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. FormData updated in Redux                                │
│    {                                                         │
│      datasource: "1__table",                                │
│      viz_type: "bar",                                       │
│      metrics: ["count"],                                    │
│      filters: [{...}]                                       │
│    }                                                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. API: POST /api/v1/charts/{id}/data/                      │
│    ChartDataCommand.run()                                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Create QueryContext                                       │
│    QueryContext(                                             │
│      datasource=Dataset.get(1),                             │
│      queries=[QueryObject(...)],                            │
│      result_type="full"                                     │
│    )                                                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Check Cache                                               │
│    cache_key = generate_cache_key(query_params)             │
│    If cached: return from Redis                             │
└─────────────────────────────────────────────────────────────┘
                            ↓ (cache miss)
┌─────────────────────────────────────────────────────────────┐
│ 6. Query Generation                                          │
│    Dataset.get_query_str() builds SQL:                      │
│    SELECT                                                    │
│      dim1, dim2,                                            │
│      COUNT(*) as count                                      │
│    FROM table                                                │
│    WHERE filter1 = 'value'                                  │
│    GROUP BY dim1, dim2                                      │
│    ORDER BY count DESC                                      │
│    LIMIT 10000                                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. Execute SQL (same as SQL Lab flow)                       │
│    Database.get_engine() → execute()                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. Post-Processing                                           │
│    - Apply pivots                                           │
│    - Rolling windows                                        │
│    - Cumulative metrics                                     │
│    - Sorting                                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 9. Cache Results                                             │
│    Store in Redis with cache_key                            │
│    TTL: 24 hours (default)                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 10. Return Chart Data                                        │
│     {                                                        │
│       data: [{dim1: 'A', count: 100}, ...],                │
│       columns: ['dim1', 'count'],                           │
│       query: "SELECT ...",                                  │
│       cache_key: "abc123"                                   │
│     }                                                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 11. Frontend: Render Chart                                   │
│     D3.js / Plotly / ECharts renders visualization          │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. SECURITY & ACCESS CONTROL

### Role-Based Access Control (RBAC)

**Framework:** Flask-AppBuilder

**Roles:**
- **Admin** - Full access
- **Alpha** - Create, edit, delete
- **Gamma** - View only
- **Custom roles** - Granular permissions

**Permissions:**
- `can_read` - View resource
- `can_write` - Edit resource
- `can_delete` - Delete resource
- `database_access` - Access specific database
- `datasource_access` - Access specific dataset

### Row-Level Security (RLS)

**Configuration:**
```python
# Applied at dataset level
# Injects WHERE clause based on user attributes

# Example:
User: john@company.com
Attribute: department = "Sales"

Original query:
SELECT * FROM orders

With RLS:
SELECT * FROM orders
WHERE department = 'Sales'
```

**Implementation:**
- Defined in Dataset model
- Applied in QueryContextProcessor
- Transparent to end users

---

## 8. MONITORING & DEBUGGING

### Logging

**Configuration:**
```python
# superset/config.py
LOG_FORMAT = '%(asctime)s:%(levelname)s:%(name)s:%(message)s'
LOG_LEVEL = 'DEBUG'
ENABLE_TIME_ROTATE = True
```

**Key loggers:**
- `superset.sqllab` - SQL Lab queries
- `superset.views.core` - API requests
- `superset.db_engine_specs` - Database operations

### Metrics & Monitoring

**Stats Logger:**
```python
from superset.stats_logger import stats_logger

stats_logger.incr('query_execution')
stats_logger.timing('query_duration', duration)
```

**Integration:**
- StatsD
- Prometheus
- Datadog

### Query Performance

**EXPLAIN Support:**
```python
# BaseEngineSpec.get_query_plan()
# Returns EXPLAIN output for query optimization
```

**Cost Estimates:**
```python
# Some databases support cost estimation
# PostgreSQL: EXPLAIN (FORMAT JSON) ...
```

---

## TÓM TẮT

### Kiến Trúc Tổng Quát
- **Frontend:** React/TypeScript với Redux state management
- **Backend:** Flask/Python với SQLAlchemy ORM
- **Database Support:** 50+ loại databases qua engine specs
- **Caching:** Multi-layer với Redis/Memcached
- **Async Processing:** Celery cho long-running queries

### Kết Nối Database
- SQLAlchemy làm ORM và connection pooling
- Engine specs customize behavior cho từng database type
- Connection pool tái sử dụng connections
- Encrypted credentials storage

### Multi-Database Queries
- ✅ Có thể query nhiều DBs song song (parallel tabs, multiple charts)
- ❌ Không thể join cross-database trong 1 query
- Workarounds: ETL pipeline, database federation

### Temporary Data Storage
1. **In-Memory:** Pandas DataFrames trong Python process
2. **Redis/Memcached:** Query results, chart data (primary cache)
3. **Metadata DB:** Fallback cache trong PostgreSQL/MySQL
4. **Browser Storage:** localStorage cho SQL Lab state
5. **Celery Backend:** Async task results

### File Tham Khảo Quan Trọng
- Backend models: `/home/user/superset/superset/models/core.py`
- DB engines: `/home/user/superset/superset/db_engine_specs/base.py`
- SQL Lab: `/home/user/superset/superset/sqllab/api.py`
- Caching: `/home/user/superset/superset/utils/cache_manager.py`
- Query execution: `/home/user/superset/superset/common/query_context.py`

---

**Tài liệu này được tạo tự động dựa trên phân tích source code Apache Superset.**
**Ngày tạo:** 2025-12-01
**Version:** 4.1.0-dev (branch: claude/document-architecture-db-01EMkLLVq5zh2eykRrygEjLh)
