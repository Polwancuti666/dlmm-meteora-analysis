# Souls.md — The Soul of Beauty & Shine POS-ERP

## Our Philosophy

> "Build systems that serve people, not the other way around."

---

## Who We Are

**Beauty & Shine** — A beauty salon business with 4 branches in Jabodetabek:
- **BSD** (Bintaro) — Flagship salon
- **HQ** (Head Office) — Central management
- **DPK** (Depok) — Growing branch
- **CBG** (Cibinong) — Community favorite

We believe technology should empower beauty professionals to focus on what they do best — making people feel beautiful.

---

## Our Technology Soul

### From Zero to Production

We built this system from scratch, learning every step:

1. **Day 1**: Started with a simple Excel spreadsheet for appointments
2. **Week 1**: Realized we needed a proper POS system
3. **Month 1**: Built our first FastAPI backend
4. **Month 2**: Added React frontend with real-time updates
5. **Month 3**: Expanded to full ERP with inventory, finance, reporting
6. **Month 4**: Deployed to production with Cloudflare Tunnel
7. **Today**: 44 sub-modules serving 4 branches

### The Tech Stack We Chose (and Why)

**FastAPI** — Because Python is readable, and async support means we handle multiple POS terminals without blocking.

**React + TypeScript** — Because type safety catches bugs before users do. No emojis in production — SVG icons only.

**PostgreSQL** — Because data integrity matters. UUID primary keys everywhere for distributed future.

**systemd (not Docker)** — Because on a 961MB VPS, every MB counts. Docker adds overhead we can't afford.

**nginx** — Because static files should be served by a web server, not a Python process.

**Cloudflare Tunnel** — Because we don't want to expose ports directly. Zero Trust security.

---

## Our Values

### 1. **Simplicity Over Cleverness**
```python
# Good: Clear, maintainable
def get_branch_id(branch_code: str) -> UUID:
    row = fetch_one("SELECT id FROM branch WHERE code = %s", (branch_code,))
    return row["id"] if row else None

# Bad: Clever but fragile
def get_branch_id(code): return UUID(code) if code else None
```

### 2. **Data Integrity Over Speed**
```python
# Always register document first (FK constraint)
execute("INSERT INTO document_registry (doc_key, ...) VALUES (%s, ...)", (doc_key,))
# Then insert dependent record
execute("INSERT INTO pos_transaction (doc_key, ...) VALUES (%s, ...)", (doc_key,))
```

### 3. **User Experience Over Developer Convenience**
```javascript
// Good: User sees current data
const now = new Date();
const today = now.getFullYear() + '-' + 
  String(now.getMonth() + 1).padStart(2, '0') + '-' + 
  String(now.getDate()).padStart(2, '0');

// Bad: Developer shortcut, wrong timezone
const today = new Date().toISOString().split('T')[0]; // UTC, not WIB!
```

### 4. **Fail Loudly, Not Silently**
```python
# Good: Clear error message
if not staff_row:
    raise HTTPException(404, f"Staff '{username}' not found")

# Bad: Silent failure
staff_id = staff_row["id"] if staff_row else None  # None → FK violation later
```

---

## Lessons Learned (The Hard Way)

### 1. **Memory Matters on Small VPS**
We learned the hard way that `npm run build` can OOM on 961MB RAM. Solution: Kill tsserver first, use `--minify false`.

### 2. **Timezone Hell**
`toISOString()` returns UTC. Our staff works in WIB (UTC+7). After midnight WIB, yesterday's data disappears. Fix: Always use local date methods.

### 3. **Cache is a Double-Edged Sword**
Service workers cache everything — including API responses. When we added new branches, old data showed. Fix: Bump `CACHE_NAME` version.

### 4. **UUID vs String IDs**
Frontend sends "KSR002" (username), backend expects UUID. Passing string to UUID column → `InvalidTextRepresentation`. Fix: Always resolve first.

### 5. **Docker vs systemd**
We had both running. Docker container had old code, systemd had new. Fixes didn't take effect until we restarted the right one.

### 6. **The In-Memory Cart Problem**
Carts stored in Python dict, not DB. Server restart → all carts lost. Fix: DB fallback for all checkout endpoints.

---

## Our Commitments

### To Our Staff
- **Fast checkout**: < 2 second response time
- **Reliable bookings**: Never lose a booking, even if server restarts
- **Clear errors**: Indonesian language, actionable messages
- **Offline-capable**: Service worker caches static assets

### To Our Developers
- **Clean code**: No emojis in production, proper types everywhere
- **Documented pitfalls**: Every gotcha has a solution documented
- **Testable**: Repository pattern makes unit testing possible
- **Deployable**: One-command build and deploy

### To Our Business
- **Audit trail**: Every transaction tracked via document_registry
- **Financial integrity**: Double-entry journal, debit = credit always
- **Branch isolation**: Each branch's data stays in its branch
- **Growth-ready**: UUID PKs, clean separation, ready for more branches

---

## The Future We're Building

### Phase 1 (Done) ✅
- POS system with checkout, booking, receipt
- Inventory management with stock cards
- Finance with journal entries
- Reporting with 8 report types
- Operations with 12 sub-modules

### Phase 2 (In Progress) 🔄
- Mobile app for therapists
- Customer loyalty program
- WhatsApp integration for reminders
- AI-powered scheduling

### Phase 3 (Planned) 📋
- Multi-tenant architecture
- Franchise management
- API marketplace for integrations
- Real-time analytics dashboard

---

## Our Team (The Bots)

### Hermes Agent
The orchestrator. Plans, delegates, reviews. Never writes code directly when a subagent can do it.

### Claude Code / Codex
The implementers. Build features, fix bugs, write tests. Follow the patterns Hermes establishes.

### GitHub Actions
The CI/CD. Runs tests, builds frontend, deploys to production.

---

## Contact & Support

- **GitHub**: https://github.com/Polwancuti666/pos-erp-v6
- **Domain**: https://beautynshine.web.id
- **Backup Repo**: https://github.com/Polwancuti666/pos-erp-backups

---

## Appendix: The DLMM Connection

While building the POS-ERP, we also explored Solana blockchain:

- **Meteora DLMM** — Dynamic Liquidity Market Maker
- **Pool tracking** — 5 pools monitored with real-time active_bin
- **CORS workaround** — Embed on-chain data server-side
- **Dashboard** — `vp-dryrun-compare.html` with simulated vs real comparison

The same principles apply: data integrity, user experience, and fail loudly.

---

*"We don't just build software. We build the foundation for beautiful experiences."*

— Beauty & Shine Tech Team, 2026
