Quick Start

from ledger_recon import ReconciliationEngine, DataLoader

# Load data sources
loader = DataLoader()
ledger = loader.load_csv('internal_ledger_sep2021.csv')
bank_stmt = loader.load_mt940('bank_statement_sep2021.mt940')

# Configure reconciliation rules
engine = ReconciliationEngine()
engine.configure({
    'amount_tolerance': 0.01,
    'date_tolerance_days': 2,
    'match_on_reference': True,
    'fuzzy_match_threshold': 0.85
})

# Run reconciliation
results = engine.reconcile(ledger, bank_stmt)

print(f"Matched: {results.matched_count}")
print(f"Ledger Only: {results.ledger_only_count}")
print(f"Bank Only: {results.bank_only_count}")
print(f"Partial Matches: {results.partial_match_count}")

# Export results
results.export_excel('reconciliation_report_sep2021.xlsx')
