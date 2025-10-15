# Market-sentiment-dashboard-
import streamlit as st
import pandas as pd
import requests
from bs4 import BeautifulSoup
import yfinance as yf
import datetime
import time
import json

# Securely store API keys (use environment variables in production)
FRED_API_KEY = '9403dc45251789895fc28b03a512a08e'  # Provided key
CFTC_TOKEN = 'YOUR_CFTC_TOKEN'  # Replace with valid token from CFTC

# Function to fetch FRED data with YoY calculation
def fetch_fred_data(series_id, api_key, yoy=False):
    try:
        # Fetch latest observation
        url = f"https://api.stlouisfed.org/fred/series/observations?series_id={series_id}&api_key={api_key}&file_type=json&sort_order=desc&limit=1"
        response = requests.get(url)
        response.raise_for_status()
        data = response.json()['observations'][0]
        latest_value = float(data['value'])
        latest_date = data['date']

        if yoy:
            # Fetch value from one year ago
            one_year_ago = (datetime.datetime.strptime(latest_date, '%Y-%m-%d') - datetime.timedelta(days=365)).strftime('%Y-%m-%d')
            url_yoy = f"https://api.stlouisfed.org/fred/series/observations?series_id={series_id}&api_key={api_key}&file_type=json&from={one_year_ago}&to={one_year_ago}&limit=1"
            response_yoy = requests.get(url_yoy)
            response_yoy.raise_for_status()
            past_data = response_yoy.json()['observations']
            if past_data:
                past_value = float(past_data[0]['value'])
                yoy_change = ((latest_value - past_value) / past_value * 100) if past_value != 0 else None
                return round(yoy_change, 2), latest_date
        return latest_value, latest_date
    except Exception as e:
        st.error(f"Error fetching FRED data for {series_id}: {e}")
        return None, None

# Fetch VIX from yfinance
def fetch_vix():
    try:
        vix = yf.Ticker("^VIX")
        return round(vix.history(period="1d")['Close'].iloc[-1], 2)
    except Exception as e:
        st.error(f"Error fetching VIX: {e}")
        return None

# Scrape CNN Fear & Greed
def fetch_fear_greed():
    try:
        url = "https://production.dataviz.cnn.io/index/fearandgreed/graphdata"
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        data = response.json()
        return round(data['fear_and_greed']['score'], 2)
    except Exception as e:
        st.error(f"Error fetching Fear & Greed: {e}")
        return None

# Scrape AAII Sentiment
def fetch_aaii_sentiment():
    try:
        url = "https://www.aaii.com/sentimentsurvey/sent_results"
        headers = {'User-Agent': 'Mozilla/5.0'}
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        soup = BeautifulSoup(response.text, 'html.parser')
        table = soup.find('table')
        if table:
            rows = table.find_all('tr')
            latest = rows[1].find_all('td')  # Adjust if structure changes
            return (
                round(float(latest[1].text.strip('%')), 2),
                round(float(latest[2].text.strip('%')), 2),
                round(float(latest[3].text.strip('%')), 2)
            )
        return None, None, None
    except Exception as e:
        st.error(f"Error fetching AAII Sentiment: {e}")
        return None, None, None

# Fetch CME FedWatch (placeholder; needs proper parsing)
def fetch_cme_fedwatch():
    try:
        url = "https://www.cmegroup.com/markets/interest-rates/cme-fedwatch-tool.html"
        headers = {'User-Agent': 'Mozilla/5.0'}
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        soup = BeautifulSoup(response.text, 'html.parser')
        # Adjust selector based on actual page structure
        prob = soup.find(text=lambda t: "25 basis points" in t.lower() if t else False)
        return 96 if prob else None  # Replace with real parsing logic
    except Exception as e:
        st.error(f"Error fetching CME FedWatch: {e}")
        return None

# Fetch COT report
def fetch_cot(token):
    try:
        url = "https://publicreporting.cftc.gov/api/v1/cot/year?reportType=ALL"
        headers = {'Authorization': f'Bearer {token}'}
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        data = response.json()
        return data  # Process specific fields as needed
    except Exception as e:
        st.error(f"Error fetching COT data: {e}")
        return None

# Fetch GPRI
def fetch_gpri():
    try:
        url = "https://www.matteoiacoviello.com/gpr_files/data_gpr_export.csv"
        df = pd.read_csv(url)
        return round(df['GPR'].iloc[-1], 2)
    except Exception as e:
        st.error(f"Error fetching GPRI: {e}")
        return None

# Fetch GDPNow
def fetch_gdpnow():
    try:
        url = "https://www.atlantafed.org/cqer/research/gdpnow"
        headers = {'User-Agent': 'Mozilla/5.0'}
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        soup = BeautifulSoup(response.text, 'html.parser')
        # Adjust selector based on actual page structure
        estimate = soup.find('div', class_='gdpnow-estimate')  # Placeholder; inspect page
        return float(estimate.text.strip('%')) if estimate else None
    except Exception as e:
        st.error(f"Error fetching GDPNow: {e}")
        return None

# Calculate NFP change
def fetch_nfp_change(api_key):
    try:
        url = f"https://api.stlouisfed.org/fred/series/observations?series_id=PAYEMS&api_key={api_key}&file_type=json&sort_order=desc&limit=2"
        response = requests.get(url)
        response.raise_for_status()
        data = response.json()['observations']
        latest = float(data[0]['value'])
        previous = float(data[1]['value'])
        return int(latest - previous)
    except Exception as e:
        st.error(f"Error fetching NFP change: {e}")
        return None

# Main dashboard
st.title("Market Sentiment & Fundamental Indicators Dashboard")
st.write(f"Data auto-fetches on load. Click 'Refresh' for updates. As of: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

if st.button("Refresh Data"):
    st.rerun()

# Fetch data
cpi, cpi_date = fetch_fred_data('CPIAUCSL', FRED_API_KEY, yoy=True)
core_cpi, _ = fetch_fred_data('CPILFESL', FRED_API_KEY, yoy=True)
ppi, _ = fetch_fred_data('PPIACO', FRED_API_KEY, yoy=True)
core_ppi, _ = fetch_fred_data('PCUOMFGOMFG', FRED_API_KEY, yoy=True)
nfp_change = fetch_nfp_change(FRED_API_KEY)
unemp, _ = fetch_fred_data('UNRATE', FRED_API_KEY)
gdp, _ = fetch_fred_data('GDP', FRED_API_KEY)
gdpnow = fetch_gdpnow()
vix = fetch_vix()
fear_greed = fetch_fear_greed()
aaii_bull, aaii_bear, aaii_neut = fetch_aaii_sentiment()
cme_prob = fetch_cme_fedwatch()
gpri = fetch_gpri()
cot = fetch_cot(CFTC_TOKEN)

# Build DataFrame
data = {
    'Indicator': ['CPI (YoY %)', 'Core CPI (YoY %)', 'PPI (YoY %)', 'Core PPI (YoY %)', 'NFP (Jobs Added)', 'Unemployment Rate (%)',
                  'GDP (Latest Q %)', 'GDPNow', 'VIX', 'CNN Fear & Greed', 'AAII Bullish (%)', 'AAII Bearish (%)', 'AAII Neutral (%)',
                  'CME FedWatch (25bp Cut Prob)', 'GPRI', 'COT Report'],
    'Value': [cpi, core_cpi, ppi, core_ppi, nfp_change, unemp, gdp, gdpnow, vix, fear_greed, aaii_bull, aaii_bear, aaii_neut, cme_prob, gpri, cot]
}
df = pd.DataFrame(data)

st.dataframe(df)

# Sample Sentiment Score (placeholder; customize as needed)
def compute_sentiment_score(vix, fear_greed, aaii_bull, aaii_bear):
    if all(x is not None for x in [vix, fear_greed, aaii_bull, aaii_bear]):
        # Normalize and weight indicators (example logic)
        vix_score = (50 - vix) / 50 * 100  # Lower VIX = more bullish
        fear_greed_score = fear_greed  # Already 0-100
        bull_bear_ratio = aaii_bull / (aaii_bear + 1) * 50  # Avoid division by zero
        return round((vix_score + fear_greed_score + bull_bear_ratio) / 3, 2)
    return None

sentiment_score = compute_sentiment_score(vix, fear_greed, aaii_bull, aaii_bear)
st.metric("Overall Sentiment Score", f"{sentiment_score}%")

# Charts
st.subheader("VIX Over Time (1 Month)")
vix_hist = yf.Ticker("^VIX").history(period="1mo")
st.line_chart(vix_hist['Close'])

# Notes
st.write("Notes: Web scraping may break if site structures change. Use official APIs where possible. CFTC API requires a valid token. For RRI/GPRI, consider caching data locally. Host with cron for auto-updates.")
