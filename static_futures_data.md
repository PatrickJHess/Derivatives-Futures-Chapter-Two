## Static Futures Data

```py
import sys
import requests
import time
import pandas as pd
from io import BytesIO
from pathlib import Path

# 1. Define repo configuration
GITHUB_RAW_BASE_URL = "https://raw.githubusercontent.com/PatrickJHess/Static_Data_Repo/main/Futures"

# 2. Environment-Aware Cache Routing with Google Drive
if 'google.colab' in sys.modules:
    from google.colab import drive
    # Mount Google Drive (will prompt for authorization if not already mounted)
    drive.mount('/content/drive')
    # Route cache to persistent Google Drive folder
    DEFAULT_CACHE_DIR = Path("/content/drive/MyDrive/Futures_Static")
else:
    # Use standard local directory for Jupyter/VSCode
    DEFAULT_CACHE_DIR = Path.cwd() / "cache" / "Futures_Static"

def static_futures_data(file_name, cache_dir=DEFAULT_CACHE_DIR, force_update=False, cache_ttl_hours=24):
    
    # 3. Standardize Parquet Filename and cache
    filename = f"{file_name}.parquet"
    # Ensure local cache folder exists
    cache_path = Path(cache_dir)
    cache_path.mkdir(parents=True, exist_ok=True)
    file_path = cache_path / filename

    # 4. Check how old the cache file is (if it exists)
    is_cache_stale = False
    if file_path.exists():
        # Last modification time number of seconds elapsed since the Unix epoch)
        file_age_seconds = time.time() - file_path.stat().st_mtime

        # Check if ttl is past due
        if file_age_seconds > (cache_ttl_hours * 3600):
            is_cache_stale = True

    # 5. Tier 1: Load Local Cache (If exists, not forced, and not stale)
    if file_path.exists() and not force_update and not is_cache_stale:
        print(f"Loading Parquet data from local cache: {filename}")
        return pd.read_parquet(file_path).set_index('Date')

    # 6. Tier 2: Cache is stale or forced update from github repo
    else:
        # Determine the reason for logging
        if force_update:
            reason = "Force update requested"
        elif is_cache_stale:
            reason = f"Cache older than {cache_ttl_hours} hours"
        else:
            reason = "Not found in cache"

        print(f"Fetching Parquet data from GitHub ({reason}): {filename}")

        github_url = f"{GITHUB_RAW_BASE_URL}/{filename}"
        try:
            response = requests.get(github_url, timeout=10)
            response.raise_for_status()

            # Read into Pandas
            df = pd.read_parquet(BytesIO(response.content))

            # Save fresh copy to cache (this automatically resets the file's timestamp)
            df.to_parquet(file_path, index=False)

            # Set index and return
            return df.set_index('Date')

        except Exception as e:
            print(f"Error loading Parquet data: {e}")
            # Fallback: if GitHub fails but we have a stale cache, use it anyway
            if file_path.exists():
                print("Falling back to stale local cache due to download error.")
                return pd.read_parquet(file_path).set_index('Date')
            return None
```
