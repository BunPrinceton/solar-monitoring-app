# Cloud Orchestration Backend - Quick Reference Guide

**Fast Lookup for Common Tasks**

---

## Quick Start (30 minutes)

```bash
# 1. Create project
mkdir cloud-orchestration-backend
cd cloud-orchestration-backend
npm init -y

# 2. Install core dependencies
npm install express cors google-spreadsheet jsonwebtoken dotenv

# 3. Create directory structure
mkdir -p src/services src/middleware src/routes admin-dashboard

# 4. Copy .env.example (see main guide)
# 5. Get Google Sheets credentials from Google Cloud Console
# 6. Create src/server.js (see main guide Section 7)
# 7. Run
node src/server.js
```

---

## Essential npm Packages

**Core:**
```bash
npm install express cors jsonwebtoken dotenv google-spreadsheet
```

**Google Sheets/Auth:**
```bash
npm install google-spreadsheet dotenv  # Service Account
npm install googleapis                 # Official Google API client
```

**Validation & Security:**
```bash
npm install joi validator helmet express-rate-limit
```

**Database (if needed):**
```bash
npm install pg  # PostgreSQL
npm install redis  # Caching
```

**Logging & Monitoring:**
```bash
npm install winston morgan prom-client sentry
```

**Automation:**
```bash
npm install node-cron puppeteer  # Scheduled tasks & browser automation
```

**Testing:**
```bash
npm install --save-dev jest supertest sinon
```

**Development:**
```bash
npm install --save-dev nodemon
```

---

## API Endpoints Reference

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | /auth/token | No | Get JWT token |
| GET | /current_readings | JWT | Fetch latest reading |
| POST | /record | JWT | Add single reading |
| POST | /bulk_record | JWT | Add multiple readings |
| POST | /push_to_utility | JWT | Submit meter to utility |
| GET | /health | No | Health check |
| GET | /metrics | No | Prometheus metrics |

---

## Google Sheets Setup Checklist

```
1. Create Google Cloud Project
   - Go to: https://console.cloud.google.com
   - Create new project: "Cloud Orchestration"

2. Enable APIs
   - Search "Google Sheets API" → Enable
   - Search "Google Drive API" → Enable

3. Create Service Account
   - IAM & Admin → Service Accounts → Create
   - Name: "cloud-orchestration-service"
   - Grant: Editor role

4. Generate Key
   - Click service account email
   - Keys tab → Add Key → JSON
   - Save as credentials.json

5. Create Spreadsheet
   - Go to Google Sheets
   - Create new spreadsheet
   - Add header row: Timestamp | Inverter_kWh | Meter_Value | Source | Note
   - Note the Spreadsheet ID from URL

6. Share with Service Account
   - Share spreadsheet with service account email from credentials.json
   - Grant "Editor" permission

7. Configure Backend
   - GOOGLE_SHEET_ID=xxxxx (from URL)
   - GOOGLE_CREDENTIALS_PATH=./credentials.json
```

---

## Environment Variables Template

```env
# Server
PORT=3000
NODE_ENV=production

# Google Sheets
GOOGLE_SHEET_ID=1mM...
GOOGLE_CREDENTIALS_PATH=./credentials.json

# JWT
JWT_SECRET=your-random-secret-32-chars-min

# Clients
CLIENT_ID_MOBILE=mobile-app
CLIENT_SECRET_MOBILE=secret-123456

# Utility Company
UTILITY_API_URL=https://api.utility.com/readings
UTILITY_API_KEY=xxx
UTILITY_LOGIN_URL=https://utility.com/login
UTILITY_USERNAME=your-account
UTILITY_PASSWORD=your-password

# Database (optional)
DATABASE_URL=postgresql://user:pass@localhost/db

# Logging
LOG_LEVEL=info

# Deployment
ALLOWED_ORIGINS=http://localhost:3000,https://yourdomain.com
```

---

## Project Structure

```
cloud-orchestration-backend/
├── src/
│   ├── server.js                 # Main entry point
│   ├── services/
│   │   ├── sheetsService.js      # Google Sheets operations
│   │   ├── tokenService.js       # JWT management
│   │   ├── utilityService.js     # Utility company integration
│   │   └── auditService.js       # Logging to database
│   ├── middleware/
│   │   ├── auth.js               # JWT verification
│   │   └── errorHandler.js       # Error handling
│   ├── utils/
│   │   ├── logger.js             # Logging
│   │   └── validators.js         # Input validation
│   └── db/
│       └── db.js                 # Database setup
├── admin-dashboard/
│   ├── index.html
│   ├── styles.css
│   ├── app.js
│   └── api.js
├── tests/
│   ├── unit/
│   └── integration/
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── package.json
└── README.md
```

---

## Core Code Snippets

### Initialize Google Sheets

```javascript
const { GoogleSpreadsheet } = require('google-spreadsheet');

async function initSheets() {
  const credentials = require('./credentials.json');
  const doc = new GoogleSpreadsheet(process.env.GOOGLE_SHEET_ID);
  await doc.useServiceAccountAuth(credentials);
  await doc.loadInfo();
  return doc.sheetsByIndex[0];
}
```

### Generate JWT Token

```javascript
const jwt = require('jsonwebtoken');

function generateToken(clientId) {
  return jwt.sign(
    { clientId },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );
}
```

### Verify JWT Middleware

```javascript
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Append to Google Sheets

```javascript
async function appendReading(sheet, reading) {
  await sheet.addRow({
    Timestamp: reading.timestamp,
    Inverter_kWh: reading.inverterKWh,
    Meter_Value: reading.meterValue,
    Source: reading.source || 'API',
    Note: reading.note || ''
  });
}
```

### Add Express Server

```javascript
const express = require('express');
const cors = require('cors');

const app = express();
app.use(express.json());
app.use(cors());

app.post('/auth/token', (req, res) => {
  const token = generateToken(req.body.clientId);
  res.json({ token, expiresIn: '7d' });
});

app.listen(process.env.PORT || 3000);
```

---

## cURL Quick Commands

```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:3000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"clientId":"mobile-app-id","clientSecret":"mobile-app-secret"}' \
  | jq -r '.token')

# Get current readings
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:3000/current_readings

# Add reading
curl -X POST http://localhost:3000/record \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "timestamp":"2025-11-07T14:30:00Z",
    "inverterKWh":5.42,
    "meterValue":142.3,
    "source":"mobile"
  }'

# Bulk add
curl -X POST http://localhost:3000/bulk_record \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "records":[
      {"timestamp":"2025-11-07T10:00:00Z","inverterKWh":3.21,"meterValue":140.1},
      {"timestamp":"2025-11-07T11:00:00Z","inverterKWh":4.12,"meterValue":141.0}
    ]
  }'
```

---

## Docker Quick Setup

```bash
# Build image
docker build -t solar-api .

# Run container
docker run -p 3000:3000 \
  -e GOOGLE_SHEET_ID=xxx \
  -e JWT_SECRET=secret \
  -v $(pwd)/credentials.json:/app/credentials.json \
  solar-api

# Using docker-compose
docker-compose up -d
```

---

## Deployment Options Comparison

| Platform | Cost | Setup Time | Pros | Cons |
|----------|------|-----------|------|------|
| Railway | $5-15/mo | 5 min | Simple, great UI | Limited customization |
| Render | $7+/mo | 5 min | Generous free tier | Cold starts |
| Fly.io | $1-20/mo | 10 min | Very affordable | More complex |
| AWS Lambda | $1-5/mo | 30 min | Pay-per-use | Cold starts, config complex |
| DigitalOcean | $5+/mo | 15 min | Full control | More DevOps knowledge |

---

## Testing Quick Commands

```bash
# Unit tests
npm test

# Integration tests
npm test -- --testPathPattern=integration

# With coverage
npm test -- --coverage

# Watch mode
npm test -- --watch

# Load testing (artillery)
npm install -g artillery
artillery run load-test.yml --target http://localhost:3000
```

---

## Common Errors & Fixes

### "Cannot find module 'google-spreadsheet'"
```bash
npm install google-spreadsheet
```

### "ENOENT: no such file or directory, open 'credentials.json'"
```bash
# Make sure credentials.json exists
# Check GOOGLE_CREDENTIALS_PATH in .env matches actual file location
```

### "Invalid client credentials" from Google
```bash
# Verify:
1. Service account email is shared in Google Sheets
2. credentials.json is valid JSON
3. Google Sheets API is enabled in Cloud Console
4. Spreadsheet ID is correct
```

### "401 Unauthorized" on protected endpoints
```bash
# Send token in Authorization header:
# Authorization: Bearer eyJhbGc...
```

### CORS errors in browser
```javascript
// Add to server.js:
app.use(cors({
  origin: ['http://localhost:3000', 'https://yourdomain.com'],
  credentials: true
}));
```

---

## Performance Tuning

### Reduce Google Sheets Calls
```javascript
// Cache results for 1 minute
const cachedData = {};
const CACHE_TTL = 60000;

async function getCurrent() {
  if (cachedData.timestamp && Date.now() - cachedData.timestamp < CACHE_TTL) {
    return cachedData.data;
  }
  
  const data = await sheetsService.getLastRow();
  cachedData.data = data;
  cachedData.timestamp = Date.now();
  return data;
}
```

### Batch Writes
```javascript
// Instead of one write per reading:
const readings = [];
for (let i = 0; i < 100; i++) {
  readings.push(newReading());
}
await sheetsService.appendBulkRows(readings);
```

### Database Indexes
```sql
CREATE INDEX idx_readings_timestamp ON readings(timestamp DESC);
CREATE INDEX idx_api_logs_created ON api_logs(created_at DESC);
```

---

## Security Checklist

- [ ] JWT_SECRET is 32+ random characters
- [ ] credentials.json is in .gitignore
- [ ] HTTPS enabled in production
- [ ] Rate limiting configured
- [ ] CORS whitelist set
- [ ] Input validation on all endpoints
- [ ] Helmet enabled (helmet middleware)
- [ ] Environment variables for all secrets
- [ ] Health check endpoint protected or public appropriately
- [ ] Error messages don't leak sensitive info

---

## Monitoring & Alerts

### Basic Health Check
```bash
curl http://localhost:3000/health
```

### Log Monitoring
```bash
# Watch logs in real-time
tail -f logs/combined.log | grep ERROR

# Count errors by endpoint
grep ERROR logs/combined.log | grep -oP '"endpoint":"\K[^"]+' | sort | uniq -c
```

### Response Time Tracking
```javascript
const start = Date.now();
// ... operation ...
const duration = Date.now() - start;
console.log(`Operation took ${duration}ms`);
```

---

## Development Workflow

```bash
# 1. Create feature branch
git checkout -b feature/new-endpoint

# 2. Make changes
# 3. Test locally
npm test

# 4. Commit
git add .
git commit -m "Add new endpoint"

# 5. Push to GitHub
git push origin feature/new-endpoint

# 6. Create Pull Request
# 7. Merge after review
# 8. Deploy to production
```

---

## Resources

**Official Docs:**
- Google Sheets API: https://developers.google.com/sheets/api
- Express.js: https://expressjs.com
- Node.js: https://nodejs.org/docs

**Libraries:**
- google-spreadsheet: https://github.com/theoephraim/node-google-spreadsheet
- jsonwebtoken: https://github.com/auth0/node-jsonwebtoken
- Puppeteer: https://pptr.dev

**Deployment:**
- Railway: https://railway.app
- Render: https://render.com
- Fly.io: https://fly.io

**Learning:**
- REST API Best Practices: https://restfulapi.net
- JWT Handbook: https://auth0.com/resources/ebooks/jwt-handbook
- Google Sheets Tips: https://support.google.com/docs

---

## Phone Support Endpoints

For mobile app integration:

```
Production: https://your-domain.com/api
Staging: https://staging-your-domain.com/api
Development: http://localhost:3000
```

**Mobile Client Headers:**
```
Content-Type: application/json
Authorization: Bearer <jwt-token>
User-Agent: YourApp/1.0 (iOS/Android)
X-Client-Version: 1.0
```

---

**Last Updated:** November 7, 2025
**Version:** 1.0

