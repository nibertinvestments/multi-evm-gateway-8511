# Getting Started Guide

## Welcome to AetherWeb3

This guide will help you get up and running with the AetherWeb3 Multi-EVM Gateway in just a few minutes. Whether you're building a DeFi application, NFT marketplace, or blockchain analytics tool, our gateway provides enterprise-grade access to multiple EVM networks through a single, unified API.

## Quick Start

### Step 1: Create Your Account

1. Visit [aetherweb3.xyz](https://aetherweb3.xyz)
2. Click "Get Started" or "Sign Up"
3. Fill in your details:
   - Username (3+ characters)
   - Email address
   - Secure password (6+ characters)
   - Select your initial plan (Free to start)
4. Verify your email address
5. Log into your dashboard

### Step 2: Get Your API Key

Once logged in:
1. Navigate to your dashboard
2. Your API key will be displayed in the "API Key" section
3. Copy and securely store your API key
4. **Important**: Keep your API key secret and never share it publicly

### Step 3: Make Your First Request

Let's get the latest Ethereum block:

```bash
curl -X POST https://multi-evm-gateway-197221342816.us-central1.run.app/ethereum \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_getBlockByNumber",
    "params": ["latest", false],
    "id": 1
  }'
```

Expected response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "number": "0x12d4f2c",
    "hash": "0x...",
    "timestamp": "0x659b8c8c",
    "...": "..."
  }
}
```

Congratulations! 🎉 You've successfully made your first API call.

## Understanding the Basics

### API Structure

All requests follow the JSON-RPC 2.0 specification:

```json
{
  "jsonrpc": "2.0",           // Protocol version
  "method": "eth_getBalance", // Method name
  "params": [...],            // Method parameters
  "id": 1                     // Request identifier
}
```

### Network Endpoints

| Network | URL | Chain ID |
|---------|-----|----------|
| Ethereum | `/ethereum` | 1 |
| Base | `/base` | 8453 |
| Arbitrum One | `/arbitrum` | 42161 |

### Authentication Headers

Always include your API key:
```http
X-API-Key: YOUR_API_KEY
Content-Type: application/json
```

## Common Use Cases

### 1. Get Account Balance

Check ETH balance for an address:

```javascript
const response = await fetch('https://multi-evm-gateway-197221342816.us-central1.run.app/ethereum', {
  method: 'POST',
  headers: {
    'X-API-Key': 'YOUR_API_KEY',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    jsonrpc: '2.0',
    method: 'eth_getBalance',
    params: ['0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5', 'latest'],
    id: 1
  })
});

const data = await response.json();
console.log('Balance (wei):', data.result);
```

### 2. Get Transaction Details

Retrieve transaction information:

```python
import requests
import json

def get_transaction(tx_hash, network='ethereum'):
    payload = {
        'jsonrpc': '2.0',
        'method': 'eth_getTransactionByHash',
        'params': [tx_hash],
        'id': 1
    }
    
    response = requests.post(
        f'https://multi-evm-gateway-197221342816.us-central1.run.app/{network}',
        headers={
            'X-API-Key': 'YOUR_API_KEY',
            'Content-Type': 'application/json'
        },
        data=json.dumps(payload)
    )
    
    return response.json()['result']

# Usage
tx = get_transaction('0x...')
print(f'From: {tx["from"]}, To: {tx["to"]}, Value: {tx["value"]}')
```

### 3. Monitor New Blocks

Real-time block monitoring with WebSocket:

```javascript
const ws = new WebSocket('wss://multi-evm-gateway-197221342816.us-central1.run.app/ws/ethereum?auth=YOUR_API_KEY');

ws.onopen = function() {
  // Subscribe to new blocks
  ws.send(JSON.stringify({
    jsonrpc: '2.0',
    method: 'eth_subscribe',
    params: ['newHeads'],
    id: 1
  }));
};

ws.onmessage = function(event) {
  const data = JSON.parse(event.data);
  if (data.method === 'eth_subscription') {
    const blockData = data.params.result;
    console.log('New block:', blockData.number, blockData.hash);
  }
};
```

### 4. Call Smart Contract

Read from a smart contract:

```bash
# Get ERC-20 token balance
curl -X POST https://multi-evm-gateway-197221342816.us-central1.run.app/ethereum \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_call",
    "params": [{
      "to": "0xA0b86a33E6441ee2BB4c6d80D5bEDBDe22E7F6B9",
      "data": "0x70a08231000000000000000000000000742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5"
    }, "latest"],
    "id": 1
  }'
```

## Development Best Practices

### 1. Error Handling

Always implement proper error handling:

```javascript
async function safeApiCall(network, method, params) {
  try {
    const response = await fetch(`https://multi-evm-gateway-197221342816.us-central1.run.app/${network}`, {
      method: 'POST',
      headers: {
        'X-API-Key': process.env.AETHERWEB3_API_KEY,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method,
        params,
        id: Date.now()
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const data = await response.json();
    
    if (data.error) {
      throw new Error(`RPC Error ${data.error.code}: ${data.error.message}`);
    }
    
    return data.result;
  } catch (error) {
    console.error('API call failed:', error);
    throw error;
  }
}
```

### 2. Rate Limit Handling

Monitor and respect rate limits:

```javascript
class RateLimitedClient {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.requestQueue = [];
    this.isProcessing = false;
  }

  async makeRequest(network, method, params) {
    return new Promise((resolve, reject) => {
      this.requestQueue.push({ network, method, params, resolve, reject });
      this.processQueue();
    });
  }

  async processQueue() {
    if (this.isProcessing || this.requestQueue.length === 0) return;
    
    this.isProcessing = true;
    
    while (this.requestQueue.length > 0) {
      const request = this.requestQueue.shift();
      
      try {
        const result = await this.executeRequest(request);
        request.resolve(result);
      } catch (error) {
        if (error.status === 429) {
          // Rate limited - put request back and wait
          this.requestQueue.unshift(request);
          await this.delay(60000); // Wait 1 minute
          continue;
        }
        request.reject(error);
      }
      
      // Small delay between requests
      await this.delay(50);
    }
    
    this.isProcessing = false;
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 3. Caching Strategy

Implement intelligent caching:

```javascript
class CachedClient {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.cache = new Map();
  }

  getCacheKey(network, method, params) {
    return `${network}:${method}:${JSON.stringify(params)}`;
  }

  shouldCache(method) {
    // Cache immutable data
    const cacheableMethods = [
      'eth_getBlockByHash',
      'eth_getTransactionByHash',
      'eth_getTransactionReceipt'
    ];
    return cacheableMethods.includes(method);
  }

  async call(network, method, params) {
    const cacheKey = this.getCacheKey(network, method, params);
    
    // Check cache first
    if (this.shouldCache(method) && this.cache.has(cacheKey)) {
      const cached = this.cache.get(cacheKey);
      if (Date.now() - cached.timestamp < 300000) { // 5 minutes
        return cached.data;
      }
    }

    // Make API call
    const result = await this.makeApiCall(network, method, params);
    
    // Cache result if appropriate
    if (this.shouldCache(method)) {
      this.cache.set(cacheKey, {
        data: result,
        timestamp: Date.now()
      });
    }

    return result;
  }
}
```

## Language-Specific Examples

### JavaScript/Node.js

Install dependencies:
```bash
npm install axios
```

Create a client:
```javascript
const axios = require('axios');

class AetherWeb3Client {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.baseUrl = 'https://multi-evm-gateway-197221342816.us-central1.run.app';
  }

  async call(network, method, params) {
    const response = await axios.post(`${this.baseUrl}/${network}`, {
      jsonrpc: '2.0',
      method,
      params,
      id: Math.floor(Math.random() * 1000000)
    }, {
      headers: {
        'X-API-Key': this.apiKey,
        'Content-Type': 'application/json'
      }
    });

    if (response.data.error) {
      throw new Error(`RPC Error: ${response.data.error.message}`);
    }

    return response.data.result;
  }

  // Convenience methods
  async getBalance(address, network = 'ethereum') {
    return this.call(network, 'eth_getBalance', [address, 'latest']);
  }

  async getBlock(blockNumber = 'latest', network = 'ethereum') {
    return this.call(network, 'eth_getBlockByNumber', [blockNumber, true]);
  }
}

// Usage
const client = new AetherWeb3Client(process.env.AETHERWEB3_API_KEY);
client.getBalance('0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5')
  .then(balance => console.log('Balance:', balance));
```

### Python

Install dependencies:
```bash
pip install requests web3
```

Create a client:
```python
import requests
import json
from typing import List, Dict, Any

class AetherWeb3Client:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = 'https://multi-evm-gateway-197221342816.us-central1.run.app'
        self.session = requests.Session()
        self.session.headers.update({
            'X-API-Key': api_key,
            'Content-Type': 'application/json'
        })

    def call(self, network: str, method: str, params: List[Any] = None) -> Any:
        payload = {
            'jsonrpc': '2.0',
            'method': method,
            'params': params or [],
            'id': 1
        }
        
        response = self.session.post(f'{self.base_url}/{network}', 
                                   data=json.dumps(payload))
        response.raise_for_status()
        
        data = response.json()
        if 'error' in data:
            raise Exception(f"RPC Error: {data['error']['message']}")
        
        return data['result']

    def get_balance(self, address: str, network: str = 'ethereum') -> str:
        return self.call(network, 'eth_getBalance', [address, 'latest'])

    def get_block(self, block_number: str = 'latest', network: str = 'ethereum') -> Dict:
        return self.call(network, 'eth_getBlockByNumber', [block_number, True])

# Usage
import os
client = AetherWeb3Client(os.getenv('AETHERWEB3_API_KEY'))
balance = client.get_balance('0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5')
print(f'Balance: {balance}')
```

### Go

Create a client:
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
)

type RPCRequest struct {
    JSONRPC string        `json:"jsonrpc"`
    Method  string        `json:"method"`
    Params  []interface{} `json:"params"`
    ID      int           `json:"id"`
}

type RPCResponse struct {
    JSONRPC string      `json:"jsonrpc"`
    Result  interface{} `json:"result,omitempty"`
    Error   *RPCError   `json:"error,omitempty"`
    ID      int         `json:"id"`
}

type RPCError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

type AetherWeb3Client struct {
    apiKey  string
    baseURL string
    client  *http.Client
}

func NewClient(apiKey string) *AetherWeb3Client {
    return &AetherWeb3Client{
        apiKey:  apiKey,
        baseURL: "https://multi-evm-gateway-197221342816.us-central1.run.app",
        client:  &http.Client{},
    }
}

func (c *AetherWeb3Client) Call(network, method string, params []interface{}) (interface{}, error) {
    request := RPCRequest{
        JSONRPC: "2.0",
        Method:  method,
        Params:  params,
        ID:      1,
    }

    jsonData, err := json.Marshal(request)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequest("POST", fmt.Sprintf("%s/%s", c.baseURL, network), bytes.NewBuffer(jsonData))
    if err != nil {
        return nil, err
    }

    req.Header.Set("X-API-Key", c.apiKey)
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    var rpcResp RPCResponse
    if err := json.Unmarshal(body, &rpcResp); err != nil {
        return nil, err
    }

    if rpcResp.Error != nil {
        return nil, fmt.Errorf("RPC Error %d: %s", rpcResp.Error.Code, rpcResp.Error.Message)
    }

    return rpcResp.Result, nil
}

func main() {
    client := NewClient(os.Getenv("AETHERWEB3_API_KEY"))
    
    balance, err := client.Call("ethereum", "eth_getBalance", []interface{}{
        "0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5",
        "latest",
    })
    
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("Balance: %v\n", balance)
}
```

## Testing Your Integration

### Unit Tests

Always test your integration:

```javascript
// Jest test example
const AetherWeb3Client = require('./aetherweb3-client');

describe('AetherWeb3Client', () => {
  let client;

  beforeEach(() => {
    client = new AetherWeb3Client(process.env.TEST_API_KEY);
  });

  test('should get latest block', async () => {
    const block = await client.getBlock('latest', 'ethereum');
    expect(block).toHaveProperty('number');
    expect(block).toHaveProperty('hash');
    expect(block).toHaveProperty('timestamp');
  });

  test('should handle invalid address', async () => {
    await expect(
      client.getBalance('invalid_address')
    ).rejects.toThrow();
  });

  test('should respect rate limits', async () => {
    // Make multiple rapid requests
    const promises = Array(25).fill().map(() => 
      client.getBlock('latest', 'ethereum')
    );

    // Some requests should be rate limited
    const results = await Promise.allSettled(promises);
    const rateLimited = results.filter(r => 
      r.status === 'rejected' && r.reason.message.includes('rate limit')
    );
    
    expect(rateLimited.length).toBeGreaterThan(0);
  });
});
```

### Load Testing

Test your application under load:

```bash
# Using Apache Bench
ab -n 1000 -c 10 -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
   -p test-payload.json https://multi-evm-gateway-197221342816.us-central1.run.app/ethereum
```

## Next Steps

Now that you're up and running:

1. **Explore the Full API**: Check out our [API Reference](../api/reference.md)
2. **Understand the Architecture**: Read our [Architecture Overview](../architecture/overview.md)
3. **Security Best Practices**: Review [Security Guidelines](../security/best-practices.md)
4. **Join the Community**: Connect with other developers
5. **Scale Your Application**: Consider upgrading to a paid plan for higher limits

## Need Help?

- **Documentation**: Browse our comprehensive docs
- **Support**: Email [support@nibertinvestments.com](mailto:support@nibertinvestments.com)
- **Sales**: Contact [sales@nibertinvestments.com](mailto:sales@nibertinvestments.com) for enterprise needs
- **Status**: Check [status page] for service health

Welcome to the AetherWeb3 ecosystem! 🚀

---

*Last Updated: January 2025*