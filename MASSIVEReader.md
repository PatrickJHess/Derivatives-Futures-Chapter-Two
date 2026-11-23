
## Massive Reader

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
