Configuration
Create config.yaml:

sources:
  ecb:
    enabled: true
    url: https://www.ecb.europa.eu/stats/eurofxref/eurofxref-daily.xml
    update_time: "16:00 CET"
  
  boc:
    enabled: true
    url: https://www.bankofchina.com/sourcedb/whpj/enindex_1619.html
    currencies: [CNY, USD, HKD, EUR, JPY]
  
  binance:
    enabled: true
    api_url: https://api.binance.com/api/v3/ticker/price
    symbols: [BTCHKD, ETHHKD, USDTBUSD]
  
  hkma:
    enabled: true
    url: https://www.hkma.gov.hk/eng/market-data-and-statistics/

storage:
  type: postgresql
  connection: postgresql://user:pass@localhost:5432/fx_rates
  
alerts:
  threshold_percent: 2.5
  notification_email: alerts@example.com
