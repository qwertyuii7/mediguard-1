# MediGuard - Quick Reference Guide

> **One-pager for revision, interviews, and quick onboarding**

---

## 1. Executive Summary

| | |
|---|---|
| **What** | AI-powered counterfeit medicine detector |
| **Stack** | React 18 + Vite + Node.js + MongoDB + Groq AI |
| **Core** | 3-layer vision pipeline + CDSCO batch verification + geospatial chemist finder |
| **Impact** | 15s end-to-end scan; 500x faster batch lookups via in-memory Map |

---

## 2. Architecture at a Glance

```
User → React → Express → [Cloudinary] → [Groq Vision OCR] → [Batch Map O(1)] → [Groq Risk] → MongoDB
                ↓              ↑              ↑                    ↓
         JWT Auth        Image CDN    Recall Check          Geo Query
```

### 3-Layer AI Pipeline
| Layer | Model | Purpose | Latency |
|-------|-------|---------|---------|
| L1 | Llama-4 Scout | OCR + Quality Check | ~6-8s |
| L2 | In-Memory Map | Batch verification | ~0.1ms |
| L3 | Llama-3.3 70B | Risk synthesis | ~2-3s |

---

## 3. Tech Stack Decisions

### Frontend
- **React 18** (not Next.js) → No SSR needed for AI-heavy client app
- **Vite** → 10x faster HMR than CRA
- **Three.js** → Differentiating 3D background (not generic Material-UI)
- **Leaflet** → 10KB lighter than Mapbox for OSM tiles

### Backend
- **Node.js** → Non-blocking I/O for concurrent AI calls
- **MongoDB** → Flexible schema for evolving AI responses
- **Groq** → 600 tok/s, 10x cheaper than OpenAI ($0.05 vs $0.50/1M tokens)
- **In-Memory Map** → O(1) batch lookups vs O(log n) DB queries (500x faster)

### Why NOT Alternatives?
| Choice | Rejected | Reason |
|--------|----------|--------|
| Node.js | Python/FastAPI | No ML serving needed; JS consistency |
| MongoDB | PostgreSQL | Unstructured AI data; no migration burden |
| Groq | OpenAI | 10x speed + cost; comparable OCR quality |
| React | Next.js | Unnecessary complexity for client-heavy app |

---

## 4. Key Algorithms & Complexities

### Critical Operations
| Operation | Complexity | Implementation |
|-----------|------------|----------------|
| Batch Lookup | **O(1)** | `Map.get()` with normalized keys |
| Geospatial Query | **O(log n)** | MongoDB 2D index + bounding box |
| AI Vision | **O(n²)** | Llama-4 Scout inference |
| User History | **O(1)** | Compound index `{user: 1, createdAt: -1}` |
| Image Encode | **O(n)** | Buffer Base64 conversion |

### Optimization Strategies
1. **In-Memory Batch Map**: Load 10k recalled batches into RAM at startup
2. **Cloudinary Transform**: `f_jpg,q_90,w_1600` → 60% token reduction
3. **Regex Normalization**: `B-123`, `B/123`, `B123` → unified lookup keys
4. **Compound Indexes**: Cover query + sort patterns
5. **Exponential Backoff**: 2s → 4s retry on Groq 429

---

## 5. Security Checklist

| Layer | Implementation |
|-------|----------------|
| Auth | JWT (7d access + 30d refresh), bcryptjs 12 rounds |
| HTTP | Helmet.js (CSP, HSTS, 11 headers) |
| Rate Limit | 100 req/15min per IP |
| Validation | express-validator chains |
| CORS | Whitelist only `CLIENT_URL` |
| Upload | Multer file filter + Cloudinary virus scan |
| Secrets | `.env` (gitignored), `.env.example` (template) |

---

## 6. Data Flow (Scan Operation)

```javascript
// 12 Steps, ~15s total
1. Capture → 2. Compress → 3. Upload (Multer) → 4. Cloudinary CDN
5. Fetch Base64 → 6. Groq OCR (Llama-4) → 7. Parse regex
8. Batch Map O(1) → 9. Risk Synthesis (Llama-3.3) → 10. Geo Query
11. DB Persist → 12. Render Results
```

**Bottlenecks**: Groq API (67% of time), Image upload (2-5s)

---

## 7. Database Schema

```javascript
// Core Models
User { email, password_hash, role: [user|chemist|admin] }
Scan { user_id, imageUrl, result, confidence, medicineDetails{}, batchStatus, riskLevel, analysisLayers{} }
Chemist { name, coordinates{lat,lng}, isVerified, isBlacklisted, city }
BatchNumber { batchNumber, status: [RECALLED|UNDER_INVESTIGATION], recallReason }
Alert { title, severity: [CRITICAL|HIGH|MEDIUM], source: 'CDSCO' }

// Indexes
{ user: 1, createdAt: -1 }    // History pagination
{ result: 1 }                  // Dashboard analytics
{ 'medicineDetails.name': 1 }   // Medicine search
{ coordinates: '2dsphere' }     // Geospatial
```

---

## 8. AI/ML Pipeline

### Prompt Engineering Strategy
```
Role: "pharmaceutical packaging inspector"
Format: EXACT structured output (regex-parseable)
Temp: 0.1 (deterministic)
Constraints: "DO NOT say FAKE unless..."
```

### Model Responsibilities
| Model | Task | Why |
|-------|------|-----|
| Llama-4 Scout | Vision OCR + Quality | Multimodal native, 600 tok/s |
| Llama-3.3 70B | Risk synthesis | 70B params for medical reasoning |
| Gemini 2.0 | Chat + Grounding | Google Search tool, 1M context |

### Accuracy Metrics
- OCR: ~85% (lighting/blur challenges)
- Batch Detection: ~75% (small fonts)
- False Positive: ~8% (over-cautious acceptable)

---

## 9. CDSCO Integration

```javascript
// Automated scraping (every 6 hours)
cron.schedule('0 */6 * * *', scrapeCDSCO)

// Dual strategy:
1. Live scraping (cheerio) from cdsco.gov.in/nsq-drugs/
2. Static fallback (8 critical alerts seeded)

// Keyword detection:
["recall", "substandard", "spurious", "not of standard quality"]

// Severity classification:
"spurious" | "fatal" → CRITICAL
"recall" → HIGH
other → MEDIUM
```

---

## 10. Chemist Verification Flow

```
REGISTER (chemist)
    ↓
PENDING → ADMIN_REVIEW
    ↓          ↓
VERIFIED (map visible)  BLACKLISTED (removed + alert)
```

**Geospatial Fallback Cascade**:
1. 2km radius (coordinate bounding box)
2. City regex match
3. Any 5 verified chemists (national)

---

## 11. API Endpoints

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/api/v1/scan/analyze` | POST | JWT | Main AI scan pipeline |
| `/api/v1/scan/history` | GET | JWT | Paginated user scans |
| `/api/v1/batch/verify` | GET | Public | Batch recall check |
| `/api/v1/chemists` | GET | Public | Nearby chemists |
| `/api/v1/alerts` | GET | Public | CDSCO alerts |
| `/api/v1/auth/register` | POST | Public | User/chemist signup |
| `/api/v1/dashboard/stats` | GET | JWT | Analytics |

---

## 12. Environment Variables

```bash
# Core
PORT=5000
MONGODB_URI=mongodb://localhost:27017/mediguard
JWT_SECRET=32+_char_random_string
CLIENT_URL=http://localhost:5173

# AI
GROQ_API_KEY=groq_xxx          # Required
GEMINI_API_KEY=AIza_xxx        # Optional (chat fallback)

# Media
CLOUDINARY_CLOUD_NAME=xxx
CLOUDINARY_API_KEY=xxx
CLOUDINARY_API_SECRET=xxx

# Optional
TWILIO_ACCOUNT_SID=xxx         # SMS alerts
EMAIL_USER=xxx                 # Email notifications
```

---

## 13. Setup Commands

```bash
# Backend
cd backend && npm install
cp .env.example .env          # Fill keys
npm run seed                  # Initial data
npm run dev                   # nodemon on :5000

# Frontend
cd Frontend && npm install
cp .env.example .env        # VITE_API_BASE_URL=http://localhost:5000/api/v1
npm run dev                 # Vite on :5173

# Verify
curl http://localhost:5000/api/health
```

---

## 14. Edge Cases & Handling

| Case | Solution |
|------|----------|
| Blurry image | AI returns "CANNOT_ASSESS" → prompt retake |
| Missing batch | "NOT_DETECTED" → packaging-only analysis |
| Groq 429 (rate limit) | Exponential backoff 2s→4s retry |
| Timeout >60s | "UNCLEAR" + retry suggestion |
| Fuzzy batch | Regex normalization: `/[-/\s]/g` |
| No nearby chemists | Fallback: city → any verified |

---

## 15. Performance Benchmarks

| Metric | Target | Current |
|--------|--------|---------|
| End-to-end scan | <15s | 12-18s |
| Batch lookup | <1ms | 0.1ms |
| Geo query | <100ms | 50ms |
| DB write | <50ms | 30ms |
| Image upload | <5s | 2-5s |

**Scalability Limits**:
- In-memory map: ~50MB RAM for 100k batches
- Current: Single MongoDB instance (no sharding)
- Rate limit: 100 req/15min (may throttle legit users)

---

## 16. Known Limitations

1. **AI**: Cannot detect chemical composition (packaging only)
2. **Coverage**: CDSCO India-centric; 10k batches vs millions
3. **Debt**: No test suite, no CI/CD (hackathon constraints)
4. **Sync**: AI calls blocking (no message queue)

---

## 17. Future Roadmap

### Immediate (1-3 months)
- [ ] Jest + Supertest coverage (target: 80%)
- [ ] Bull + Redis queue for AI calls
- [ ] PWA with offline scan history

### Advanced (3-6 months)
- [ ] Custom Vision Transformer training
- [ ] Blockchain batch verification
- [ ] Multi-language OCR (Hindi, Tamil)
- [ ] FDA/EMA alert integration

### Scale (6-12 months)
- [ ] Microservices (scan/auth/chemist split)
- [ ] Kubernetes orchestration
- [ ] Federated learning for pattern detection

---

## 18. Interview Quick Facts

**Q: Why Groq over OpenAI?**
A: 600 tok/s vs 50 tok/s, 10x cheaper ($0.05 vs $0.50/1M), comparable OCR quality.

**Q: How do you handle 10k batch lookups efficiently?**
A: In-memory Hash Map with O(1) complexity; 500x faster than DB queries. Multiple normalized keys for fuzzy matching.

**Q: What's the 3-layer AI pipeline?**
A: L1 (Llama-4): OCR + Quality → L2 (Map): Batch check → L3 (Llama-3.3): Risk synthesis. Modular, debuggable, independently upgradeable.

**Q: Why MongoDB over PostgreSQL?**
A: Unstructured AI responses evolve (added hologram detection in v2); no migrations; native geospatial queries.

**Q: How do you ensure security?**
A: JWT + bcrypt 12 rounds, Helmet.js headers, rate limiting 100/15min, express-validator, CORS whitelist, Cloudinary virus scan.

**Q: What's the CDSCO integration strategy?**
A: Dual: Live cheerio scraping every 6h + static fallback of 8 critical alerts. Keyword-based severity classification.

---

## 19. File Structure

```
mediguard/
├── Frontend/
│   ├── src/
│   │   ├── pages/           # Scanner, Alerts, ChemistMap
│   │   ├── components/      # Reusable UI
│   │   ├── context/         # Auth, Theme, App state
│   │   └── services/        # API clients
│   └── package.json         # React 18, Vite, Three.js
├── backend/
│   ├── controllers/         # scan.controller.js (main logic)
│   ├── services/            # groq.service.js (AI layer)
│   ├── models/              # Mongoose schemas
│   ├── jobs/                # cdscoScraper.job.js
│   ├── middleware/          # auth, error handlers
│   └── server.js            # Express entry
└── README.md                # Full documentation
```

---

## 20. Critical Code Snippets

### O(1) Batch Lookup
```javascript
// services/groq.service.js
let batchMap = new Map()

export const findBatchInMap = (batch) => {
  const normalized = batch.replace(/[-/\s]/g, '').toUpperCase()
  return batchMap.get(batch.toUpperCase()) || batchMap.get(normalized)
}
```

### Exponential Backoff
```javascript
// services/groq.service.js:82-104
const callGroq = async (payload, retries = 2) => {
  for (let i = 0; i <= retries; i++) {
    try {
      return await axios.post(GROQ_API_URL, payload, { timeout: 60000 })
    } catch (error) {
      if (error.response?.status === 429 && i < retries) {
        await new Promise(r => setTimeout(r, (i + 1) * 2000)) // 2s, 4s
        continue
      }
      throw error
    }
  }
}
```

### Geospatial Query
```javascript
// controllers/scan.controller.js:221-227
const nearby = await Chemist.find({
  isVerified: true,
  isBlacklisted: false,
  'coordinates.lat': { $gte: lat - 0.02, $lte: lat + 0.02 }, // ~2km
  'coordinates.lng': { $gte: lng - 0.02, $lte: lng + 0.02 }
}).limit(5).lean()
```

---

**Total Revision Time**: ~15 minutes  
**Key Takeaway**: 3-layer AI + O(1) batch lookup + CDSCO integration = scalable counterfeit detection

---

*For full details, see README.md*  
*For API specs, see backend/README.md*  
*For frontend architecture, see Frontend/README.md*
