API Reference
Bitcoin Methods
getAddress(address) - Get address balance and stats

getTransactions(address, limit) - Get transaction history

getTransaction(txid) - Get transaction details

getUTXOs(address) - Get unspent transaction outputs

Ethereum Methods
getBalance(address) - Get ETH balance

getTokenBalance(address, contract) - Get ERC20 token balance

getTransactions(address, limit) - Get transaction history

getTransaction(txid) - Get transaction details

getContractABI(address) - Get verified contract ABI

TRON Methods
getAccount(address) - Get account details

getTransactions(address, limit) - Get transaction history

getTRC20Balance(address, contract) - Get TRC20 token balance


Configuration

{
  "networks": {
    "ethereum": {
      "apiKey": "YOUR_KEY",
      "endpoint": "https://api.etherscan.io/api"
    },
    "bitcoin": {
      "endpoint": "https://blockstream.info/api"
    },
    "tron": {
      "apiKey": "YOUR_KEY",
      "endpoint": "https://api.tronscan.org/api"
    }
  },
  "monitoring": {
    "defaultPollingInterval": 30000,
    "maxAddressesPerMonitor": 100,
    "webhookUrl": "https://your-server.com/webhook"
  }
}
