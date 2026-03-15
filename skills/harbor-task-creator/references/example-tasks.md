# Example Task Walkthroughs

Four complete, annotated task examples showing simple, medium, complex, and GPU/MCP patterns.

---

## Example 1: Simple -- Fix a Python Bug

A minimal task where the agent fixes a single bug in a Python file.

### task.toml

```toml
version = "1.0"

[metadata]
difficulty = "easy"
category = "programming"
tags = ["python", "debugging"]

[agent]
timeout_sec = 300.0

[verifier]
timeout_sec = 60.0
```

Short timeouts -- this is an easy task that should complete quickly. All other fields use defaults (1 CPU, 2048 MB memory, etc.).

### instruction.md

```markdown
# Fix the Fibonacci Function

The file `/app/fibonacci.py` contains a function `fib(n)` that should
return the nth Fibonacci number (0-indexed: fib(0)=0, fib(1)=1, fib(2)=1, ...).

The function has a bug that causes it to return incorrect values for n > 2.
Fix the bug. Do not rename the function or change its signature.
```

Note: Specifies exact file path, expected behavior with concrete examples, and constraints. The output file path (`/app/fibonacci.py`) is explicitly mentioned -- this satisfies the `file_reference_mentioned` quality check.

### environment/Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY fibonacci.py /app/fibonacci.py
```

Note: Uses `python:3.12-slim` because the task is pure Python. Does NOT copy tests/ or solution/.

### environment/fibonacci.py

```python
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 3)  # Bug: should be n - 2
```

### tests/test.sh

```bash
#!/bin/bash
if python3 -c "
import sys
sys.path.insert(0, '/app')
from fibonacci import fib
assert fib(0) == 0
assert fib(1) == 1
assert fib(5) == 5
assert fib(10) == 55
print('All tests passed')
"; then
  echo 1 > /logs/verifier/reward.txt
else
  echo 0 > /logs/verifier/reward.txt
fi
```

Note: Simple inline Python test. Both branches write to reward.txt -- the reward file is always written. For this simple task, pytest would be overkill.

### solution/solve.sh

```bash
#!/bin/bash
sed -i 's/fib(n - 3)/fib(n - 2)/' /app/fibonacci.py
```

Note: Uses sed for a targeted fix -- this is real computation (pattern matching and replacement), not a hardcoded answer. Validate with: `harbor trials start --path ./fix-fibonacci --agent oracle`

---

## Example 2: Medium -- Build a REST API

The agent builds a small API from a specification. Tests use pytest with CTRF output.

### task.toml

```toml
version = "1.0"

[metadata]
difficulty = "medium"
category = "programming"
tags = ["python", "flask", "api"]

[agent]
timeout_sec = 600.0

[verifier]
timeout_sec = 120.0

[environment]
memory_mb = 4096
```

Increased memory for the Flask application. Verifier gets 120s to install test deps and run pytest.

### instruction.md

```markdown
# Build a Task Tracker API

Create a REST API at `/app/api.py` using Flask that runs on port 5000.

Endpoints:
- `POST /tasks` -- Create a task. Body: `{"title": "string", "done": false}`. Returns the created task with an auto-incremented integer `id` field. Status 201.
- `GET /tasks` -- List all tasks. Returns `{"tasks": [...]}`. Status 200.
- `GET /tasks/<id>` -- Get a single task by id. Returns the task object or 404.
- `PATCH /tasks/<id>` -- Update a task. Body: any subset of `{"title", "done"}`. Returns updated task or 404.
- `DELETE /tasks/<id>` -- Delete a task. Returns `{"deleted": true}` or 404.

Store tasks in memory (no database needed). IDs start at 1.
```

Note: Every endpoint has its method, path, request body, response body, and status code specified. The output file path `/app/api.py` is explicitly stated. The response schema for each endpoint is documented -- this is important for the `structured_data_schema` quality check.

### environment/Dockerfile

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir flask==3.1.0
WORKDIR /app
```

Note: Flask is pinned to an exact version. The agent might need Flask to develop the API, so it belongs in the Dockerfile. Test dependencies (pytest, requests) go in test.sh.

### tests/test.sh

```bash
#!/bin/bash
set -uo pipefail

# Install test dependencies (pinned versions)
apt-get update && apt-get install -y curl
curl -LsSf https://astral.sh/uv/0.9.7/install.sh | sh
source $HOME/.local/bin/env

# Start the API in background
cd /app && python3 api.py &
API_PID=$!
sleep 2

# Run pytest with CTRF for Harbor Viewer
uvx --with pytest==8.4.1 --with pytest-json-ctrf==0.3.5 --with requests==2.32.3 \
  pytest --ctrf /logs/verifier/ctrf.json /tests/test_api.py -rA || true

EXIT_CODE=${PIPESTATUS[0]}
if [ "$EXIT_CODE" -eq 0 ]; then
  echo 1 > /logs/verifier/reward.txt
else
  echo 0 > /logs/verifier/reward.txt
fi

kill $API_PID 2>/dev/null
```

Key patterns demonstrated:
- `set -uo pipefail` without `-e`: prevents early exit before writing reward
- `|| true` after pytest: prevents pipefail from killing the script
- `${PIPESTATUS[0]}`: captures pytest's actual exit code, not the `|| true`
- uv for fast, isolated test dependency installation
- All test dependencies pinned: `pytest==8.4.1`, `requests==2.32.3`
- CTRF output to `/logs/verifier/ctrf.json` for the Harbor Viewer
- API started in background, killed after tests

### tests/test_api.py

```python
import requests

BASE = "http://localhost:5000"

def test_create_and_list():
    r = requests.post(f"{BASE}/tasks", json={"title": "Buy milk", "done": False})
    assert r.status_code == 201
    task = r.json()
    assert task["id"] == 1
    assert task["title"] == "Buy milk"

    r = requests.get(f"{BASE}/tasks")
    assert r.status_code == 200
    assert len(r.json()["tasks"]) == 1

def test_get_single():
    r = requests.get(f"{BASE}/tasks/1")
    assert r.status_code == 200
    assert r.json()["title"] == "Buy milk"

def test_update():
    r = requests.patch(f"{BASE}/tasks/1", json={"done": True})
    assert r.status_code == 200
    assert r.json()["done"] is True

def test_delete():
    r = requests.delete(f"{BASE}/tasks/1")
    assert r.status_code == 200
    assert r.json()["deleted"] is True

def test_not_found():
    assert requests.get(f"{BASE}/tasks/999").status_code == 404
```

Note: Every endpoint and behavior described in instruction.md is tested. Test function names are descriptive. This satisfies both `behavior_in_tests` and `informative_test_structure` quality checks.

---

## Example 3: Complex -- Multi-Container with Database

A task requiring the agent to work with an existing PostgreSQL database. Demonstrates docker-compose.yaml, health checks, and service networking.

### task.toml

```toml
version = "1.0"

[metadata]
difficulty = "hard"
category = "devops"
tags = ["sql", "postgres", "python", "migration"]

[agent]
timeout_sec = 900.0

[verifier]
timeout_sec = 180.0
env = { PGHOST = "postgres", PGUSER = "app", PGPASSWORD = "secret", PGDATABASE = "shop" }

[environment]
cpus = 2
memory_mb = 4096
```

Note: The `[verifier].env` injects Postgres connection details into test.sh so the test script can connect to the database service. The agent gets 15 minutes because database tasks involve more exploration.

### instruction.md

```markdown
# Database Migration

A PostgreSQL database is running at `postgresql://app:secret@postgres:5432/shop`.

The `products` table has columns: `id`, `name`, `price`, `category`.

Write a Python migration script at `/app/migrate.py` that:
1. Adds a `discount_pct` column (integer, default 0, range 0-100)
2. Sets `discount_pct = 20` for all products in category "electronics"
3. Creates a view `discounted_products` showing `id`, `name`, `price`, `discount_pct`, and `final_price` (price * (100 - discount_pct) / 100)

Run the script with `python3 /app/migrate.py`.
```

Note: Connection string, table schema, exact column specs, and the output file path are all spelled out. The agent knows exactly what to build and where.

### environment/docker-compose.yaml

```yaml
services:
  main:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://app:secret@postgres:5432/shop

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: shop
    volumes:
      - ./seed.sql:/docker-entrypoint-initdb.d/seed.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d shop"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Key patterns:
- Primary service is named `main` (required -- Harbor hardcodes this name for exec/upload/download)
- `depends_on` with `condition: service_healthy` ensures Postgres is ready before main starts
- Seed data loaded via Postgres init directory (`/docker-entrypoint-initdb.d/`)
- `seed.sql` is in the `environment/` directory alongside the Dockerfile and docker-compose.yaml

### environment/Dockerfile

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir psycopg2-binary==2.9.10
WORKDIR /app
```

### environment/seed.sql

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(10,2) NOT NULL,
    category TEXT NOT NULL
);
INSERT INTO products (name, price, category) VALUES
    ('Laptop', 999.99, 'electronics'),
    ('Headphones', 79.99, 'electronics'),
    ('Desk Chair', 249.99, 'furniture'),
    ('Notebook', 4.99, 'stationery');
```

### tests/test.sh

```bash
#!/bin/bash
set -uo pipefail

# Install test dependencies
apt-get update && apt-get install -y curl postgresql-client
curl -LsSf https://astral.sh/uv/0.9.7/install.sh | sh
source $HOME/.local/bin/env

# Run the agent's migration (it may have already run, but run again to be safe)
python3 /app/migrate.py 2>/dev/null || true

# Run pytest against the database
uvx --with pytest==8.4.1 --with pytest-json-ctrf==0.3.5 --with psycopg2-binary==2.9.10 \
  pytest --ctrf /logs/verifier/ctrf.json /tests/test_migration.py -rA || true

EXIT_CODE=${PIPESTATUS[0]}
if [ "$EXIT_CODE" -eq 0 ]; then
  echo 1 > /logs/verifier/reward.txt
else
  echo 0 > /logs/verifier/reward.txt
fi
```

Note: `postgresql-client` is installed in test.sh (test dependency), not the Dockerfile. The `PGHOST`, `PGUSER`, `PGPASSWORD`, `PGDATABASE` env vars come from `[verifier].env` in task.toml, so test scripts and psycopg2 can connect to Postgres without hardcoding credentials.

### tests/test_migration.py

```python
import psycopg2
import os

def get_conn():
    return psycopg2.connect(
        host=os.environ["PGHOST"],
        user=os.environ["PGUSER"],
        password=os.environ["PGPASSWORD"],
        dbname=os.environ["PGDATABASE"],
    )

def test_discount_column_exists():
    with get_conn() as conn:
        with conn.cursor() as cur:
            cur.execute("""
                SELECT column_name, data_type
                FROM information_schema.columns
                WHERE table_name = 'products' AND column_name = 'discount_pct'
            """)
            row = cur.fetchone()
            assert row is not None, "discount_pct column missing"
            assert "int" in row[1].lower(), f"Expected integer type, got {row[1]}"

def test_electronics_discount():
    with get_conn() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT discount_pct FROM products WHERE category = 'electronics'")
            rows = cur.fetchall()
            assert all(r[0] == 20 for r in rows), f"Expected 20, got {rows}"

def test_discounted_products_view():
    with get_conn() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM discounted_products ORDER BY id")
            rows = cur.fetchall()
            assert len(rows) == 4

def test_final_price_calculation():
    with get_conn() as conn:
        with conn.cursor() as cur:
            cur.execute("""
                SELECT name, final_price FROM discounted_products
                WHERE name = 'Laptop'
            """)
            row = cur.fetchone()
            assert row is not None
            # 999.99 * (100 - 20) / 100 = 799.992
            assert abs(float(row[1]) - 799.992) < 0.01
```

---

## Example 4: MCP Task with Custom Tools

A task where the agent uses MCP tools provided by a custom server running alongside the environment.

### task.toml

```toml
version = "1.0"

[metadata]
difficulty = "medium"
category = "programming"
tags = ["mcp", "tools", "api"]

[agent]
timeout_sec = 600.0

[verifier]
timeout_sec = 120.0

[environment]
memory_mb = 4096

[[environment.mcp_servers]]
name = "data-tools"
transport = "streamable-http"
url = "http://mcp-server:8000/mcp"
```

Note: The MCP server is configured in task.toml with its name, transport type, and URL. The URL uses the Docker Compose service name `mcp-server` for DNS resolution.

### instruction.md

```markdown
# Inventory Analysis

You have access to an MCP tool server called "data-tools" that provides:
- `get_inventory()` -- Returns a JSON list of inventory items with fields: `sku`, `name`, `quantity`, `price`, `warehouse`
- `get_sales(days: int)` -- Returns sales records for the last N days with fields: `sku`, `units_sold`, `date`

Use these tools to:
1. Identify the top 3 items by revenue (units_sold * price) over the last 30 days
2. Identify items with quantity < 10 that had sales in the last 7 days (restock candidates)

Write the results to `/app/report.json` with this exact schema:
```json
{
  "top_revenue": [
    {"sku": "...", "name": "...", "revenue": 0.0}
  ],
  "restock_candidates": [
    {"sku": "...", "name": "...", "quantity": 0, "recent_sales": 0}
  ]
}
```
```

### environment/docker-compose.yaml

```yaml
services:
  main:
    build: .
    depends_on:
      mcp-server:
        condition: service_healthy

  mcp-server:
    build:
      context: .
      dockerfile: Dockerfile.mcp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### environment/Dockerfile

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir requests==2.32.3
WORKDIR /app
```

### environment/Dockerfile.mcp

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir fastapi==0.115.0 uvicorn==0.34.0 mcp-server-sdk==0.3.0
WORKDIR /server
COPY mcp_server.py /server/mcp_server.py
COPY data/ /server/data/
CMD ["uvicorn", "mcp_server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### tests/test.sh

```bash
#!/bin/bash
set -uo pipefail

apt-get update && apt-get install -y curl
curl -LsSf https://astral.sh/uv/0.9.7/install.sh | sh
source $HOME/.local/bin/env

uvx --with pytest==8.4.1 --with pytest-json-ctrf==0.3.5 \
  pytest --ctrf /logs/verifier/ctrf.json /tests/test_report.py -rA || true

EXIT_CODE=${PIPESTATUS[0]}
if [ "$EXIT_CODE" -eq 0 ]; then
  echo 1 > /logs/verifier/reward.txt
else
  echo 0 > /logs/verifier/reward.txt
fi
```

### tests/test_report.py

```python
import json
import os

def test_report_exists():
    assert os.path.exists("/app/report.json"), "report.json not found"

def test_report_schema():
    with open("/app/report.json") as f:
        data = json.load(f)

    assert "top_revenue" in data
    assert "restock_candidates" in data
    assert len(data["top_revenue"]) == 3

    for item in data["top_revenue"]:
        assert "sku" in item
        assert "name" in item
        assert "revenue" in item
        assert isinstance(item["revenue"], (int, float))

def test_revenue_ordering():
    with open("/app/report.json") as f:
        data = json.load(f)

    revenues = [item["revenue"] for item in data["top_revenue"]]
    assert revenues == sorted(revenues, reverse=True), "Items not sorted by revenue"
```

This example demonstrates: MCP server configuration in task.toml, multi-container setup with a custom MCP server, health checks to ensure the MCP server is ready, and testing the agent's use of external tools.
