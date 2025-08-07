TrueSight DAO Signature Verification Service
============================================

![TrueSight DAO Logo](https://github.com/TrueSightDAO/.github/blob/main/assets/20240612_truesight_dao_logo_square.png?raw=true)

Purpose
-------

This repository contains the Node.js implementation of TrueSight DAO's signature verification service, which mirrors the functionality of the verification tool at:

https://dapp.truesight.me/verify_request.html

The service provides cryptographic verification of:
- Withdrawal requests
- Tree planting reports
- Other DAO transactions
- Community contribution claims

Key Features
------------
- RSA-SHA256 signature verification
- Standardized request format validation
- Contributor identification
- Lightweight and fast verification
- Serverless deployment ready

Technology Stack
----------------
- Node.js v22.12.0
- Express.js (web framework)
- jsrsasign (cryptographic operations)
- Vercel/Cloudflare Workers (deployment targets)

Local Development Setup
-----------------------

### Prerequisites

1. Node.js v22.12.0 (https://nodejs.org/)
2. npm (comes with Node.js)
3. Git (optional)

### Installation

```bash
git clone https://github.com/TrueSightDAO/signature-verification-service.git
cd signature-verification-service
npm install
```

### Configuration

Create a `.env` file:
```
PORT=3000
NODE_ENV=development
```

### Running the Service

#### Development Mode
```
npm run dev
```

#### Production Mode
```
npm start
```

Service runs at: http://localhost:3000

API Usage
---------

### Verify Endpoint

```bash
curl -X POST http://localhost:3000/verify \
  -H 'Content-Type: application/json' \
  -d '{"input_text":"[REQUEST]...\n--------\n\nMy Digital Signature:...\nRequest Transaction ID:..."}'
```

Deployment
----------

### Vercel

```bash
npm install -g vercel
vercel
```

Files
-----

**index.js**
```javascript
// Required: npm install express crypto jsrsasign
const express = require('express');
const crypto = require('crypto');
const KJUR = require('jsrsasign');
const app = express();

app.use(express.json());
app.use(express.text());

app.post('/verify', async (req, res) => {
  try {
    const inputText = req.body.input_text || req.body;
    
    if (!inputText) {
      return res.status(400).json({
        valid: false,
        error: 'No input text provided'
      });
    }

    const parts = inputText.split('\n\n');
    if (parts.length < 3) {
      throw new Error('Invalid input format: Must include message, signature and transaction ID');
    }

    const message = parts[0];
    const signatureLine = parts[1];
    const transactionIdLine = parts[2];

    if (!signatureLine.startsWith('My Digital Signature: ') || 
        !transactionIdLine.startsWith('Request Transaction ID: ')) {
      throw new Error('Invalid signature or transaction ID format');
    }

    const publicKeyPem = signatureLine.substring('My Digital Signature: '.length).trim();
    const signatureBase64 = transactionIdLine.substring('Request Transaction ID: '.length).trim();

    if (!message.includes('--------')) {
      throw new Error('Message must end with "--------"');
    }

    const messageToSign = message.split('--------')[0].trim();

    const rsaKey = KJUR.KEYUTIL.getKey(publicKeyPem);
    const sig = new KJUR.crypto.Signature({ alg: 'SHA256withRSA' });
    sig.init(rsaKey);
    sig.updateString(messageToSign);
    const isValid = sig.verify(Utilities.base64Decode(signatureBase64));

    res.json({
      valid: isValid,
      message: isValid ? 'Signature is valid' : 'Signature is invalid'
    });

  } catch (error) {
    res.status(400).json({
      valid: false,
      error: error.message
    });
  }
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Verification service running on port ${port}`);
});
```

**package.json**
```json
{
  "name": "signature-verification",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "jsrsasign": "^10.6.1"
  }
}
```

**vercel.json**
```json
{
  "version": 2,
  "builds": [
    {
      "src": "index.js",
      "use": "@vercel/node"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "index.js"
    }
  ]
}
```