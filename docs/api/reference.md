# API Reference Documentation

## Overview

The AetherWeb3 Multi-EVM Gateway provides a unified RESTful JSON-RPC API for interacting with multiple EVM-compatible blockchain networks. This documentation covers all available endpoints, authentication methods, and usage examples.

## Base URL

```
https://multi-evm-gateway-197221342816.us-central1.run.app
```

## Authentication

All API requests require authentication using an API key. Include your API key in the request headers:

```http
X-API-Key: YOUR_API_KEY
```

### Getting Your API Key

1. Register at [aetherweb3.xyz](https://aetherweb3.xyz)
2. Complete email verification
3. Access your dashboard to retrieve your API key

## Supported Networks

| Network | Endpoint | Chain ID | RPC Specification |
|---------|----------|----------|-------------------|
| Ethereum Mainnet | `/ethereum` | 1 | JSON-RPC 2.0 |
| Base | `/base` | 8453 | JSON-RPC 2.0 |
| Arbitrum One | `/arbitrum` | 42161 | JSON-RPC 2.0 |

## Rate Limits

Rate limits are applied per API key and reset every minute:

| Plan | Requests/Minute | Daily Limit | Monthly Limit |
|------|----------------|-------------|---------------|
| Free | 20 | 120,000 | N/A |
| Starter | 60 | 120,000 + overages | 25M |
| Pro | 200 | 120,000 + overages | 200M |
| Enterprise | Custom | Custom | Custom |

### Rate Limit Headers

All responses include rate limit information:

```http
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 19
X-RateLimit-Reset: 1640995200
```

## Standard JSON-RPC Methods

All networks support standard Ethereum JSON-RPC methods. Here are the most commonly used:

### eth_getBalance

Get the balance of an address.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": ["0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5", "latest"],
  "id": 1
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0x1b1ae4d6e2ef500000"
}
```

### eth_getBlockByNumber

Get information about a block by number.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBlockByNumber",
  "params": ["latest", true],
  "id": 1
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "number": "0x1b4",
    "hash": "0xe670ec64341771606e55d6b4ca35a1a6b75ee3d5145a99d05921026d1527331",
    "parentHash": "0x9646252be9520f6e71339a8df9c55e4d7619deeb018d2a3f2d21fc165dde5eb5",
    "timestamp": "0x55ba467c",
    "transactions": [...],
    "...": "..."
  }
}
```

### eth_sendRawTransaction

Submit a signed transaction to the network.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "eth_sendRawTransaction",
  "params": ["0xf86c098504a817c800825208943535353535353535353535353535353535353535880de0b6b3a76400008025a028ef61340bd939bc2195fe537567866003e1a15d3c71ff63e1590620aa636276a067ccc224e3300cba3b2e5e6f0f4a6a3b0c2a2c5c8c4b5e5d5b3c3e8f8c1b8a6c"],
  "id": 1
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0xe670ec64341771606e55d6b4ca35a1a6b75ee3d5145a99d05921026d1527331"
}
```

### eth_call

Execute a contract call without creating a transaction.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "eth_call",
  "params": [{
    "to": "0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b",
    "data": "0x70a082310000000000000000000000002f2a0e4ee7e1c4cbea6e38fe1c5b2bb3e9da0b04"
  }, "latest"],
  "id": 1
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0x0000000000000000000000000000000000000000000000000de0b6b3a7640000"
}
```

## WebSocket API

For real-time data, connect to our WebSocket endpoints:

```
wss://multi-evm-gateway-197221342816.us-central1.run.app/ws/{network}?auth=YOUR_API_KEY
```

### Subscription Methods

#### newHeads

Subscribe to new block headers:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_subscribe",
  "params": ["newHeads"],
  "id": 1
}
```

#### logs

Subscribe to logs matching filter criteria:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_subscribe",
  "params": ["logs", {
    "address": "0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b",
    "topics": ["0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"]
  }],
  "id": 1
}
```

#### newPendingTransactions

Subscribe to pending transactions:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_subscribe",
  "params": ["newPendingTransactions"],
  "id": 1
}
```

### Unsubscribing

```json
{
  "jsonrpc": "2.0",
  "method": "eth_unsubscribe",
  "params": ["0x12345..."],
  "id": 1
}
```

## Error Handling

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 400 | Bad Request |
| 401 | Unauthorized (Invalid API Key) |
| 403 | Forbidden (Rate Limit Exceeded) |
| 429 | Too Many Requests |
| 500 | Internal Server Error |
| 503 | Service Unavailable |

### JSON-RPC Error Codes

| Code | Message | Description |
|------|---------|-------------|
| -32700 | Parse error | Invalid JSON was received |
| -32600 | Invalid Request | The JSON sent is not a valid Request object |
| -32601 | Method not found | The method does not exist / is not available |
| -32602 | Invalid params | Invalid method parameter(s) |
| -32603 | Internal error | Internal JSON-RPC error |

### Error Response Format

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": "Expected address to be a 40-character hex string"
  },
  "id": 1
}
```

## Code Examples

### JavaScript (Node.js)

```javascript
const axios = require('axios');

const apiKey = 'YOUR_API_KEY';
const baseUrl = 'https://multi-evm-gateway-197221342816.us-central1.run.app';

async function getBalance(address, network = 'ethereum') {
  try {
    const response = await axios.post(`${baseUrl}/${network}`, {
      jsonrpc: '2.0',
      method: 'eth_getBalance',
      params: [address, 'latest'],
      id: 1
    }, {
      headers: {
        'X-API-Key': apiKey,
        'Content-Type': 'application/json'
      }
    });
    
    return response.data.result;
  } catch (error) {
    console.error('Error:', error.response?.data || error.message);
    throw error;
  }
}

// Usage
getBalance('0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5')
  .then(balance => console.log('Balance:', balance))
  .catch(console.error);
```

### Python

```python
import requests
import json

class AetherWeb3Client:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = 'https://multi-evm-gateway-197221342816.us-central1.run.app'
        self.headers = {
            'X-API-Key': api_key,
            'Content-Type': 'application/json'
        }
    
    def call_method(self, network, method, params, id=1):
        payload = {
            'jsonrpc': '2.0',
            'method': method,
            'params': params,
            'id': id
        }
        
        response = requests.post(
            f'{self.base_url}/{network}',
            headers=self.headers,
            data=json.dumps(payload)
        )
        
        response.raise_for_status()
        return response.json()
    
    def get_balance(self, address, network='ethereum'):
        result = self.call_method(network, 'eth_getBalance', [address, 'latest'])
        return result['result']

# Usage
client = AetherWeb3Client('YOUR_API_KEY')
balance = client.get_balance('0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5')
print(f'Balance: {balance}')
```

### cURL

```bash
#!/bin/bash

API_KEY="YOUR_API_KEY"
BASE_URL="https://multi-evm-gateway-197221342816.us-central1.run.app"

# Get balance
curl -X POST "${BASE_URL}/ethereum" \
  -H "X-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_getBalance",
    "params": ["0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5", "latest"],
    "id": 1
  }'

# Get latest block
curl -X POST "${BASE_URL}/ethereum" \
  -H "X-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_getBlockByNumber",
    "params": ["latest", false],
    "id": 1
  }'
```

## Performance Optimization

### Batch Requests

Reduce latency by batching multiple requests:

```json
[
  {
    "jsonrpc": "2.0",
    "method": "eth_getBalance",
    "params": ["0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5", "latest"],
    "id": 1
  },
  {
    "jsonrpc": "2.0",
    "method": "eth_getBalance",
    "params": ["0x8ba1f109551bD432803012645Hac136c80a1f1bb", "latest"],
    "id": 2
  }
]
```

### Caching Recommendations

- Cache static data (contract ABIs, block data older than 10 blocks)
- Use ETags for conditional requests
- Implement client-side caching for immutable data

### Connection Pooling

For high-throughput applications:
- Reuse HTTP connections
- Implement connection pooling
- Use HTTP/2 when available

## Support and Resources

### Getting Help

- **Technical Support**: [support@nibertinvestments.com](mailto:support@nibertinvestments.com)
- **Sales Inquiries**: [sales@nibertinvestments.com](mailto:sales@nibertinvestments.com)
- **Status Page**: Monitor service health and incidents
- **Discord Community**: Join our developer community

### Additional Resources

- [Getting Started Guide](../guides/getting-started.md)
- [Architecture Overview](../architecture/overview.md)
- [Security Best Practices](../security/best-practices.md)
- [Troubleshooting Guide](../guides/troubleshooting.md)

---

*Last Updated: January 2025*