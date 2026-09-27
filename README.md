# StockSense

**A C++-Powered Inventory Decision Engine for Small Retailers**

StockSense turns raw inventory and sales data into actionable decisions for small Indian retailers (kirana stores): what's about to expire, what's about to run out, and what to restock first — and why.

This is **not** a stock-market or investment tool. It is strictly a small-retailer inventory decision-support system.

---

## Problem Statement

Small retailers can usually tell you *how much* stock they have. They can rarely tell you:

- Which products are going to expire soon?
- Which products may run out soon?
- Which products should be restocked first — and why, specifically?
- Which products are fast-moving vs. slow-moving?

StockSense answers these questions with a real C++ algorithmic engine, not guesswork.

---

## Features

- **Product & Supplier Management** — categories, suppliers with lead times, minimum stock thresholds.
- **Batch-Based Inventory** — every purchase is a batch with its own expiry date, enabling batch-level (not just product-level) expiry tracking.
- **FEFO Sales Deduction** — recording a sale deducts stock from the soonest-expiring batch first (First-Expiry-First-Out), inside a transaction with row locking to prevent overselling under concurrent sales.
- **Expiry Analysis** — a min-heap-based engine classifies every batch as EXPIRED / CRITICAL / WARNING / SAFE and always surfaces the most urgent one first.
- **Stockout Estimation** — sales-velocity-based (a 7-day sliding window), explicitly *not* machine learning, with honest handling of zero-sales and zero-stock edge cases.
- **Restock Priority Engine** — a weighted, explainable 0–100 score combining stockout risk, minimum-stock risk, sales velocity, and supplier lead-time risk, with human-readable reasons attached to every score.
- **Dashboard, Alerts, Reports** — a React UI that surfaces all of the above.

---

## Architecture

```text
User (Retailer)
   ↓
React Frontend (Dashboard / Products / Inventory / Sales / Alerts / Reports)
   ↓ HTTPS / REST (JSON)
Node.js + Express Backend (auth, validation, MySQL queries, orchestration)
   ↓ SQL              ↓ JSON via stdin/stdout
MySQL Database    C++17 Decision Engine (short-lived subprocess per request)
                       ↓
                  DSA: unordered_map, priority_queue (min/max-heap), sort(), sliding window
```

The backend never re-implements the analysis logic — it gathers the relevant rows from MySQL, hands them to the compiled C++ binary as JSON on stdin, and relays the JSON it gets back on stdout. See [`docs/architecture.md`](docs/architecture.md) for the full request lifecycle and why a subprocess-per-request design was chosen over a persistent C++ service.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React, React Router, Recharts, Axios |
| Backend | Node.js, Express, mysql2, bcrypt, JWT |
| Database | MySQL 8 |
| Algorithm Engine | C++17, nlohmann/json (single header) |

---

## DSA Used (and why)

| DSA | Where | Why |
|---|---|---|
| `unordered_map<int, Product>` | `InventoryEngine` | O(1) average product lookup while scoring, instead of an O(n) scan per batch |
| `priority_queue` (min-heap) | `ExpiryAnalyzer` | Always-available O(1) peek at the single most urgent batch; O(log n) insert |
| `priority_queue` (max-heap, bounded) | `RestockEngine::topK` | Top-K retrieval for the dashboard without sorting the entire catalog: O(n log k) |
| `std::sort` | `RestockEngine::fullRanking`, sales reports | Full ranking reports genuinely need every item ordered — the right tool here, not a heap |
| Sliding window (`vector<int>`, size 7) | `StockoutAnalyzer` | Adaptive, honest velocity estimate without full time-series infrastructure |

Full complexity analysis and design rationale: [`docs/algorithms.md`](docs/algorithms.md).

---

## Project Structure

```text
stocksense/
├── frontend/       # React app
├── backend/        # Node.js + Express API
├── cpp-engine/     # C++17 decision engine (compiled to a standalone binary)
├── database/       # schema.sql, seed.sql
├── docs/           # SRS, architecture, algorithms, API docs, testing
└── screenshots/
```

---

## Installation & Running

### Prerequisites
- Node.js 18+
- MySQL 8+
- A C++17 compiler (g++ 9+ or clang 10+)
- CMake 3.10+ (optional — you can also compile directly with g++, see below)

### 1. Database

```bash
mysql -u root -p < database/schema.sql
mysql -u root -p < database/seed.sql
```

### 2. C++ Engine

```bash
cd cpp-engine
# Option A: CMake
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make

# Option B: direct g++ (no CMake required)
g++ -std=c++17 -O2 -Iinclude -o build/stocksense_engine \
    src/main.cpp src/InventoryEngine.cpp src/ExpiryAnalyzer.cpp \
    src/StockoutAnalyzer.cpp src/RestockEngine.cpp
```

Sanity-check it directly:
```bash
echo '{"mode":"all","products":[],"batches":[],"sales_summary":[]}' | ./build/stocksense_engine
```

### 3. Backend

```bash
cd backend
cp .env.example .env   # fill in your MySQL credentials and JWT secret
npm install
npm start               # or: npm run dev (nodemon)
```

### 4. Frontend

```bash
cd frontend
cp .env.example .env
npm install
npm start
```

Visit `http://localhost:3000`. Demo login: `demo@stocksense.local` — **note:** the seeded password hash corresponds to `password123`; change it immediately in any non-local environment.

---

## C++ ↔ Node.js Integration

The backend spawns the compiled `stocksense_engine` binary as a child process per analysis request, writes a JSON payload to its stdin, and reads a JSON response from its stdout. See [`backend/src/cppBridge/engine.js`](backend/src/cppBridge/engine.js) and [`docs/algorithms.md`](docs/algorithms.md) for the exact input/output contract.

---

## Testing

```bash
# C++ unit tests (assert-based, no external framework dependency)
cd cpp-engine
g++ -std=c++17 -Iinclude -o build/test_expiry tests/test_expiry.cpp src/ExpiryAnalyzer.cpp && ./build/test_expiry
g++ -std=c++17 -Iinclude -o build/test_stockout tests/test_stockout.cpp src/StockoutAnalyzer.cpp && ./build/test_stockout
g++ -std=c++17 -Iinclude -o build/test_restock tests/test_restock.cpp src/RestockEngine.cpp src/StockoutAnalyzer.cpp && ./build/test_restock
```

See [`docs/testing.md`](docs/testing.md) for the full edge-case matrix (expired batches, zero sales, zero stock, long supplier lead times, etc.).

---

## Documentation

- [`docs/SRS.md`](docs/SRS.md) — Software Requirements Specification
- [`docs/architecture.md`](docs/architecture.md) — full architecture & request lifecycle
- [`docs/database-design.md`](docs/database-design.md) — schema, relationships, design decisions
- [`docs/algorithms.md`](docs/algorithms.md) — DSA usage, complexity analysis, JSON contracts
- [`docs/api-documentation.md`](docs/api-documentation.md) — every REST endpoint
- [`docs/testing.md`](docs/testing.md) — test scenarios and edge cases

---

## Future Scope

- Role-based multi-user accounts (owner vs. staff)
- Real supplier integration (auto-generate purchase orders from restock priorities)
- Caching layer for dashboard analysis if catalog size grows large enough to matter
- Mobile-friendly / offline-first UI for low-connectivity areas
- Barcode scanning for batch entry and sales recording

---

## License

MIT — see [`LICENSE`](LICENSE).
