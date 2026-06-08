# ERP & POS Build Guide — From Zero to Production

## Project Overview
**Beauty & Shine POS-ERP** — Full-stack beauty salon management system.

- **Domain**: beautynshine.web.id
- **GitHub**: Polwancuti666/pos-erp-v6
- **Architecture**: FastAPI + React + PostgreSQL
- **4 Branches**: BSD (Bintaro), HQ (Head Office), DPK (Depok), CBG (Cibinong)

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | FastAPI (Python) + psycopg v3 |
| Frontend | React 18 + TypeScript + Vite 5 + Tailwind CSS |
| Database | PostgreSQL (Docker) |
| Auth | Custom HMAC-SHA256 JWT (no PyJWT) |
| Process Manager | systemd (`pos-erp.service`) |
| Web Server | nginx (reverse proxy + static files) |
| Tunnel | Cloudflare Zero Trust |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Browser                                        │
│  ├── /app/* → ERP (index.html)                  │
│  └── /pos/* → POS (pos.html)                    │
├─────────────────────────────────────────────────┤
│  Nginx                                          │
│  ├── /api/* → FastAPI (port 8000)               │
│  ├── /assets/* → Static files                   │
│  └── /* → SPA fallback                          │
├─────────────────────────────────────────────────┤
│  FastAPI (systemd managed)                      │
│  ├── pos_router_v2.py                           │
│  ├── checkout_router.py                         │
│  ├── inventory_router_v2.py                     │
│  ├── finance_router_v2.py                       │
│  └── closing_router.py                          │
├─────────────────────────────────────────────────┤
│  PostgreSQL (Docker)                            │
│  └── Tables: app_user, branch, treatment,       │
│      product, pos_transaction, etc.             │
└─────────────────────────────────────────────────┘
```

---

## Module Structure (44 sub-modules)

### ERP Modules
1. **Dashboard** (1) — KPI & monitoring
2. **Master Data** (12) — Treatments, Products, Branches, Users, Customers, COA, Vouchers, Promos, Categories
3. **Inventory** (5) — Stock Card, Batches, BOM, Low Stock, Alerts
4. **Finance** (5) — Journal, Trial Balance, AP, Bank, Assets
5. **Accounting** (2) — COA Upload, COA Management
6. **Reporting** (8) — Sales, Treatment, Payment, Therapist, Commission, Inventory, Finance, Shift
7. **Operations** (12) — Production, Bank Recon, Cost Center, Schedule, Certification, Pricelist, Cancel Reason, Recurring Journal, Cash Flow, WhatsApp, Executive, Settlement

### POS Modules
8. **POS System** — Home, Kasir, Booking, Checkout, Receipt, Shift, Voucher, Treatment Record

---

## Branch UUIDs

| Branch | UUID | Code |
|--------|------|------|
| HQ | `738401e4-...` | HQ |
| BSD (Bintaro) | `fbd70198-...` | BSD |
| DPK (Depok) | `6548cf5b-...` | DPK |
| CBG (Cibinong) | `8e72d6df-...` | CBG |

---

## Development Workflow

### Step 1: Database Schema
```python
from pos_erp.db import execute, fetch_all

execute('''CREATE TABLE IF NOT EXISTS table_name (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    branch_id UUID REFERENCES branch(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
)''')
```

**CRITICAL**: Check PK types before creating FK! Some tables use UUID, some use SERIAL (INT).

### Step 2: Backend Router
```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field
from pos_erp.db import fetch_all, fetch_one, execute, execute_returning

router = APIRouter(prefix="/api/module", tags=["Module"])

class CreateReq(BaseModel):
    name: str
    branch_code: str = Field(..., alias="branchCode")
    model_config = {"populate_by_name": True}

@router.get("")
def list_items():
    return {"items": fetch_all("SELECT * FROM table ORDER BY created_at DESC")}

@router.post("")
def create_item(req: CreateReq):
    return execute_returning(
        "INSERT INTO table (name, branch_code) VALUES (%s,%s) RETURNING *",
        (req.name, req.branch_code)
    )
```

### Step 3: Register Router
```python
# In fastapi_app.py
from pos_erp.routers.new_router import router as new_router
app.include_router(new_router)
```

### Step 4: Frontend API Client
```typescript
// In client.ts
getItems: () => fetchJSON('/module/items'),
createItem: (data: any) => fetchJSON('/module/items', { 
  method: 'POST', body: JSON.stringify(data) 
}),
```

### Step 5: Frontend Page
```tsx
import { useState, useEffect } from 'react';
import { fetchJSON } from '../api/client';
import Icon from '../components/Icon';

export default function ModulePage() {
  const [data, setData] = useState<any[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchJSON('/api/module')
      .then(res => setData(res.items || []))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Loading...</div>;
  return (
    <div>
      <h1><Icon name="master" size={24} /> Module Name</h1>
      {/* Table/form content */}
    </div>
  );
}
```

### Step 6: Build & Deploy
```bash
# Kill memory-heavy processes first
pkill -f tsserver; pkill -f typescript-language-server

# Build frontend
cd /root/pos-erp-v6/frontend
NODE_OPTIONS="--max-old-space-size=512" node node_modules/vite/bin/vite.js build --minify false

# Deploy
cp -r dist/* /var/www/bsn-erp/
cp -r dist/* /var/www/pos.beautyandshine.com/
systemctl restart pos-erp
systemctl reload nginx
```

---

## Common Pitfalls & Solutions

### 1. CORS Issues with Local Files
**Problem**: Browser blocks API calls from `file://` origin.
**Solution**: Embed data server-side in HTML.

### 2. Systemd vs Docker Confusion
**Problem**: Editing files on host but Docker container has different code.
**Solution**: Always use `systemctl restart pos-erp` (NOT `docker restart`).

### 3. Frontend Date Filter (UTC vs WIB)
**Problem**: `toISOString()` returns UTC, not local timezone.
**Solution**:
```javascript
const now = new Date();
const today = now.getFullYear() + '-' + 
  String(now.getMonth() + 1).padStart(2, '0') + '-' + 
  String(now.getDate()).padStart(2, '0');
```

### 4. In-Memory Cart Lost on Restart
**Problem**: `_carts` dict resets when server restarts.
**Solution**: Add DB fallback in checkout endpoints.

### 5. UUID vs String Staff IDs
**Problem**: Frontend sends username strings, DB expects UUID.
**Solution**: Resolve before insert:
```python
staff_row = fetch_one("SELECT id FROM app_user WHERE username = %s", (req.therapist_id,))
staff_uuid = staff_row["id"] if staff_row else None
```

### 6. Service Worker Stale Cache
**Problem**: Old JS served from cache after deploy.
**Solution**: Bump `CACHE_NAME` in `sw.js`.

### 7. Vite Build OOM on Low-Memory VPS
**Problem**: Build hangs on 961MB RAM server.
**Solution**: Kill tsserver, use `--minify false`.

### 8. Branch Context Mismatch
**Problem**: Staff.branch vs pos_branch_id from localStorage diverge.
**Solution**: Always send `pos_branch_id` from localStorage.

### 9. Document Registry FK Violation
**Problem**: Insert into dependent table before registering doc_key.
**Solution**: Register in `document_registry` FIRST.

### 10. Pydantic Alias for camelCase
**Problem**: Frontend sends camelCase, backend expects snake_case.
**Solution**:
```python
class Request(BaseModel):
    branch_code: str = Field(..., alias="branchCode")
    model_config = {"populate_by_name": True}
```

---

## Key File Locations

| File | Purpose |
|------|---------|
| `src/pos_erp/fastapi_app.py` | Main app, router registration |
| `src/pos_erp/routers/pos_router_v2.py` | POS transactions |
| `src/pos_erp/routers/checkout_router.py` | Checkout flow |
| `src/pos_erp/db.py` | PostgreSQL connection |
| `frontend/src/api/client.ts` | API client |
| `frontend/src/components/Icon.tsx` | SVG icon component |
| `frontend/src/components/Layout.tsx` | ERP sidebar layout |
| `frontend/src/components/PosLayout.tsx` | POS layout |
| `/etc/systemd/system/pos-erp.service` | Systemd service |
| `/etc/nginx/sites-enabled/` | Nginx configs |

---

## Database Tables

| Table | PK Type | Description |
|-------|---------|-------------|
| `app_user` | UUID | Staff accounts |
| `branch` | UUID | Branch locations |
| `treatment` | UUID | Services/treatments |
| `product` | UUID | Products |
| `pos_transaction` | UUID | POS transactions |
| `pos_transaction_item` | UUID | Transaction line items |
| `pos_daily_closing` | UUID | Daily closing records |
| `document_registry` | TEXT | Document key tracking |
| `chart_of_account` | UUID | Chart of accounts |
| `journal_entry` | UUID | Journal entries |

---

## Deployment Checklist

1. ✅ Database schema created
2. ✅ Backend router registered
3. ✅ Frontend page created
4. ✅ API methods added to client.ts
5. ✅ Routes added to App.tsx/PosApp.tsx
6. ✅ Navigation added to Layout.tsx
7. ✅ Build frontend (vite build)
8. ✅ Copy dist to nginx roots
9. ✅ Restart backend (systemd)
10. ✅ Reload nginx
11. ✅ Verify health endpoint

---

## References

- `pos-erp-v6-development` — Full development workflow
- `pos-beauty-shine` — Deployment & debugging
- `domain-driven-fastapi-erp` — Architecture patterns
- `pos-erp-module-development` — Adding new modules
