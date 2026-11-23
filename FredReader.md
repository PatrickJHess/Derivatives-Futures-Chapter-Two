## Fred Reader

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

