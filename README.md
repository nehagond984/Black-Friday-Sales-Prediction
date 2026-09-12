# Jr Data Engineer Assignment — Sales Data Pipeline

An end-to-end data pipeline that cleans raw sales transaction data, builds
sales-analysis tables, and loads everything into PostgreSQL running in Docker.

## Project Structure

```
project/
├── data/                   # raw + generated CSVs (generated files are gitignored)
│   └── assignment_dataset.csv
├── src/
│   ├── clean.py            # Step 1: cleans the raw dataset
│   ├── transform.py        # Step 2: builds daily/monthly/top-items tables
│   ├── load_to_db.py       # Step 3: loads tables into PostgreSQL
│   └── main.py             # runs all three steps in sequence
├── docker/
│   └── docker-compose.yml  # PostgreSQL service
├── requirements.txt
├── .gitignore
└── README.md
```

## What the Pipeline Does

1. **Clean** (`src/clean.py`)
   - Normalizes inconsistent date formats
   - Removes exact duplicate rows
   - Drops rows missing critical fields (transaction id, customer id, quantity, price)
   - Fills non-critical nulls (region, category, discount)
   - Removes invalid rows (zero/negative quantity or price)
   - Recomputes `total_amount` so it's always consistent with `quantity * unit_price - discount`

2. **Transform** (`src/transform.py`) — produces three analysis tables:
   - `daily_sales.csv` — revenue, quantity, and transaction count per day
   - `monthly_sales.csv` — revenue, quantity, and transaction count per month
   - `top_items.csv` — items ranked by total revenue, descending

3. **Load** (`src/load_to_db.py`) — loads `cleaned_sales`, `daily_sales`,
   `monthly_sales`, and `top_items` into PostgreSQL tables.

## Setup

### Prerequisites
- Python 3.10+
- Docker & Docker Compose

### 1. Clone and install dependencies
```bash
git clone <your-repo-url>
cd project
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Start PostgreSQL
```bash
cd docker
docker compose up -d
cd ..
```

This starts a `postgres:16-alpine` container on `localhost:5432` with:
- Database: `sales_db`
- User: `sales_user`
- Password: `sales_pass`

Check it's healthy:
```bash
docker compose -f docker/docker-compose.yml ps
```

### 3. Place your raw data
Put your source file at `data/assignment_dataset.csv` (a sample is already
included so the pipeline can be run and demoed out of the box).

## Running the Pipeline

Run everything in one go:
```bash
cd src
python main.py
```

Or run each stage individually:
```bash
python src/clean.py --input data/assignment_dataset.csv --output data/cleaned_sales.csv
python src/transform.py --input data/cleaned_sales.csv --outdir data
python src/load_to_db.py
```

## Verifying the Data in PostgreSQL

```bash
docker exec -it sales_postgres psql -U sales_user -d sales_db
```

```sql
\dt
SELECT * FROM daily_sales ORDER BY sale_date;
SELECT * FROM monthly_sales ORDER BY sale_month;
SELECT * FROM top_items ORDER BY rank LIMIT 10;
```

## Configuration

`load_to_db.py` reads connection settings from environment variables
(defaults match `docker-compose.yml`):

| Variable            | Default      |
|---------------------|--------------|
| `POSTGRES_HOST`     | `localhost`  |
| `POSTGRES_PORT`     | `5432`       |
| `POSTGRES_DB`       | `sales_db`   |
| `POSTGRES_USER`     | `sales_user` |
| `POSTGRES_PASSWORD` | `sales_pass` |

## Notes

- The sample `data/assignment_dataset.csv` included in this repo is
  synthetically generated to match the expected schema and includes
  intentional messiness (nulls, duplicates, invalid quantities/prices,
  inconsistent date formats) so the cleaning logic is exercised end-to-end.
  Replace it with the real dataset to run against actual data — no code
  changes are required.
