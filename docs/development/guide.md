# Development Guide

## Overview

This guide covers everything you need to know about developing with and contributing to the AetherWeb3 Multi-EVM Gateway project. Whether you're integrating our API into your application or contributing to the platform itself, this guide will help you get started.

## Development Environment Setup

### Prerequisites

- **Node.js**: Version 18.x or higher
- **npm**: Version 8.x or higher
- **Git**: For version control
- **Docker**: For containerized development (optional)
- **Google Cloud SDK**: For cloud deployments

### Local Development Setup

1. **Clone the Repository**
```bash
git clone https://github.com/nibertinvestments/multi-evm-gateway-8511.git
cd multi-evm-gateway-8511
```

2. **Install Dependencies**
```bash
npm install
```

3. **Environment Configuration**
```bash
# Copy environment template
cp .env.example .env

# Edit environment variables
vim .env
```

Required environment variables:
```bash
# API Configuration
NODE_ENV=development
PORT=3000
API_BASE_URL=http://localhost:3000

# Google Cloud Configuration
GOOGLE_CLOUD_PROJECT_ID=your-project-id
FIRESTORE_COLLECTION=users-dev

# Blockchain RPC Endpoints
ETHEREUM_RPC_URL=https://your-ethereum-rpc-url
BASE_RPC_URL=https://your-base-rpc-url
ARBITRUM_RPC_URL=https://your-arbitrum-rpc-url

# Security
JWT_SECRET=your-jwt-secret
API_KEY_SALT=your-api-key-salt

# Stripe (for billing)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Rate Limiting
REDIS_URL=redis://localhost:6379
```

4. **Start Development Services**
```bash
# Start Redis (for rate limiting)
docker run -d -p 6379:6379 redis:alpine

# Start the development server
npm run dev
```

### Docker Development Environment

For a fully containerized development environment:

```bash
# Build development image
docker build -t aetherweb3-dev -f Dockerfile.dev .

# Start development environment
docker-compose -f docker-compose.dev.yml up
```

Docker Compose configuration (`docker-compose.dev.yml`):
```yaml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - REDIS_URL=redis://redis:6379
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - redis

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  firestore-emulator:
    image: google/cloud-sdk:alpine
    command: >
      sh -c "gcloud beta emulators firestore start
             --host-port=0.0.0.0:8080
             --project=test-project"
    ports:
      - "8080:8080"
```

## Project Structure

```
multi-evm-gateway-8511/
├── docs/                          # Documentation
│   ├── api/                       # API documentation
│   ├── architecture/              # Architecture docs
│   ├── development/               # Development guides
│   ├── deployment/                # Deployment guides
│   ├── security/                  # Security documentation
│   └── guides/                    # User guides
├── src/                           # Source code
│   ├── controllers/               # Route controllers
│   ├── middleware/                # Express middleware
│   ├── services/                  # Business logic services
│   ├── models/                    # Data models
│   ├── utils/                     # Utility functions
│   ├── config/                    # Configuration files
│   └── app.js                     # Main application file
├── tests/                         # Test files
│   ├── unit/                      # Unit tests
│   ├── integration/               # Integration tests
│   ├── e2e/                       # End-to-end tests
│   └── fixtures/                  # Test fixtures
├── scripts/                       # Build and deployment scripts
├── .github/                       # GitHub workflows
├── package.json                   # NPM dependencies
├── Dockerfile                     # Production Docker image
├── Dockerfile.dev                 # Development Docker image
└── README.md                      # Project README
```

## API Development

### Creating New Endpoints

1. **Define Route Controller**
```javascript
// src/controllers/networkController.js
const { Router } = require('express');
const NetworkService = require('../services/NetworkService');
const authMiddleware = require('../middleware/auth');
const rateLimitMiddleware = require('../middleware/rateLimit');

const router = Router();

router.post('/:network', 
  authMiddleware,
  rateLimitMiddleware,
  async (req, res) => {
    try {
      const { network } = req.params;
      const { method, params, id } = req.body;
      
      const result = await NetworkService.handleRPCCall(network, method, params);
      
      res.json({
        jsonrpc: '2.0',
        result,
        id
      });
    } catch (error) {
      res.status(500).json({
        jsonrpc: '2.0',
        error: {
          code: -32603,
          message: error.message
        },
        id: req.body.id
      });
    }
  }
);

module.exports = router;
```

2. **Implement Service Logic**
```javascript
// src/services/NetworkService.js
const EthereumAdapter = require('./adapters/EthereumAdapter');
const BaseAdapter = require('./adapters/BaseAdapter');
const ArbitrumAdapter = require('./adapters/ArbitrumAdapter');

class NetworkService {
  static getAdapter(network) {
    switch (network) {
      case 'ethereum':
        return new EthereumAdapter();
      case 'base':
        return new BaseAdapter();
      case 'arbitrum':
        return new ArbitrumAdapter();
      default:
        throw new Error(`Unsupported network: ${network}`);
    }
  }

  static async handleRPCCall(network, method, params) {
    const adapter = this.getAdapter(network);
    return adapter.call(method, params);
  }
}

module.exports = NetworkService;
```

3. **Create Network Adapter**
```javascript
// src/services/adapters/EthereumAdapter.js
const axios = require('axios');
const BaseAdapter = require('./BaseAdapter');

class EthereumAdapter extends BaseAdapter {
  constructor() {
    super('ethereum', process.env.ETHEREUM_RPC_URL);
  }

  async call(method, params) {
    try {
      const response = await axios.post(this.rpcUrl, {
        jsonrpc: '2.0',
        method,
        params,
        id: 1
      }, {
        timeout: 30000,
        headers: {
          'Content-Type': 'application/json'
        }
      });

      if (response.data.error) {
        throw new Error(response.data.error.message);
      }

      return response.data.result;
    } catch (error) {
      throw new Error(`Ethereum RPC Error: ${error.message}`);
    }
  }
}

module.exports = EthereumAdapter;
```

### Adding New Blockchain Networks

To add support for a new blockchain network:

1. **Create Network Adapter**
```javascript
// src/services/adapters/PolygonAdapter.js
const BaseAdapter = require('./BaseAdapter');

class PolygonAdapter extends BaseAdapter {
  constructor() {
    super('polygon', process.env.POLYGON_RPC_URL);
    this.chainId = 137;
  }

  async call(method, params) {
    // Implement network-specific logic
    if (method === 'polygon_specificMethod') {
      return this.handlePolygonSpecificMethod(params);
    }

    // Fall back to standard Ethereum JSON-RPC
    return super.call(method, params);
  }

  async handlePolygonSpecificMethod(params) {
    // Implement Polygon-specific functionality
  }
}

module.exports = PolygonAdapter;
```

2. **Update Network Service**
```javascript
// Add to NetworkService.getAdapter()
case 'polygon':
  return new PolygonAdapter();
```

3. **Add Route Configuration**
```javascript
// src/app.js
app.use('/polygon', networkController);
```

4. **Update Documentation**
```markdown
# Add to API documentation
| Polygon | `/polygon` | 137 | JSON-RPC 2.0 |
```

## Testing

### Unit Tests

Write comprehensive unit tests for all components:

```javascript
// tests/unit/services/NetworkService.test.js
const NetworkService = require('../../../src/services/NetworkService');
const EthereumAdapter = require('../../../src/services/adapters/EthereumAdapter');

describe('NetworkService', () => {
  describe('getAdapter', () => {
    it('should return EthereumAdapter for ethereum network', () => {
      const adapter = NetworkService.getAdapter('ethereum');
      expect(adapter).toBeInstanceOf(EthereumAdapter);
    });

    it('should throw error for unsupported network', () => {
      expect(() => {
        NetworkService.getAdapter('unsupported');
      }).toThrow('Unsupported network: unsupported');
    });
  });

  describe('handleRPCCall', () => {
    it('should call adapter with correct parameters', async () => {
      const mockAdapter = {
        call: jest.fn().mockResolvedValue('0x123')
      };
      
      jest.spyOn(NetworkService, 'getAdapter').mockReturnValue(mockAdapter);

      const result = await NetworkService.handleRPCCall('ethereum', 'eth_getBalance', ['0x...', 'latest']);

      expect(mockAdapter.call).toHaveBeenCalledWith('eth_getBalance', ['0x...', 'latest']);
      expect(result).toBe('0x123');
    });
  });
});
```

### Integration Tests

Test API endpoints end-to-end:

```javascript
// tests/integration/api.test.js
const request = require('supertest');
const app = require('../../src/app');

describe('API Integration Tests', () => {
  const validApiKey = 'test-api-key';

  beforeEach(() => {
    // Mock authentication middleware
    jest.mock('../../src/middleware/auth', () => (req, res, next) => {
      req.user = { id: 'test-user', apiKey: validApiKey };
      next();
    });
  });

  describe('POST /ethereum', () => {
    it('should return balance for valid address', async () => {
      const response = await request(app)
        .post('/ethereum')
        .set('X-API-Key', validApiKey)
        .send({
          jsonrpc: '2.0',
          method: 'eth_getBalance',
          params: ['0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5', 'latest'],
          id: 1
        });

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('result');
      expect(response.body.jsonrpc).toBe('2.0');
      expect(response.body.id).toBe(1);
    });

    it('should return error for invalid method', async () => {
      const response = await request(app)
        .post('/ethereum')
        .set('X-API-Key', validApiKey)
        .send({
          jsonrpc: '2.0',
          method: 'invalid_method',
          params: [],
          id: 1
        });

      expect(response.status).toBe(500);
      expect(response.body).toHaveProperty('error');
    });
  });
});
```

### Load Testing

Test performance under load:

```javascript
// tests/load/loadTest.js
const { check } = require('k6');
const http = require('k6/http');

export let options = {
  stages: [
    { duration: '5m', target: 100 },   // Ramp up to 100 users
    { duration: '10m', target: 100 },  // Stay at 100 users
    { duration: '5m', target: 0 },     // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    http_req_failed: ['rate<0.05'],    // Error rate under 5%
  },
};

export default function() {
  const payload = JSON.stringify({
    jsonrpc: '2.0',
    method: 'eth_getBalance',
    params: ['0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5', 'latest'],
    id: 1
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': 'test-api-key',
    },
  };

  const response = http.post('http://localhost:3000/ethereum', payload, params);

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response has result': (r) => JSON.parse(r.body).hasOwnProperty('result'),
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
}
```

Run load tests:
```bash
k6 run tests/load/loadTest.js
```

## Code Quality

### ESLint Configuration

```javascript
// .eslintrc.js
module.exports = {
  env: {
    node: true,
    es2021: true,
    jest: true
  },
  extends: [
    'eslint:recommended',
    'airbnb-base'
  ],
  parserOptions: {
    ecmaVersion: 12,
    sourceType: 'module'
  },
  rules: {
    'no-console': 'warn',
    'no-unused-vars': 'error',
    'prefer-const': 'error',
    'no-var': 'error',
    'object-shorthand': 'error',
    'prefer-template': 'error'
  }
};
```

### Prettier Configuration

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false
}
```

### Husky Git Hooks

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm test"
    }
  },
  "lint-staged": {
    "src/**/*.js": [
      "eslint --fix",
      "prettier --write",
      "git add"
    ]
  }
}
```

## Debugging

### Development Debugging

Use VS Code debugging configuration:

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Server",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/src/app.js",
      "env": {
        "NODE_ENV": "development"
      },
      "console": "integratedTerminal",
      "restart": true,
      "runtimeExecutable": "nodemon"
    }
  ]
}
```

### Request Debugging

Add comprehensive logging:

```javascript
// src/middleware/logging.js
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

const requestLogger = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info({
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration,
      userAgent: req.get('User-Agent'),
      ip: req.ip
    });
  });

  next();
};

module.exports = { logger, requestLogger };
```

### Performance Profiling

Use clinic.js for performance analysis:

```bash
# Install clinic.js
npm install -g clinic

# Profile CPU usage
clinic doctor -- node src/app.js

# Profile event loop
clinic bubbleprof -- node src/app.js

# Analyze heap usage
clinic heapprofiler -- node src/app.js
```

## Documentation

### API Documentation

Use Swagger/OpenAPI for API documentation:

```yaml
# swagger.yml
openapi: 3.0.0
info:
  title: AetherWeb3 Multi-EVM Gateway API
  version: 1.0.0
  description: Enterprise blockchain API gateway

paths:
  /{network}:
    post:
      summary: Execute JSON-RPC call
      parameters:
        - name: network
          in: path
          required: true
          schema:
            type: string
            enum: [ethereum, base, arbitrum]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                jsonrpc:
                  type: string
                  example: "2.0"
                method:
                  type: string
                  example: "eth_getBalance"
                params:
                  type: array
                  example: ["0x742d35cc6634c0532925a3b8d3ac74e4dc3c8ef5", "latest"]
                id:
                  type: integer
                  example: 1
      responses:
        200:
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  jsonrpc:
                    type: string
                  result:
                    type: string
                  id:
                    type: integer
```

### Code Documentation

Use JSDoc for code documentation:

```javascript
/**
 * Network service for handling blockchain RPC calls
 * @class NetworkService
 */
class NetworkService {
  /**
   * Get appropriate adapter for the specified network
   * @param {string} network - The blockchain network name
   * @returns {BaseAdapter} The network adapter instance
   * @throws {Error} When network is not supported
   */
  static getAdapter(network) {
    // Implementation
  }

  /**
   * Handle RPC call for specified network
   * @param {string} network - The blockchain network name
   * @param {string} method - The RPC method name
   * @param {Array} params - The RPC method parameters
   * @returns {Promise<any>} The RPC call result
   * @throws {Error} When RPC call fails
   */
  static async handleRPCCall(network, method, params) {
    // Implementation
  }
}
```

## Deployment

### Development Deployment

Deploy to development environment:

```bash
# Build Docker image
docker build -t aetherweb3-dev .

# Deploy to Google Cloud Run
gcloud run deploy aetherweb3-dev \
  --image aetherweb3-dev \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars NODE_ENV=development
```

### Production Deployment

Production deployment with CI/CD:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '18'
      - run: npm ci
      - run: npm test
      - run: npm run lint

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: google-github-actions/setup-gcloud@v0
        with:
          service_account_key: ${{ secrets.GCP_SA_KEY }}
          project_id: ${{ secrets.GCP_PROJECT_ID }}
      
      - run: |
          gcloud builds submit --tag gcr.io/${{ secrets.GCP_PROJECT_ID }}/aetherweb3
          gcloud run deploy aetherweb3 \
            --image gcr.io/${{ secrets.GCP_PROJECT_ID }}/aetherweb3 \
            --platform managed \
            --region us-central1
```

## Contributing

### Pull Request Process

1. **Fork the Repository**
2. **Create Feature Branch**
```bash
git checkout -b feature/add-polygon-support
```

3. **Make Changes and Test**
```bash
npm test
npm run lint
```

4. **Commit Changes**
```bash
git commit -m "feat: add Polygon network support"
```

5. **Push and Create Pull Request**
```bash
git push origin feature/add-polygon-support
```

### Commit Message Format

Follow conventional commits:

```
type(scope): description

[optional body]

[optional footer]
```

Types:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Maintenance tasks

### Code Review Guidelines

- Ensure all tests pass
- Maintain code coverage above 80%
- Follow established coding standards
- Include documentation for new features
- Add appropriate error handling
- Consider security implications

---

*Last Updated: January 2025*