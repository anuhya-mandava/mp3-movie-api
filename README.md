# MP3: Movie API with SQLite and Caching

This FastAPI application stores a movie collection in SQLite, queries it with SQL, cleans it with the reusable MP1 pipeline in `data_pipeline/`, and caches repeated retrievals. `POST /upload` checks that a CSV has exactly the columns `title`, `release_year`, `rating`, and `genre`, and `src/storage.py` then replaces the stored collection in one transaction, rolling back if anything fails. `src/database.py` creates the `movies` table in `data/movies.db`, so the data persists when the server restarts. `GET /movies` returns up to `limit` movies with the total count, and `GET /movies/search` uses SQL to filter by minimum rating, return only the requested columns, and sort by rating, with values passed through `?` placeholders and column names checked against an allowed list. `POST /movies/process` loads the stored movies, validates and cleans them using a JSON configuration defined in `src/models.py`, and saves the result back to SQLite. `src/cache.py` caches `GET /movies` responses in FakeRedis for 90 seconds per `limit`, and the cache is cleared only after an upload or processing step saves successfully, so clients never receive outdated data. Every request is logged by `src/middleware.py` with a request ID, status code, and duration.

## Commands

Set up and activate the environment (Windows PowerShell):

```powershell
python -m venv venv
venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

Start the API, then open http://127.0.0.1:8000/docs:

```powershell
python -m uvicorn main:app --reload
```

Run the tests:

```powershell
python -m pytest tests -v
```

## Sample processing request

After uploading `fixtures/sample_movies.csv`, send this body to `POST /movies/process`:

```json
{
  "validation": {
    "required_columns": ["title", "release_year", "rating", "genre"],
    "numeric_columns": ["release_year", "rating"]
  },
  "processing": {
    "remove_duplicates": true,
    "missing": {"enabled": true, "axis": "rows"},
    "outliers": {
      "enabled": true,
      "columns": ["rating"],
      "method": "iqr",
      "threshold": 1.5
    }
  }
}
```

Expected result:

```json
{
  "validation_rows_removed": 0,
  "cleaning_report": {
    "rows_before": 20,
    "rows_after": 16,
    "rows_removed": 4,
    "columns_before": 4,
    "columns_after": 4,
    "columns_removed": 0
  }
}
```