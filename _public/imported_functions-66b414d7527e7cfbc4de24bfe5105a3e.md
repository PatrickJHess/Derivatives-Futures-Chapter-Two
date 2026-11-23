#  Imported Functions




:::{dropdown} Click to see `MASSIVEReader`

```py
import os
import time
import json
import requests
import pandas as pd
import pandas_market_calendars as mcal
from .massive_base import MassiveBase


class MASSIVEReader(MassiveBase):
    def __init__(self, api_key: str = None, key_name: str = "", calls_per_minute: int = 4):
        super().__init__(api_key=api_key, key_name=key_name, calls_per_minute=calls_per_minute)
        
        # 📅 Initialize market calendars once for the whole class
        self.cme_cal = mcal.get_calendar('CME_TradeDate')   # For Futures
        self.nyse_cal = mcal.get_calendar('NYSE') # For Stocks

    def _align_trade_dates(self, start_date: str, end_date: str, market: str = "CME") -> tuple:
        """
        Universal calendar logic to snap weekends and holidays to valid trade dates.
        Prevents phantom cache misses and needless API calls.
        """
        cal = self.cme_cal if market == "CME" else self.nyse_cal
        
        # Snap start date FORWARD to the next valid trading day
        valid_starts = cal.valid_days(start_date, pd.to_datetime(start_date) + pd.Timedelta(days=15))
        aligned_start = valid_starts[0].strftime('%Y-%m-%d')
        
        # Snap end date BACKWARD to the most recent valid trading day
        # 1. Convert input to pandas Timestamp and drop time component
        end_dt = pd.to_datetime(end_date).normalize()
        today = pd.Timestamp.today().normalize()
    
        # 2. Cap at today first (prevents querying future schedule data)
        capped_end_dt = min(end_dt, today)
        
        # 3. Query calendar using capped_end_dt
        valid_ends = cal.valid_days(capped_end_dt - pd.Timedelta(days=15), capped_end_dt)
        aligned_end = valid_ends[-1].strftime('%Y-%m-%d')
        
        return aligned_start, aligned_end

    def _execute_with_cache(self, fetch_callback, ticker, start_date, end_date, resolution):
        """
        Universal caching engine driven directly by the CSV file contents.
        Loads existing data first, checks the date index, and dynamically fetches
        only the missing blocks before or after the cached range.
        """
        safe_ticker = ticker.replace(":", "_")
        filepath = os.path.join(self.cache_dir, f"{safe_ticker}_{resolution}_data.csv")
        
        # 1. LOAD THE EXISTING CACHE (The Single Source of Truth)
        if os.path.exists(filepath):
            try:
                df = pd.read_csv(filepath, index_col=0)
                df.index = pd.to_datetime(df.index, errors='coerce', utc=True)
                df = df[df.index.notnull()]
                if getattr(df.index, 'tz', None) is not None:
                    df.index = df.index.tz_localize(None)
                df = df.sort_index()
            except Exception:
                # If the CSV is corrupt, pretend it doesn't exist to force a fresh pull
                df = pd.DataFrame()
        else:
            df = pd.DataFrame()

        # 2. CALCULATE MISSING DELTAS
        fetch_ranges = []
        
        if df.empty:
            # We have nothing; fetch the whole requested window
            fetch_ranges.append((start_date, end_date))
        else:
            req_start = pd.to_datetime(start_date)
            req_end = pd.to_datetime(end_date)
            c_start = pd.to_datetime(df.index.min().strftime('%Y-%m-%d'))
            c_end = pd.to_datetime(df.index.max().strftime('%Y-%m-%d'))
            
            # Check for missing data BEFORE our cache
            if req_start < c_start:
                # Shift 1 day back to avoid re-downloading c_start
                fetch_end = (c_start - pd.Timedelta(days=1)).strftime('%Y-%m-%d')
                fetch_ranges.append((start_date, fetch_end))
            
            # Check for missing data AFTER our cache
            if req_end > c_end:
                # Shift 1 day forward to avoid re-downloading c_end
                fetch_start = (c_end + pd.Timedelta(days=1)).strftime('%Y-%m-%d')
                
                # GUARDRAIL: Only fetch if there is at least one business day in the gap.
                missing_bdays = len(pd.bdate_range(start=fetch_start, end=end_date))
                
                if missing_bdays > 0:
                    fetch_ranges.append((fetch_start, end_date))
                else:
                    print(f"✅ Gap ({fetch_start} to {end_date}) contains no trading days. Skipping API call.")

        # 3. IF NO DELTAS, WE HAVE A FULL CACHE HIT
        if not fetch_ranges:
            print(f"⚡ Full cache hit for {ticker} ({resolution}). Loaded directly from CSV.")
            return df.loc[start_date:end_date]

        # 4. FETCH ONLY THE MISSING PIECES
        df_list = [df] if not df.empty else []
        
        for f_start, f_end in fetch_ranges:
            print(f"☁️ Downloading {ticker} ({resolution}) via API from {f_start} to {f_end}...")
            self._enforce_speed_limit()

            try:
                new_data = fetch_callback(f_start, f_end)
                if new_data is not None and not new_data.empty:
                    df_list.append(new_data)
                    
            except Exception as e:
                if "429" in str(e):
                    print(f"\n🚨 SDK Server Crash! The API forcefully rejected {ticker}.")
                    print("😴 Forcing a hard 65-second server-reset penalty...")
                    time.sleep(65)
                    print(f"🔄 Attempting {ticker} one final time...")
                    return self._execute_with_cache(fetch_callback, ticker, start_date, end_date, resolution)
                else:
                    print(f"❌ Connection or Parsing error: {e}")
                    return None

        # 5. MERGE, CLEAN, AND SAVE
        if len(df_list) == (1 if not df.empty else 0):
            print(f"⚠️ API returned no new data for requested ranges.")
            return df.loc[start_date:end_date] if not df.empty else None

        combined_df = pd.concat(df_list).drop_duplicates().sort_index()
        
        # Clean up any duplicated dates safely before saving
        combined_df = combined_df[~combined_df.index.duplicated(keep='last')]
        combined_df.to_csv(filepath)

        true_start = combined_df.index.min().strftime('%Y-%m-%d')
        true_end = combined_df.index.max().strftime('%Y-%m-%d')
        print(f"💾 Merged and saved {ticker} to cache. (Total Coverage: {true_start} to {true_end})")

        return combined_df.loc[start_date:end_date]


    # =========================================================================
    # STOCKS & EQUITIES PROVIDER METHODS
    # =========================================================================
    def get_stock_data(self, ticker: str, start_date: str, end_date: str, timespan: str = "day", multiplier: int = 1) -> pd.DataFrame:
        # 🛡️ Clean the dates before anything else!
        if timespan.lower() in ["day", "daily", "session"]:
            start_date, end_date = self._align_trade_dates(start_date, end_date, market="NYSE")

        print(f"📈 Fetching stock data for {ticker} ({multiplier} {timespan})...")

        def _fetch(f_start, f_end):
            aggs = self.client.get_aggs(
                ticker=ticker,
                multiplier=multiplier,
                timespan=timespan,
                limit=50000,
                from_=f_start,
                to=f_end
            )
            df = pd.DataFrame(aggs)
            if not df.empty:
                df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
                df.set_index('timestamp', inplace=True)
                df.index = df.index.tz_localize('UTC').tz_convert('America/New_York').tz_localize(None)
            return df

        resolution_label = f"{multiplier}_{timespan}"

        return self._execute_with_cache(
            fetch_callback=_fetch,
            ticker=ticker,
            start_date=start_date,
            end_date=end_date,
            resolution=resolution_label
        )


    # =========================================================================
    # DIVIDENDS PROVIDER METHOD
    # =========================================================================
    def get_dividend_data(self, ticker: str, start_date: str = "1970-01-01", end_date: str = "2099-12-31") -> pd.DataFrame:
        print(f"💰 Fetching dividend data for {ticker}...")

        def _fetch(f_start, f_end):
            url = "https://api.massive.com/stocks/v1/dividends"
            params = {
                "ticker": ticker,
                "limit": 1000, 
                "sort": "ex_dividend_date.asc",
                "apiKey": self.api_key
            }

            response = requests.get(url, params=params)
            response.raise_for_status()

            data = response.json()
            df = pd.DataFrame(data.get("results", []))

            if not df.empty:
                time_col = 'ex_dividend_date'
                if time_col in df.columns:
                    df[time_col] = pd.to_datetime(df[time_col])
                    df.set_index(time_col, inplace=True)
                    df.index = df.index.normalize()
            return df

        return self._execute_with_cache(
            fetch_callback=_fetch,
            ticker=ticker,
            start_date=start_date,
            end_date=end_date,
            resolution="dividends"
        )


    # =========================================================================
    # FUTURES PROVIDER METHODS
    # =========================================================================
    def get_futures_data(self, contract_symbol: str, start_date: str, end_date: str, timespan: str = "day", multiplier: int = 1) -> pd.DataFrame:
        is_daily_resolution = timespan.lower() in ["day", "daily", "session"]
        
        # 🛡️ Clean the dates before anything else!
        if is_daily_resolution:
            start_date, end_date = self._align_trade_dates(start_date, end_date, market="CME")

        print(f"🚀 Fetching futures data for {contract_symbol} ({multiplier} {timespan})...")

        def _fetch(f_start, f_end):
            url = f"https://api.massive.com/futures/v1/aggs/{contract_symbol}"
            
            # 1. Apply start-date shift ONLY to daily/session data
            if is_daily_resolution:
                massive_timespan = "session"
                api_start = (pd.to_datetime(f_start) - pd.Timedelta(days=1)).strftime('%Y-%m-%d')
            else:
                 massive_timespan = "min" if timespan.lower() == "minute" else timespan.lower()
                 # Intraday bars (1_min, 5_min, 60_min) keep exact start boundary
                 api_start = f_start

            # Guaranteed end-date buffer for both resolutions
            api_end = (pd.to_datetime(f_end) + pd.Timedelta(days=1)).strftime('%Y-%m-%d')
            resolution_string = f"{multiplier}{massive_timespan}"

            params = {
                "resolution": resolution_string,
                "window_start.gte": api_start,
                "window_start.lte": api_end,
                "limit": 50000,
                "sort": "window_start.asc",
                "apiKey": self.api_key
            }

            response = requests.get(url, params=params)
            response.raise_for_status()

            data = response.json()
            df = pd.DataFrame(data.get("results", []))

            if not df.empty:
                time_col = 'timestamp' if 'timestamp' in df.columns else ('window_start' if 'window_start' in df.columns else 't')

                try:
                    df[time_col] = pd.to_datetime(df[time_col], unit='ns')
                except ValueError:
                    df[time_col] = pd.to_datetime(df[time_col])

                df.set_index(time_col, inplace=True)

                # 2. Map evening window_start times (18:00) to the Trade Date ONLY for daily bars
                if timespan in ['day', 'week', 'month', 'session']:
                    # Massive API stamps daily futures bars at session open (the evening before).
                    # Shift all daily/session timestamps forward 1 day to match the Trade Date. 
                    df.index = (df.index + pd.Timedelta(days=1)).normalize()
                else:
                    if getattr(df.index, 'tz', None) is None:
                        df.index = df.index.tz_localize('UTC')
                    df.index = df.index.tz_convert('America/New_York').tz_localize(None)
            
            df = df.rename(columns={'ticker':'Symbol'})
            return df

        resolution_label = f"{multiplier}_{timespan}"

        return self._execute_with_cache(
            fetch_callback=_fetch,
            ticker=contract_symbol,
            start_date=start_date,
            end_date=end_date,
            resolution=resolution_label
        )
```

:::
:::{dropdown} Click to see `FredReader`

```py
import json
import os
import requests
import pandas as pd
import datetime as dt
import pandas_datareader.data as web
from .fred_base import FredBase
from financial_quant.security.credentials import secure_key_setup

class FredReader(FredBase):

    def __init__(
        self,
        api_key=None,
        key_name="fred_key",
        github_raw_base=None,
        cache_dir=None,
        enable_github_repo=False,  # <-- Staging flag: Set to False for now
    ):
        super(FredReader, self).__init__(api_key=api_key, key_name=key_name)
        if cache_dir:
            self.cache_dir = cache_dir

        self.github_raw_base = (
            github_raw_base.rstrip("/") if github_raw_base else None
        )
        self.enable_github_repo = enable_github_repo

    # --- PUBLIC WRAPPER METHOD ---
    def get_series(
        self, series_ids, start_date=None, end_date=None, ttl_days=7
    ):
        """Fetches one or more series and returns them in a single merged DataFrame."""
        if isinstance(series_ids, str):
            series_ids = [series_ids]
        elif not hasattr(series_ids, "__iter__"):
            raise TypeError(
                "series_ids must be a string or an iterable of strings."
            )

        dataframes = []

        for series_id in series_ids:
            print(f"\n☁️--- Processing {series_id} ---")
            df = self._get_single_series(
                series_id,
                start_date=start_date,
                end_date=end_date,
                ttl_days=ttl_days,
            )
            if df is not None and not df.empty:
                dataframes.append(df)
            else:
                print(f"⚠️ Skipping {series_id}: No data was returned.")

        if dataframes:
            if len(dataframes) == 1:
                return dataframes[0]
            print("\n🧩 Merging all series into a single DataFrame...")
            combined_df = pd.concat(dataframes, axis=1, join="outer")
            combined_df.sort_index(inplace=True)
            print("✅ Merge complete!")
            return combined_df
        else:
            print("❌ No data could be retrieved.")
            return None

    # --- CORE WORKHORSE METHOD ---
    def _get_single_series(
        self, series_id, start_date=None, end_date=None, ttl_days=7
    ):
        # Local Parquet & Metadata Paths
        filepath = os.path.abspath(
            os.path.join(self.cache_dir, f"{series_id}_fred.parquet")
        )
        metadata_file = os.path.abspath(
            os.path.join(self.cache_dir, f"{series_id}_metadata.json")
        )

        metadata = {}
        metadata_is_stale = True
        metadata_updated_this_run = False

        # --- NESTED METADATA HELPER ---
        def update_metadata():
            nonlocal metadata, metadata_updated_this_run
            if not self.api_key:
                return True

            meta_url = f"https://api.stlouisfed.org/fred/series?series_id={series_id}&api_key={self.api_key}&file_type=json"
            try:
                meta_data = self._make_api_request(
                    meta_url, series_id=series_id
                )
                if not meta_data or "seriess" not in meta_data:
                    return False

                if len(meta_data["seriess"]) > 0:
                    series_info = meta_data["seriess"][0]
                    metadata["series_inception"] = series_info.get(
                        "observation_start"
                    )
                    metadata["series_last_observed"] = series_info.get(
                        "observation_end"
                    )
                    metadata["title"] = series_info.get("title")
                    metadata["frequency"] = series_info.get("frequency")
                    metadata["last_updated"] = dt.datetime.now(
                        dt.timezone.utc
                    ).isoformat()

                    os.makedirs(self.cache_dir, exist_ok=True)
                    with open(metadata_file, "w") as f:
                        json.dump(metadata, f, indent=4)

                    metadata_updated_this_run = True
                    print("⏳ Metadata synced and timestamp updated.")
                    return True
            except Exception as e:
                print(f"⚠️ Could not update metadata: {e}")
                return False

        # --- 1. READ LOCAL METADATA ---
        if os.path.exists(metadata_file):
            try:
                with open(metadata_file, "r") as f:
                    metadata = json.load(f)
            except (json.JSONDecodeError, ValueError):
                print(
                    f"⚠️ Warning: Corrupt metadata for {series_id}. Resetting..."
                )
                metadata = {}
                try:
                    os.remove(metadata_file)
                except Exception:
                    pass

            if "last_updated" in metadata:
                last_updated = dt.datetime.fromisoformat(
                    metadata["last_updated"]
                )
                if last_updated.tzinfo is None:
                    last_updated = last_updated.replace(tzinfo=dt.timezone.utc)

                days_old = (
                    dt.datetime.now(dt.timezone.utc) - last_updated
                ).days
                if days_old < ttl_days:
                    metadata_is_stale = False
                    print(f"🕒 Metadata is fresh ({days_old} days old).")
                else:
                    print(
                        f"⏳ Metadata is {days_old} days old (TTL: {ttl_days}). It's stale."
                    )

        # --- 2. SMART PING ---
        if not metadata:
            print(f"🆕 First run for {series_id}. Initializing metadata...")
            update_metadata()
        elif (
            self.api_key
            and end_date
            and "series_last_observed" in metadata
            and metadata_is_stale
        ):
            if pd.to_datetime(end_date) > pd.to_datetime(
                metadata["series_last_observed"]
            ):
                print(
                    "🔎 Requested date exceeds known end date. Checking for updates..."
                )
                update_metadata()

        # --- 3. CLAMP DATES ---
        if "series_inception" in metadata and start_date:
            clamped_start = max(
                pd.to_datetime(start_date),
                pd.to_datetime(metadata["series_inception"]),
            )
            if pd.to_datetime(start_date) < clamped_start:
                start_date = clamped_start.strftime("%Y-%m-%d")
                print(
                    f"⚠️ Adjusted start date to First Available Data: {start_date}"
                )

        if "series_last_observed" in metadata and end_date:
            clamped_end = min(
                pd.to_datetime(end_date),
                pd.to_datetime(metadata["series_last_observed"]),
            )
            if pd.to_datetime(end_date) > clamped_end:
                end_date = clamped_end.strftime("%Y-%m-%d")
                print(
                    f"⚠️ FRED has no data past {end_date}. Adjusted request to match."
                )

        # --- 4. CHECK LOCAL CACHE (PARQUET) ---
        if os.path.exists(filepath) and not metadata_is_stale:
            try:
                df = pd.read_parquet(filepath)
                cache_start, cache_end = df.index.min(), df.index.max()

                valid_start = not (
                    start_date
                    and pd.to_datetime(start_date)
                    < cache_start - pd.Timedelta(days=5)
                )
                valid_end = not (
                    end_date and pd.to_datetime(end_date) > cache_end
                )

                if valid_start and valid_end:
                    print(f"✅ Loaded {series_id} from local Parquet cache.")
                    if start_date:
                        df = df[df.index >= pd.to_datetime(start_date)]
                    if end_date:
                        df = df[df.index <= pd.to_datetime(end_date)]
                    return df
            except Exception as e:
                print(
                    f"⚠️ Local cache corrupt or unreadable: {e}. Moving to next tier..."
                )

        # --- 5. CHECK GITHUB STATIC DATA REPO TIER ---
        if self.enable_github_repo and self.github_raw_base:
            github_url = f"{self.github_raw_base}/{series_id}_fred.parquet"
            print(f"🌐 [Staged Tier] Checking GitHub Data Repo: {github_url}...")
            try:
                # Attempt to stream parquet directly from GitHub
                df = pd.read_parquet(github_url)
                print(
                    f"✅ Found {series_id} in GitHub Repo! Caching locally..."
                )

                # Save directly to environment cache
                os.makedirs(self.cache_dir, exist_ok=True)
                df.to_parquet(filepath)

                if start_date:
                    df = df[df.index >= pd.to_datetime(start_date)]
                if end_date:
                    df = df[df.index <= pd.to_datetime(end_date)]
                return df

            except Exception as e:
                # Controlled exception catching during staging
                print(
                    f"ℹ️ GitHub Repo lookup bypassed/failed ({e}). Proceeding to API tier..."
                )
        else:
            print(" Skipping GitHub static repo check (Tier disabled).")

        # --- 6. API FETCH (FRED / WEB) ---
# --- TIER 3: LIVE API FETCH ---
        print(f"☁️ Fetching fresh {series_id} observations from API...")
        try:
            if self.api_key is None:
                df = web.DataReader(
                    series_id, "fred", start=start_date, end=end_date
                )
            else:
                url = f"https://api.stlouisfed.org/fred/series/observations?series_id={series_id}&api_key={self.api_key}&file_type=json"
                if start_date:
                    url += f"&observation_start={start_date}"
                if end_date:
                    url += f"&observation_end={end_date}"

                data = self._make_api_request(url)
                if not data or "observations" not in data:
                    return None

                df = pd.DataFrame(data["observations"])
                df["DATE"] = pd.to_datetime(df["date"])
                df[series_id] = pd.to_numeric(df["value"], errors="coerce")
                df = df.set_index("DATE")
                df = df[[series_id]]

            df.dropna(inplace=True)

            # Save newly fetched API data to local Parquet cache
            os.makedirs(self.cache_dir, exist_ok=True)
            df.to_parquet(filepath)
            print(f"Success! Saved fresh Parquet data to {filepath}.")
            return df

        except Exception as e:
            print(f"❌ An error occurred in API fetch: {e}")
            return None
```
::::
:::{dropdown} Click to see `static_futures_data`

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
::::
:::{dropdown} Click to see `graph_uncertainty_arb`

```py
mport os
import sys
import re
import pandas as pd
import numpy as np
from pathvalidate import sanitize_filepath
from matplotlib import pyplot as plt
import openpyxl
from openpyxl.utils import get_column_letter
from IPython.display import display, Markdown as md
from pathlib import Path
import altair as alt
def graph_uncertainty_arb(df_list, asset):
    """
    Generates a 2-tier dashboard:
    Top Row: Static side-by-side Volume and Closing Price charts.
    Bottom Row: Interactive Overview/Detail chart explicitly for Closing Price.
    """
    # 1. Prepare the combined dataset
    df_combined = pd.concat(df_list).reset_index(names='date')

    price_domain = [df_combined['close'].min() * 0.9, df_combined['close'].max() * 1.1]
    volume_domain = [df_combined['volume'].min() * 0.9, df_combined['volume'].max() * 1.1]

    # Shared base chart ensures both rows get the legend by default
    base_chart = alt.Chart(df_combined).mark_line().encode(
        color=alt.Color('symbol:N', legend=alt.Legend(title="Symbol"))
    )

    # --- TOP ROW: Static Side-by-Side ---
    volume_chart = base_chart.encode(
        x=alt.X('date:T', title='Date'),
        y=alt.Y('volume:Q', title='Volume', scale=alt.Scale(domain=volume_domain))
    ).properties(
        width=300, height=350, title=f'{asset} Trading Volume - Full Timeline'
    )

    price_chart = base_chart.encode(
        x=alt.X('date:T', title='Date'),
        y=alt.Y('close:Q', title='Price', scale=alt.Scale(domain=price_domain))
    ).properties(
        width=300, height=350, title=f'{asset} Closing Prices - Arb Effect'
    )

    top_row = volume_chart | price_chart


    # --- BOTTOM ROW: Interactive Arb Effect (Price Only) ---
    brush = alt.selection_interval(encodings=['x'])

    price_base = base_chart.encode(
        y=alt.Y('close:Q', title='Price', scale=alt.Scale(domain=price_domain))
    )

    # Upper Detailed Price Chart (Keeps the legend from base_chart)
    price_upper = price_base.encode(
        x=alt.X('date:T', scale=alt.Scale(domain=brush), title='')
    ).properties(
        width=650, height=260, title=f'Interactive Detail: {asset} Arb Effect'
    )

    # Lower Overview Price Chart (Legend explicitly hidden)
    price_lower = price_base.encode(
        x=alt.X('date:T', title='Date (Drag to Zoom)'),
        color=alt.Color('symbol:N', legend=None) 
    ).properties(
        width=650, height=150
    ).add_params(
        brush
    )

    bottom_row = price_upper & price_lower


    # --- FINAL DASHBOARD COMBINATION ---
    # We resolve the legend independently so Altair doesn't try to force 
    # the top row and bottom row to share a single, awkwardly placed legend.
    final_dashboard = (top_row & bottom_row).resolve_legend(
        color="independent"
    )

    return final_dashboard
```
::::
:::{dropdown} Click to see `one_y_axis`

```py
import os
import sys
import re
import pandas as pd
import numpy as np
from pathvalidate import sanitize_filepath
from matplotlib import pyplot as plt
import openpyxl
from openpyxl.utils import get_column_letter
from IPython.display import display, Markdown as md
from pathlib import Path
import altair as alt
def one_y_axis(x_data, y_data_list, title="", xlabel="", ylabel="", 
                     series_labels=None, markers=None, colors=None,
                     figure_size=(10, 6), y_limits=None, 
                     save_config=None, fill_config=None):
    """
    Plots data on a single y-axis.

    Args:
        x_data (array-like): Data for the x-axis.
        y_data_list (list of array-like): A list of datasets for the y-axis.
        title (str): The title of the graph.
        xlabel (str): The label for the x-axis.
        ylabel (str): The label for the y-axis.
        series_labels (list of str, optional): Identifiers for each data series. 
        markers (list of str, optional): The markers to use for each series.
        colors (list of str, optional): Colors for each series.
        figure_size (tuple): The width and height of the figure in inches.
        y_limits (tuple, optional): The (min, max) values for the y-axis.
        save_config (dict, optional): Config for saving the file. Keys: 'volume', 'chapter', 'file_name'.
        fill_config (dict, optional): Config for filling areas. Keys: 'Between' (list of 1 or 2 indices),
                                      'Start', 'End', 'Colors', 'Labels', 'Alpha'.
    """


    num_series = len(y_data_list)

    # --- Smart Defaults (Frictionless Inputs) ---
    series_labels = series_labels or [f"Series {i+1}" for i in range(num_series)]
    markers = markers or [""] * num_series
    colors = colors or plt.cm.viridis_r(np.linspace(0, 1, num_series))

    # Input Validation
    if not (len(series_labels) == len(markers) == len(colors) == num_series):
        raise ValueError("Lengths of 'series_labels', 'markers', and 'colors' must match 'y_data_list'.")

    # --- Plotting Setup (Protects Global State) ---
    with plt.style.context('ggplot'):
        fig, ax = plt.subplots(figsize=figure_size)
        fig.suptitle(title)

        # --- Pythonic Loop ---
        for y_data, label, marker, color in zip(y_data_list, series_labels, markers, colors):
            ax.plot(x_data, y_data, label=label, marker=marker, color=color)

        # --- Implement the Missing Fill Feature ---
        if fill_config:
            indices = fill_config.get('Between', [0])
            y1 = y_data_list[indices[0]]
            y2 = y_data_list[indices[1]] if len(indices) > 1 else np.zeros_like(y1)
            
            start, end = fill_config.get('Start', 0), fill_config.get('End', len(x_data))
            
            ax.fill_between(
                x_data[start:end], y1[start:end], y2[start:end],
                color=fill_config.get('Colors', 'gray'),
                alpha=fill_config.get('Alpha', 0.3),
                label=fill_config.get('Labels', None)
            )

        # --- Final Touches ---
        if y_limits:
            ax.set_ylim(y_limits)
        ax.set_xlabel(xlabel)
        ax.set_ylabel(ylabel)
        ax.legend()
        plt.tight_layout()

        # --- Save Figure ---
        if save_config:
            # Assuming save_results is imported/defined elsewhere in your script
            path = save_results(save_config=save_config)
            if path:
                plt.savefig(path, dpi=300, bbox_inches='tight')

        plt.show()
```
::::