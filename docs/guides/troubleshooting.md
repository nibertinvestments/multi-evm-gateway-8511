# Troubleshooting Guide

## Overview

This guide covers common issues, their solutions, and debugging techniques for the AetherWeb3 Multi-EVM Gateway. If you can't find a solution here, please contact our support team.

## Common Issues

### 1. Authentication Problems

#### Invalid API Key Error
```json
{
  "error": "Invalid API key",
  "code": 401
}
```

**Causes:**
- API key is incorrect or expired
- API key is not included in request headers
- API key format is malformed

**Solutions:**
1. **Verify API Key**: Check your dashboard for the correct API key
2. **Check Header Format**: Ensure you're using `X-API-Key` header
3. **Regenerate Key**: Create a new API key if the current one is compromised

```javascript
// Correct header format
const headers = {
  'X-API-Key': 'your-api-key-here',
  'Content-Type': 'application/json'
};
```

#### API Key Not Found
```json
{
  "error": "API key not found",
  "code": 401
}
```

**Solution**: Ensure your account is active and API key exists in our system.

### 2. Rate Limiting Issues

#### Rate Limit Exceeded
```json
{
  "error": "Rate limit exceeded",
  "code": 429,
  "resetTime": 1640995200
}
```

**Understanding Rate Limits:**
- **Free Tier**: 20 requests/minute, 120,000/day
- **Starter**: 60 requests/minute, 25M/month
- **Pro**: 200 requests/minute, 200M/month

**Solutions:**
1. **Implement Backoff**: Wait for reset time before retrying
2. **Optimize Requests**: Batch multiple operations
3. **Upgrade Plan**: Consider higher tier for more capacity
4. **Cache Results**: Store frequently accessed data locally

```javascript
// Exponential backoff implementation
async function retryWithBackoff(apiCall, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await apiCall();
    } catch (error) {
      if (error.code === 429 && attempt < maxRetries - 1) {
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
}
```

### 3. Network Connection Issues

#### Connection Timeout
```
Error: Request timeout after 30000ms
```

**Causes:**
- Network connectivity issues
- High network latency
- Blockchain network congestion
- Service overload

**Solutions:**
1. **Increase Timeout**: Set longer timeout values
2. **Retry Logic**: Implement automatic retries
3. **Check Network Status**: Verify blockchain network health
4. **Use Different Network**: Try alternative networks if available

```javascript
// Configurable timeout
const config = {
  timeout: 60000, // 60 seconds
  retries: 3
};

async function makeRequestWithTimeout(url, data, config) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), config.timeout);

  try {
    const response = await fetch(url, {
      method: 'POST',
      body: JSON.stringify(data),
      headers: { 'Content-Type': 'application/json' },
      signal: controller.signal
    });
    return response.json();
  } finally {
    clearTimeout(timeoutId);
  }
}
```

#### DNS Resolution Failures
```
Error: getaddrinfo ENOTFOUND multi-evm-gateway-197221342816.us-central1.run.app
```

**Solutions:**
1. **Check Internet Connection**: Verify network connectivity
2. **DNS Settings**: Try different DNS servers (8.8.8.8, 1.1.1.1)
3. **Corporate Firewall**: Contact IT about firewall rules
4. **Use IP Address**: Temporarily use direct IP if DNS fails

### 4. JSON-RPC Errors

#### Method Not Found
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32601,
    "message": "Method not found"
  },
  "id": 1
}
```

**Causes:**
- Typo in method name
- Method not supported by the network
- Network-specific method used on wrong network

**Solutions:**
1. **Check Method Name**: Verify spelling and case
2. **Consult Documentation**: Check supported methods for each network
3. **Network Compatibility**: Use correct method for target network

```javascript
// Supported methods by network
const SUPPORTED_METHODS = {
  ethereum: ['eth_getBalance', 'eth_getBlockByNumber', 'eth_sendRawTransaction'],
  base: ['eth_getBalance', 'eth_getBlockByNumber', 'eth_sendRawTransaction'],
  arbitrum: ['eth_getBalance', 'eth_getBlockByNumber', 'eth_sendRawTransaction', 'arb_getL1BaseFee']
};

function validateMethod(network, method) {
  if (!SUPPORTED_METHODS[network]?.includes(method)) {
    throw new Error(`Method ${method} not supported on ${network}`);
  }
}
```

#### Invalid Params
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params"
  },
  "id": 1
}
```

**Common Parameter Issues:**
- Incorrect parameter count
- Wrong parameter types
- Invalid address format
- Invalid block number format

**Solutions:**
1. **Validate Parameters**: Check type and format before sending
2. **Use Examples**: Copy working examples from documentation
3. **Address Format**: Ensure addresses start with 0x and are 42 characters

```javascript
// Parameter validation
function validateEthereumAddress(address) {
  if (typeof address !== 'string') {
    throw new Error('Address must be a string');
  }
  if (!/^0x[a-fA-F0-9]{40}$/.test(address)) {
    throw new Error('Invalid Ethereum address format');
  }
}

function validateBlockNumber(blockNumber) {
  const valid = ['latest', 'earliest', 'pending'];
  if (valid.includes(blockNumber)) return true;
  if (/^0x[0-9a-fA-F]+$/.test(blockNumber)) return true;
  if (Number.isInteger(blockNumber) && blockNumber >= 0) return true;
  throw new Error('Invalid block number format');
}
```

### 5. WebSocket Connection Issues

#### Connection Failed
```
WebSocket connection failed: Error during WebSocket handshake
```

**Causes:**
- Invalid WebSocket URL
- Missing authentication parameter
- Network firewall blocking WebSocket connections
- Proxy server issues

**Solutions:**
1. **Check URL Format**: Verify WebSocket URL structure
2. **Authentication**: Include API key in connection string
3. **Firewall Rules**: Ensure WebSocket traffic is allowed
4. **Proxy Configuration**: Configure proxy for WebSocket support

```javascript
// Correct WebSocket connection
const ws = new WebSocket(
  `wss://multi-evm-gateway-197221342816.us-central1.run.app/ws/ethereum?auth=${apiKey}`
);

ws.onopen = function() {
  console.log('WebSocket connected');
};

ws.onerror = function(error) {
  console.error('WebSocket error:', error);
  // Implement reconnection logic
};

ws.onclose = function(event) {
  console.log('WebSocket closed:', event.code, event.reason);
  // Implement auto-reconnection
};
```

#### Subscription Failures
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32600,
    "message": "Invalid subscription"
  },
  "id": 1
}
```

**Solutions:**
1. **Valid Subscription Types**: Use supported subscription methods
2. **Parameter Format**: Check subscription parameters
3. **Connection State**: Ensure WebSocket is open before subscribing

```javascript
// Supported subscription types
const VALID_SUBSCRIPTIONS = [
  'newHeads',
  'logs', 
  'newPendingTransactions',
  'syncing'
];

function subscribe(ws, type, params = []) {
  if (!VALID_SUBSCRIPTIONS.includes(type)) {
    throw new Error(`Invalid subscription type: ${type}`);
  }

  const subscription = {
    jsonrpc: '2.0',
    method: 'eth_subscribe',
    params: [type, ...params],
    id: Math.floor(Math.random() * 1000000)
  };

  ws.send(JSON.stringify(subscription));
}
```

## Debugging Techniques

### 1. Enable Verbose Logging

```javascript
// Add detailed logging to your requests
class DebugClient {
  async makeRequest(network, method, params) {
    const requestId = Math.random().toString(36).substr(2, 9);
    
    console.log(`[${requestId}] Request:`, {
      network,
      method,
      params,
      timestamp: new Date().toISOString()
    });

    try {
      const response = await this.apiCall(network, method, params);
      
      console.log(`[${requestId}] Response:`, {
        result: response,
        timestamp: new Date().toISOString()
      });
      
      return response;
    } catch (error) {
      console.error(`[${requestId}] Error:`, {
        error: error.message,
        stack: error.stack,
        timestamp: new Date().toISOString()
      });
      throw error;
    }
  }
}
```

### 2. Network Connectivity Testing

```bash
# Test basic connectivity
curl -v https://multi-evm-gateway-197221342816.us-central1.run.app/ethereum

# Test with authentication
curl -v -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://multi-evm-gateway-197221342816.us-central1.run.app/ethereum

# Test WebSocket connection
wscat -c "wss://multi-evm-gateway-197221342816.us-central1.run.app/ws/ethereum?auth=your-api-key"
```

### 3. Request/Response Inspection

```javascript
// Intercept and log all HTTP requests
const originalFetch = global.fetch;
global.fetch = async (url, options) => {
  console.log('HTTP Request:', {
    url,
    method: options?.method || 'GET',
    headers: options?.headers,
    body: options?.body
  });

  const response = await originalFetch(url, options);
  
  console.log('HTTP Response:', {
    status: response.status,
    statusText: response.statusText,
    headers: Object.fromEntries(response.headers.entries())
  });

  return response;
};
```

### 4. Performance Monitoring

```javascript
// Monitor request performance
class PerformanceMonitor {
  constructor() {
    this.metrics = [];
  }

  async timeRequest(apiCall) {
    const start = performance.now();
    const startMemory = process.memoryUsage();

    try {
      const result = await apiCall();
      const end = performance.now();
      const endMemory = process.memoryUsage();

      this.metrics.push({
        duration: end - start,
        memoryDelta: endMemory.heapUsed - startMemory.heapUsed,
        timestamp: new Date().toISOString(),
        success: true
      });

      return result;
    } catch (error) {
      const end = performance.now();
      
      this.metrics.push({
        duration: end - start,
        error: error.message,
        timestamp: new Date().toISOString(),
        success: false
      });

      throw error;
    }
  }

  getStats() {
    const successful = this.metrics.filter(m => m.success);
    const failed = this.metrics.filter(m => !m.success);

    return {
      totalRequests: this.metrics.length,
      successfulRequests: successful.length,
      failedRequests: failed.length,
      averageResponseTime: successful.reduce((sum, m) => sum + m.duration, 0) / successful.length,
      errorRate: (failed.length / this.metrics.length) * 100
    };
  }
}
```

## Environment-Specific Issues

### 1. Browser Environment

#### CORS Issues
```
Access to fetch at 'https://...' from origin 'https://...' has been blocked by CORS policy
```

**Solution**: API calls from browsers must be made from allowed origins. For security, direct browser calls are restricted.

**Workaround**: Use a backend proxy:
```javascript
// Backend proxy example (Express.js)
app.post('/api/blockchain/:network', async (req, res) => {
  try {
    const response = await fetch(`https://multi-evm-gateway-197221342816.us-central1.run.app/${req.params.network}`, {
      method: 'POST',
      headers: {
        'X-API-Key': process.env.AETHERWEB3_API_KEY,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(req.body)
    });
    
    const data = await response.json();
    res.json(data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### 2. Node.js Environment

#### SSL Certificate Issues
```
Error: unable to verify the first certificate
```

**Solution**: Configure SSL properly:
```javascript
// For development only - don't use in production
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';

// Better solution: configure proper SSL
const https = require('https');
const agent = new https.Agent({
  rejectUnauthorized: true,
  ca: [/* your certificate chain */]
});
```

### 3. Corporate Networks

#### Proxy Configuration
```javascript
// Configure proxy for corporate networks
const HttpsProxyAgent = require('https-proxy-agent');

const proxyAgent = new HttpsProxyAgent({
  host: 'proxy.company.com',
  port: 8080,
  auth: 'username:password' // if required
});

const response = await fetch(url, {
  agent: proxyAgent,
  // ... other options
});
```

## Error Codes Reference

### HTTP Status Codes

| Code | Meaning | Common Causes | Solutions |
|------|---------|---------------|-----------|
| 400 | Bad Request | Invalid JSON, malformed request | Validate request format |
| 401 | Unauthorized | Invalid/missing API key | Check API key |
| 403 | Forbidden | Rate limit exceeded | Implement backoff |
| 404 | Not Found | Invalid endpoint | Check URL |
| 429 | Too Many Requests | Rate limit hit | Wait and retry |
| 500 | Internal Server Error | Server-side issue | Contact support |
| 502 | Bad Gateway | Service temporarily down | Retry later |
| 503 | Service Unavailable | Maintenance mode | Check status page |

### JSON-RPC Error Codes

| Code | Message | Meaning | Solutions |
|------|---------|---------|-----------|
| -32700 | Parse error | Invalid JSON | Check JSON syntax |
| -32600 | Invalid Request | Malformed RPC request | Validate RPC format |
| -32601 | Method not found | Unknown method | Check method name |
| -32602 | Invalid params | Wrong parameters | Validate parameters |
| -32603 | Internal error | Server error | Contact support |

## Getting Additional Help

### Self-Help Resources
1. **Documentation**: Check our comprehensive docs
2. **Status Page**: Monitor service health
3. **Community Forum**: Search existing discussions
4. **GitHub Issues**: Check known issues

### Contact Support

When contacting support, please include:

```
Subject: [Issue Type] Brief description

Environment:
- Programming Language: Node.js v18.x
- Library/Framework: Express.js v4.x
- Network: Ethereum
- API Plan: Starter

Issue Description:
Detailed description of the problem

Request Details:
- Method: eth_getBalance  
- Parameters: ["0x...", "latest"]
- Timestamp: 2025-01-13T10:30:00Z
- Request ID: req_abc123 (if available)

Error Message:
Complete error message and stack trace

Steps to Reproduce:
1. Step one
2. Step two
3. Step three

Expected vs Actual Behavior:
What should happen vs what actually happens
```

**Support Channels:**
- **Technical Issues**: [support@nibertinvestments.com](mailto:support@nibertinvestments.com)
- **Billing Questions**: [billing@nibertinvestments.com](mailto:billing@nibertinvestments.com)
- **Sales Inquiries**: [sales@nibertinvestments.com](mailto:sales@nibertinvestments.com)
- **Emergency Issues**: Include "URGENT" in subject line

**Response Times:**
- **Free Tier**: 48-72 hours
- **Starter**: 24-48 hours  
- **Pro**: 8-24 hours
- **Enterprise**: 4-8 hours
- **Critical Issues**: 1-4 hours (Enterprise only)

---

*Last Updated: January 2025*