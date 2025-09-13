# Security Best Practices

## Overview

Security is paramount when building blockchain applications. This guide covers best practices for securely integrating with the AetherWeb3 Multi-EVM Gateway and protecting your applications and users.

## API Key Security

### 1. API Key Management

**DO:**
- Store API keys in environment variables
- Use different API keys for different environments
- Rotate API keys regularly (every 90 days)
- Implement API key rotation in your CI/CD pipeline
- Use secrets management services (AWS Secrets Manager, HashiCorp Vault)

**DON'T:**
- Hardcode API keys in source code
- Commit API keys to version control
- Share API keys via email or chat
- Use production API keys in development/testing
- Log API keys in application logs

### 2. Environment Variable Setup

```bash
# .env file (never commit this!)
AETHERWEB3_API_KEY=your_api_key_here
AETHERWEB3_API_KEY_STAGING=staging_api_key_here
```

```javascript
// Good: Using environment variables
const apiKey = process.env.AETHERWEB3_API_KEY;

// Bad: Hardcoded API key
const apiKey = "ak_live_1234567890abcdef"; // Never do this!
```

### 3. API Key Rotation

Implement automatic API key rotation:

```javascript
class SecureAPIClient {
  constructor() {
    this.primaryKey = process.env.AETHERWEB3_PRIMARY_KEY;
    this.fallbackKey = process.env.AETHERWEB3_FALLBACK_KEY;
    this.keyRotationDate = new Date(process.env.KEY_ROTATION_DATE);
  }

  getCurrentKey() {
    const now = new Date();
    const daysSinceRotation = (now - this.keyRotationDate) / (1000 * 60 * 60 * 24);
    
    // Use fallback key if primary key is older than 90 days
    return daysSinceRotation > 90 ? this.fallbackKey : this.primaryKey;
  }

  async makeRequest(network, method, params) {
    let apiKey = this.getCurrentKey();
    
    try {
      return await this.callAPI(network, method, params, apiKey);
    } catch (error) {
      if (error.status === 401) {
        // Try with fallback key
        apiKey = apiKey === this.primaryKey ? this.fallbackKey : this.primaryKey;
        return await this.callAPI(network, method, params, apiKey);
      }
      throw error;
    }
  }
}
```

## Request Security

### 1. HTTPS Enforcement

Always use HTTPS for all requests:

```javascript
// Good: Using HTTPS
const baseURL = 'https://multi-evm-gateway-197221342816.us-central1.run.app';

// Bad: Using HTTP (never in production)
const baseURL = 'http://multi-evm-gateway-197221342816.us-central1.run.app';
```

### 2. Request Validation

Validate all inputs before sending requests:

```javascript
class SecureClient {
  validateAddress(address) {
    if (!address || typeof address !== 'string') {
      throw new Error('Address must be a string');
    }
    
    if (!/^0x[a-fA-F0-9]{40}$/.test(address)) {
      throw new Error('Invalid Ethereum address format');
    }
    
    return address.toLowerCase();
  }

  validateBlockNumber(blockNumber) {
    if (blockNumber === 'latest' || blockNumber === 'earliest' || blockNumber === 'pending') {
      return blockNumber;
    }
    
    if (typeof blockNumber === 'number' && blockNumber >= 0) {
      return `0x${blockNumber.toString(16)}`;
    }
    
    if (typeof blockNumber === 'string' && /^0x[0-9a-fA-F]+$/.test(blockNumber)) {
      return blockNumber;
    }
    
    throw new Error('Invalid block number');
  }

  async getBalance(address, blockNumber = 'latest', network = 'ethereum') {
    const validAddress = this.validateAddress(address);
    const validBlockNumber = this.validateBlockNumber(blockNumber);
    
    return this.call(network, 'eth_getBalance', [validAddress, validBlockNumber]);
  }
}
```

### 3. Request Signing (Enterprise)

For enterprise applications, implement request signing:

```javascript
const crypto = require('crypto');

class SignedRequestClient {
  constructor(apiKey, secretKey) {
    this.apiKey = apiKey;
    this.secretKey = secretKey;
  }

  signRequest(method, url, body, timestamp) {
    const payload = `${method}\n${url}\n${body}\n${timestamp}`;
    return crypto
      .createHmac('sha256', this.secretKey)
      .update(payload)
      .digest('hex');
  }

  async makeSecureRequest(network, method, params) {
    const timestamp = Date.now().toString();
    const body = JSON.stringify({
      jsonrpc: '2.0',
      method,
      params,
      id: 1
    });
    
    const url = `/${network}`;
    const signature = this.signRequest('POST', url, body, timestamp);

    const response = await fetch(`https://multi-evm-gateway-197221342816.us-central1.run.app${url}`, {
      method: 'POST',
      headers: {
        'X-API-Key': this.apiKey,
        'X-Timestamp': timestamp,
        'X-Signature': signature,
        'Content-Type': 'application/json'
      },
      body
    });

    return response.json();
  }
}
```

## Rate Limiting and DDoS Protection

### 1. Implement Client-Side Rate Limiting

```javascript
class RateLimitedClient {
  constructor(apiKey, requestsPerMinute = 20) {
    this.apiKey = apiKey;
    this.requestsPerMinute = requestsPerMinute;
    this.requestHistory = [];
  }

  canMakeRequest() {
    const now = Date.now();
    const oneMinuteAgo = now - 60000;
    
    // Remove requests older than 1 minute
    this.requestHistory = this.requestHistory.filter(time => time > oneMinuteAgo);
    
    return this.requestHistory.length < this.requestsPerMinute;
  }

  async makeRequest(network, method, params) {
    if (!this.canMakeRequest()) {
      const oldestRequest = Math.min(...this.requestHistory);
      const waitTime = 60000 - (Date.now() - oldestRequest);
      
      throw new Error(`Rate limit exceeded. Wait ${Math.ceil(waitTime / 1000)} seconds.`);
    }

    this.requestHistory.push(Date.now());
    return this.callAPI(network, method, params);
  }
}
```

### 2. Implement Exponential Backoff

```javascript
class ResilientClient {
  async makeRequestWithRetry(network, method, params, maxRetries = 3) {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        return await this.makeRequest(network, method, params);
      } catch (error) {
        if (error.status === 429 && attempt < maxRetries - 1) {
          const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
          await this.delay(delay);
          continue;
        }
        throw error;
      }
    }
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Data Protection

### 1. Sensitive Data Handling

Never log sensitive information:

```javascript
class SecureLogger {
  static sanitizeForLogging(data) {
    const sensitive = ['privateKey', 'apiKey', 'password', 'secret'];
    const sanitized = { ...data };
    
    for (const key of sensitive) {
      if (sanitized[key]) {
        sanitized[key] = '[REDACTED]';
      }
    }
    
    return sanitized;
  }

  static logRequest(method, params) {
    const sanitized = this.sanitizeForLogging({ method, params });
    console.log('API Request:', sanitized);
  }
}
```

### 2. Encryption at Rest

Encrypt sensitive data when storing locally:

```javascript
const crypto = require('crypto');

class SecureStorage {
  constructor(encryptionKey) {
    this.algorithm = 'aes-256-gcm';
    this.key = crypto.scryptSync(encryptionKey, 'salt', 32);
  }

  encrypt(text) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher(this.algorithm, this.key, { iv });
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  }

  decrypt(encryptedData) {
    const decipher = crypto.createDecipher(this.algorithm, this.key, {
      iv: Buffer.from(encryptedData.iv, 'hex')
    });
    
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}
```

## Network Security

### 1. IP Whitelisting (Enterprise)

For enterprise applications, implement IP whitelisting:

```javascript
// Server-side middleware example
const allowedIPs = process.env.ALLOWED_IPS?.split(',') || [];

function ipWhitelistMiddleware(req, res, next) {
  const clientIP = req.ip || req.connection.remoteAddress;
  
  if (allowedIPs.length > 0 && !allowedIPs.includes(clientIP)) {
    return res.status(403).json({
      error: 'IP address not whitelisted',
      clientIP
    });
  }
  
  next();
}
```

### 2. CORS Configuration

Properly configure CORS for web applications:

```javascript
// Express.js CORS configuration
const cors = require('cors');

const corsOptions = {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  methods: ['GET', 'POST'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400 // Cache preflight for 24 hours
};

app.use(cors(corsOptions));
```

## Application Security

### 1. Input Sanitization

Always sanitize user inputs:

```javascript
const validator = require('validator');

class InputSanitizer {
  static sanitizeAddress(address) {
    if (!address || typeof address !== 'string') {
      throw new Error('Invalid address input');
    }
    
    // Remove any non-hex characters except 0x prefix
    const cleaned = address.replace(/[^0-9a-fA-F]/g, '');
    
    if (cleaned.length !== 40) {
      throw new Error('Address must be 40 hex characters');
    }
    
    return `0x${cleaned}`;
  }

  static sanitizeAmount(amount) {
    if (!validator.isNumeric(amount.toString())) {
      throw new Error('Amount must be numeric');
    }
    
    const num = parseFloat(amount);
    if (num < 0 || !isFinite(num)) {
      throw new Error('Amount must be a positive finite number');
    }
    
    return num;
  }

  static sanitizeNetwork(network) {
    const allowedNetworks = ['ethereum', 'base', 'arbitrum'];
    
    if (!allowedNetworks.includes(network)) {
      throw new Error(`Unsupported network: ${network}`);
    }
    
    return network;
  }
}
```

### 2. Error Handling

Implement secure error handling:

```javascript
class SecureErrorHandler {
  static handleAPIError(error) {
    // Log full error details internally
    console.error('Internal error:', error);
    
    // Return sanitized error to client
    if (error.status >= 400 && error.status < 500) {
      // Client errors - safe to expose
      return {
        error: error.message,
        code: error.status
      };
    } else {
      // Server errors - don't expose internal details
      return {
        error: 'Internal server error',
        code: 500
      };
    }
  }

  static async safeAPICall(apiCall) {
    try {
      return await apiCall();
    } catch (error) {
      throw this.handleAPIError(error);
    }
  }
}
```

## Smart Contract Security

### 1. Transaction Verification

Always verify transaction parameters:

```javascript
class TransactionValidator {
  static validateGasPrice(gasPrice) {
    const gasPriceWei = BigInt(gasPrice);
    const maxGasPrice = BigInt('100000000000'); // 100 gwei
    
    if (gasPriceWei > maxGasPrice) {
      throw new Error('Gas price too high');
    }
    
    return gasPrice;
  }

  static validateGasLimit(gasLimit) {
    const limit = parseInt(gasLimit);
    const maxGasLimit = 10000000; // 10M gas
    
    if (limit > maxGasLimit) {
      throw new Error('Gas limit too high');
    }
    
    return limit;
  }

  static validateTransaction(tx) {
    return {
      ...tx,
      gasPrice: this.validateGasPrice(tx.gasPrice),
      gasLimit: this.validateGasLimit(tx.gasLimit),
      to: InputSanitizer.sanitizeAddress(tx.to),
      value: tx.value ? this.validateValue(tx.value) : '0x0'
    };
  }
}
```

### 2. Contract Call Safety

Implement safe contract interaction patterns:

```javascript
class SafeContractCaller {
  async safeCall(network, to, data, value = '0x0') {
    // Simulate the call first
    try {
      await this.simulateCall(network, to, data, value);
    } catch (error) {
      throw new Error(`Call simulation failed: ${error.message}`);
    }

    // If simulation passes, make the actual call
    return this.makeCall(network, to, data, value);
  }

  async simulateCall(network, to, data, value) {
    return this.apiClient.call(network, 'eth_call', [{
      to,
      data,
      value
    }, 'latest']);
  }

  async estimateGas(network, to, data, value) {
    const gasEstimate = await this.apiClient.call(network, 'eth_estimateGas', [{
      to,
      data,
      value
    }]);

    // Add 20% buffer to gas estimate
    const gasWithBuffer = Math.floor(parseInt(gasEstimate, 16) * 1.2);
    return `0x${gasWithBuffer.toString(16)}`;
  }
}
```

## Monitoring and Alerting

### 1. Security Monitoring

Implement security monitoring:

```javascript
class SecurityMonitor {
  constructor() {
    this.alertThresholds = {
      failedRequests: 10,      // 10 failed requests in 1 minute
      rateLimitHits: 5,        // 5 rate limit hits in 1 minute
      suspiciousIPs: 3         // 3 different suspicious IPs
    };
    
    this.securityEvents = [];
  }

  logSecurityEvent(type, details) {
    const event = {
      type,
      details,
      timestamp: Date.now(),
      ip: details.ip || 'unknown'
    };

    this.securityEvents.push(event);
    this.checkAlertThresholds();
    
    // Log to security system
    console.warn('Security Event:', event);
  }

  checkAlertThresholds() {
    const oneMinuteAgo = Date.now() - 60000;
    const recentEvents = this.securityEvents.filter(e => e.timestamp > oneMinuteAgo);
    
    const failedRequests = recentEvents.filter(e => e.type === 'failed_request').length;
    const rateLimitHits = recentEvents.filter(e => e.type === 'rate_limit').length;
    
    if (failedRequests > this.alertThresholds.failedRequests) {
      this.triggerAlert('High number of failed requests', { count: failedRequests });
    }
    
    if (rateLimitHits > this.alertThresholds.rateLimitHits) {
      this.triggerAlert('High number of rate limit hits', { count: rateLimitHits });
    }
  }

  triggerAlert(message, data) {
    console.error('SECURITY ALERT:', message, data);
    // Send to alerting system (PagerDuty, Slack, etc.)
  }
}
```

### 2. Audit Logging

Implement comprehensive audit logging:

```javascript
class AuditLogger {
  static log(action, userId, details = {}) {
    const auditEvent = {
      timestamp: new Date().toISOString(),
      action,
      userId,
      details: SecureLogger.sanitizeForLogging(details),
      sessionId: details.sessionId,
      ip: details.ip,
      userAgent: details.userAgent
    };

    // Log to audit trail
    console.log('AUDIT:', JSON.stringify(auditEvent));
    
    // Store in secure audit database
    this.storeAuditEvent(auditEvent);
  }

  static async storeAuditEvent(event) {
    // Store in tamper-proof audit log
    // Implementation depends on your audit system
  }
}
```

## Compliance and Regulations

### 1. GDPR Compliance

Handle user data according to GDPR:

```javascript
class GDPRCompliantClient {
  async deleteUserData(userId) {
    // Remove all user data
    await this.database.deleteUser(userId);
    
    // Log the deletion for audit purposes
    AuditLogger.log('user_data_deleted', userId, {
      reason: 'GDPR right to erasure',
      deletedAt: new Date().toISOString()
    });
  }

  async exportUserData(userId) {
    const userData = await this.database.getUserData(userId);
    
    // Log the export for audit purposes
    AuditLogger.log('user_data_exported', userId, {
      reason: 'GDPR data portability request',
      exportedAt: new Date().toISOString()
    });
    
    return this.sanitizeExportData(userData);
  }

  sanitizeExportData(data) {
    // Remove internal system fields
    const { internalId, systemFlags, ...exportData } = data;
    return exportData;
  }
}
```

### 2. SOC 2 Compliance

Implement SOC 2 security controls:

```javascript
class SOC2Controls {
  static enforcePasswordPolicy(password) {
    if (password.length < 12) {
      throw new Error('Password must be at least 12 characters');
    }
    
    if (!/(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/.test(password)) {
      throw new Error('Password must contain uppercase, lowercase, number, and special character');
    }
  }

  static enforceSessionTimeout(session) {
    const maxSessionAge = 8 * 60 * 60 * 1000; // 8 hours
    const sessionAge = Date.now() - session.createdAt;
    
    if (sessionAge > maxSessionAge) {
      throw new Error('Session expired');
    }
  }

  static enforceDataRetention(data) {
    const maxRetentionPeriod = 7 * 365 * 24 * 60 * 60 * 1000; // 7 years
    const dataAge = Date.now() - data.createdAt;
    
    if (dataAge > maxRetentionPeriod) {
      return this.archiveData(data);
    }
    
    return data;
  }
}
```

## Security Checklist

### Development
- [ ] API keys stored in environment variables
- [ ] Input validation implemented
- [ ] Error handling sanitizes sensitive data
- [ ] HTTPS enforced for all requests
- [ ] Request signing implemented (enterprise)
- [ ] Rate limiting configured
- [ ] Audit logging enabled

### Deployment
- [ ] Secrets management system configured
- [ ] Network security groups configured
- [ ] SSL/TLS certificates installed
- [ ] Monitoring and alerting set up
- [ ] Security scanning integrated into CI/CD
- [ ] Incident response procedures documented

### Operations
- [ ] Regular security assessments conducted
- [ ] API key rotation scheduled
- [ ] Monitoring dashboards reviewed
- [ ] Security logs analyzed
- [ ] Compliance requirements verified
- [ ] Staff security training completed

---

*Last Updated: January 2025*