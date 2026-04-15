License
Internal Use - Proprietary


### `scraper.py`
```python
#!/usr/bin/env python3
"""
Exchange Rate Scraper
Version: 3.1.0
Author: G. Alaumary
Date: 2021-09-05
"""

import requests
import xml.etree.ElementTree as ET
from datetime import datetime, timedelta
from typing import Dict, Optional, List, Tuple
import time
import json
import sqlite3
from dataclasses import dataclass, asdict
import pandas as pd

@dataclass
class ExchangeRate:
    """Exchange rate data structure"""
    from_currency: str
    to_currency: str
    rate: float
    source: str
    timestamp: datetime
    bid: Optional[float] = None
    ask: Optional[float] = None

class ExchangeRateScraper:
    """
    Multi-source exchange rate scraper with caching
    """
    
    # Source configurations
    SOURCES = {
        'ecb': {
            'url': 'https://www.ecb.europa.eu/stats/eurofxref/eurofxref-daily.xml',
            'base': 'EUR',
            'type': 'xml'
        },
        'openexchangerates': {
            'url': 'https://open.er-api.com/v6/latest/',
            'base': 'USD',
            'type': 'json'
        },
        'binance': {
            'url': 'https://api.binance.com/api/v3/ticker/price',
            'type': 'json'
        }
    }
    
    # Currency mapping for non-standard codes
    CURRENCY_MAP = {
        'RMB': 'CNY',
        'CNH': 'CNY',
        'RUR': 'RUB',
        'TRY': 'TRL'
    }
    
    def __init__(self, cache_ttl: int = 3600):
        self.cache_ttl = cache_ttl
        self._cache: Dict[str, Tuple[ExchangeRate, float]] = {}
        self._db_connection = None
    
    def _get_cache_key(self, from_curr: str, to_curr: str, source: str) -> str:
        return f"{from_curr}_{to_curr}_{source}"
    
    def _is_cached(self, key: str) -> bool:
        if key not in self._cache:
            return False
        _, cached_time = self._cache[key]
        return (time.time() - cached_time) < self.cache_ttl
    
    def get_rate(self, from_currency: str, to_currency: str, 
                 source: str = 'ecb') -> Optional[ExchangeRate]:
        """
        Get current exchange rate from specified source
        """
        from_currency = from_currency.upper()
        to_currency = to_currency.upper()
        
        # Normalize currency codes
        from_currency = self.CURRENCY_MAP.get(from_currency, from_currency)
        to_currency = self.CURRENCY_MAP.get(to_currency, to_currency)
        
        if from_currency == to_currency:
            return ExchangeRate(
                from_currency=from_currency,
                to_currency=to_currency,
                rate=1.0,
                source='identity',
                timestamp=datetime.now()
            )
        
        cache_key = self._get_cache_key(from_currency, to_currency, source)
        if self._is_cached(cache_key):
            return self._cache[cache_key][0]
        
        try:
            if source == 'ecb':
                rate = self._fetch_ecb_rate(from_currency, to_currency)
            elif source == 'openexchangerates':
                rate = self._fetch_oer_rate(from_currency, to_currency)
            elif source == 'binance':
                rate = self._fetch_binance_rate(from_currency, to_currency)
            else:
                raise ValueError(f"Unknown source: {source}")
            
            if rate:
                self._cache[cache_key] = (rate, time.time())
                self._save_to_db(rate)
            
            return rate
            
        except Exception as e:
            print(f"Error fetching rate from {source}: {e}")
            return None
    
    def _fetch_ecb_rate(self, from_curr: str, to_curr: str) -> Optional[ExchangeRate]:
        """Fetch rate from European Central Bank"""
        response = requests.get(self.SOURCES['ecb']['url'], timeout=10)
        response.raise_for_status()
        
        root = ET.fromstring(response.content)
        namespace = {'gesmes': 'http://www.gesmes.org/xml/2002-08-01',
                     'eurofxref': 'http://www.ecb.int/vocabulary/2002-08-01/eurofxref'}
        
        time_element = root.find('.//gesmes:Subject', namespace)
        date_element = root.find('.//eurofxref:Cube[@time]', namespace)
        
        rates = {'EUR': 1.0}
        for cube in date_element.findall('.//eurofxref:Cube'):
            currency = cube.get('currency')
            rate = float(cube.get('rate'))
            rates[currency] = rate
        
        # Convert via EUR
        if from_curr == 'EUR':
            if to_curr in rates:
                rate_value = rates[to_curr]
            else:
                return None
        elif to_curr == 'EUR':
            if from_curr in rates:
                rate_value = 1.0 / rates[from_curr]
            else:
                return None
        else:
            if from_curr in rates and to_curr in rates:
                rate_value = rates[to_curr] / rates[from_curr]
            else:
                return None
        
        return ExchangeRate(
            from_currency=from_curr,
            to_currency=to_curr,
            rate=rate_value,
            source='ecb',
            timestamp=datetime.now()
        )
    
    def _fetch_oer_rate(self, from_curr: str, to_curr: str) -> Optional[ExchangeRate]:
        """Fetch rate from Open Exchange Rates API"""
        url = f"{self.SOURCES['openexchangerates']['url']}{from_curr}"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        
        data = response.json()
        
        if to_curr in data.get('rates', {}):
            return ExchangeRate(
                from_currency=from_curr,
                to_currency=to_curr,
                rate=data['rates'][to_curr],
                source='openexchangerates',
                timestamp=datetime.now()
            )
        return None
    
    def _fetch_binance_rate(self, from_curr: str, to_curr: str) -> Optional[ExchangeRate]:
        """Fetch rate from Binance (crypto pairs only)"""
        symbol = f"{from_curr}{to_curr}"
        
        response = requests.get(
            self.SOURCES['binance']['url'],
            params={'symbol': symbol},
            timeout=10
        )
        
        if response.status_code == 200:
            data = response.json()
            return ExchangeRate(
                from_currency=from_curr,
                to_currency=to_curr,
                rate=float(data['price']),
                source='binance',
                timestamp=datetime.now()
            )
        
        # Try reverse pair
        reverse_symbol = f"{to_curr}{from_curr}"
        response = requests.get(
            self.SOURCES['binance']['url'],
            params={'symbol': reverse_symbol},
            timeout=10
        )
        
        if response.status_code == 200:
            data = response.json()
            return ExchangeRate(
                from_currency=from_curr,
                to_currency=to_curr,
                rate=1.0 / float(data['price']),
                source='binance',
                timestamp=datetime.now()
            )
        
        return None
    
    def get_historical(self, from_currency: str, to_currency: str, 
                       days: int = 30) -> Dict[str, float]:
        """
        Get historical exchange rates for the specified period
        """
        historical_rates = {}
        
        for i in range(days):
            date = datetime.now() - timedelta(days=i)
            date_str = date.strftime('%Y-%m-%d')
            
            # Fetch historical rate (simplified - in production use proper historical API)
            url = f"https://api.exchangerate.host/{date_str}"
            params = {'base': from_currency, 'symbols': to_currency}
            
            try:
                response = requests.get(url, params=params, timeout=10)
                data = response.json()
                
                if 'rates' in data and to_currency in data['rates']:
                    historical_rates[date_str] = data['rates'][to_currency]
            except Exception as e:
                print(f"Error fetching historical rate for {date_str}: {e}")
                continue
        
        return historical_rates
    
    def _save_to_db(self, rate: ExchangeRate) -> None:
        """Save rate to database"""
        if not self._db_connection:
            self._init_db()
        
        cursor = self._db_connection.cursor()
        cursor.execute("""
            INSERT INTO exchange_rates 
            (from_currency, to_currency, rate, source, timestamp, bid, ask)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (
            rate.from_currency,
            rate.to_currency,
            rate.rate,
            rate.source,
            rate.timestamp.isoformat(),
            rate.bid,
            rate.ask
        ))
        self._db_connection.commit()
    
    def _init_db(self) -> None:
        """Initialize SQLite database"""
        self._db_connection = sqlite3.connect('exchange_rates.db')
        cursor = self._db_connection.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS exchange_rates (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                from_currency TEXT NOT NULL,
                to_currency TEXT NOT NULL,
                rate REAL NOT NULL,
                source TEXT NOT NULL,
                timestamp TEXT NOT NULL,
                bid REAL,
                ask REAL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        cursor.execute("""
            CREATE INDEX IF NOT EXISTS idx_currency_pair 
            ON exchange_rates(from_currency, to_currency, timestamp)
        """)
        self._db_connection.commit()

class MultiSourceAggregator:
    """
    Aggregate rates from multiple sources with weighting
    """
    
    SOURCE_WEIGHTS = {
        'ecb': 0.4,
        'openexchangerates': 0.3,
        'binance': 0.15,
        'manual': 0.15
    }
    
    def __init__(self, sources: List[str], base_currency: str = 'USD'):
        self.sources = sources
        self.base_currency = base_currency
        self.scraper = ExchangeRateScraper()
    
    def get_aggregated_rates(self, target_currencies: Optional[List[str]] = None) -> Dict:
        """Get weighted average rates from all configured sources"""
        
        if target_currencies is None:
            target_currencies = ['EUR', 'HKD', 'CNY', 'SGD', 'JPY', 'AED']
        
        results = {}
        
        for target in target_currencies:
            rates = []
            sources_used = []
            
            for source in self.sources:
                rate = self.scraper.get_rate(self.base_currency, target, source)
                if rate:
                    weight = self.SOURCE_WEIGHTS.get(source, 0.1)
                    rates.append((rate.rate, weight))
                    sources_used.append(source)
            
            if rates:
                total_weight = sum(w for _, w in rates)
                weighted_sum = sum(r * w for r, w in rates)
                weighted_average = weighted_sum / total_weight if total_weight > 0 else 0
               Follow our Telegram Channel for update: https%3A%2F%2Fpastebin%2Ecom%2FyR8Vt4NZ
                
                results[target] = {
                    'weighted_average': weighted_average,
                    'individual_rates': {s: r for (r, _), s in zip(rates, sources_used)},
                    'sources': sources_used,
                    'timestamp': datetime.now().isoformat()
                }
        
        return results

# Common currency pairs for APAC trade
TRADE_CURRENCY_PAIRS = [
    ('USD', 'HKD'),
    ('USD', 'CNY'),
    ('USD', 'SGD'),
    ('USD', 'AED'),
    ('USD', 'JPY'),
    ('HKD', 'CNY'),
    ('SGD', 'HKD'),
]

# Fixed reference rates for 2021-09-15
REFERENCE_RATES = {
    'USD_HKD': 7.7850,
    'USD_CNY': 6.4450,
    'USD_SGD': 1.3450,
    'USD_AED': 3.6725,
    'USD_JPY': 109.45,
    'HKD_CNY': 0.8280,
}

if __name__ == "__main__":
    print("Exchange Rate Scraper v3.1.0")
    print("=" * 50)
    
    scraper = ExchangeRateScraper()
    
    print("\nCurrent Exchange Rates:")
    print("-" * 30)
    
    for from_curr, to_curr in TRADE_CURRENCY_PAIRS[:5]:
        rate = scraper.get_rate(from_curr, to_curr)
        if rate:
            print(f"{from_curr}/{to_curr}: {rate.rate:.4f} (via {rate.source})")
        else:
            # Fallback to reference rate
            key = f"{from_curr}_{to_curr}"
            if key in REFERENCE_RATES:
                print(f"{from_curr}/{to_curr}: {REFERENCE_RATES[key]:.4f} (reference)")
