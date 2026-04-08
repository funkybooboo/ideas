# OSS Funding Proxy — The Funding Layer for Software

## Vision

A **payment-enabled proxy layer** that sits transparently between developers and existing package repositories. Install software the normal way. Support its maintainers automatically.

> Not a new package manager. A new economy for the ones we already use.

---

## Problem

Open-source powers modern infrastructure, yet:

- Maintainers are underpaid or unpaid for critical work
- Existing funding tools (GitHub Sponsors, Open Collective, Patreon) are:
  - Manual opt-in
  - Disconnected from actual usage
  - Ignored by most companies despite heavy dependency on OSS
- Companies consume OSS freely—often legally required to do nothing more

The result: [log4shell](https://en.wikipedia.org/wiki/Log4Shell), [xz utils backdoor](https://en.wikipedia.org/wiki/XZ_Utils_backdoor), [left-pad](https://qz.com/646467/how-one-programmer-broke-the-internet-by-deleting-a-tiny-piece-of-code). Understaffed, burned-out maintainers are a systemic infrastructure risk.

### Why Existing Solutions Fall Short

| Solution | Problem |
|---|---|
| GitHub Sponsors | Voluntary, manual, not tied to usage |
| Open Collective | Donation-based, high friction |
| Tidelift | Requires maintainer signup + SLA work |
| thanks.dev | Analytics only, no payment enforcement |
| FOSSA / TLDR Legal | License compliance, not funding |

---

## Solution

A **proxy layer** that mirrors existing package registries and adds a funding transaction at install time—transparently, with no workflow change for the developer.

```
Before:  developer  →  npm registry  →  package
After:   developer  →  funding proxy  →  npm registry  →  package
                              ↓
                       micropayment recorded
                       funds queued for distribution
```

The proxy is **protocol-compliant** with existing package manager clients. No client-side changes required beyond pointing to a new registry URL.

---

## How It Works

### 1. Developer Setup (one-time)

```bash
# npm example
npm config set registry https://registry.fundingproxy.io

# pip example
pip config set global.index-url https://pypi.fundingproxy.io/simple/

# apt example (add to sources.list.d)
deb https://apt.fundingproxy.io/ubuntu jammy main
```

Fund a wallet (credit card, invoice, or crypto). Set a monthly budget.

### 2. At Install Time

```
npm install express
```

1. Request hits the proxy
2. Proxy resolves `express` + all transitive dependencies
3. Funding allocation calculated across the dependency tree
4. Package delivered unchanged (proxy caches upstream artifacts)
5. Payment event queued (not charged per-install; batched)

### 3. Distribution

- Weekly batch: wallet charges accumulated micropayments
- Attribution graph maps packages → maintainers → contributors
- Funds distributed via bank transfer, PayPal, or crypto wallet

---

## System Architecture

### Component Map

```
┌─────────────────────────────────────────────────────────┐
│  Developer / CI system                                  │
│  (standard package manager, zero code changes)          │
└───────────────────────┬─────────────────────────────────┘
                        │  HTTPS (native registry protocol)
┌───────────────────────▼─────────────────────────────────┐
│  Protocol Adapter Layer                                  │
│  npm-adapter / pip-adapter / apt-adapter / cargo-adapter│
│  - Speaks each registry's native protocol               │
│  - Proxies upstream; caches immutable artifacts         │
└───────────┬───────────────────────────┬─────────────────┘
            │                           │
┌───────────▼──────────┐   ┌────────────▼────────────────┐
│  Dependency Graph     │   │  Upstream Mirror Cache       │
│  Resolver             │   │  (CDN-backed, immutable)     │
│  - Builds full dep    │   │  - Packages never modified   │
│    tree per install   │   │  - SHA256 verified           │
│  - Weights by depth   │   └─────────────────────────────┘
└───────────┬───────────┘
            │
┌───────────▼──────────────────────────────────────────────┐
│  Attribution Engine                                       │
│  - Maps packages → maintainers → contributors             │
│  - Sources: package.json authors, CODEOWNERS, git log,    │
│    PyPI maintainers API, npm ownership API                │
│  - Configurable split rules (maintainer vs contributors)  │
└───────────┬───────────────────────────────────────────────┘
            │
┌───────────▼──────────────────────────────────────────────┐
│  Payment Engine                                           │
│  - Wallet balances (per user / per org)                   │
│  - Micropayment ledger (not per-transaction; batched)     │
│  - Policy engine: caps, priorities, exclusions            │
│  - Payout processor: Stripe Connect, PayPal Payouts       │
└──────────────────────────────────────────────────────────┘
```

---

## Attribution Engine (the hard part)

Mapping from "package downloaded" → "who gets paid" is non-trivial.

### Data Sources (per ecosystem)

| Ecosystem | Maintainer data | Contributor data |
|---|---|---|
| npm | `package.json` authors + owners API | GitHub repo commit graph |
| PyPI | Maintainers API | GitHub/GitLab commit graph |
| apt/deb | `debian/control` Maintainer field | `git log` on Salsa/GitHub |
| cargo | `Cargo.toml` authors | crates.io owners API |

### Allocation Strategy

Default: **weighted by dependency depth + download share**

```
express (direct dep)       → 40% of package's share
  accepts (depth 1)        → 20%
  finalhandler (depth 1)   → 20%
  ...transitive deps       → remaining 20%, split equally
```

Configurable per organization:
- Fund only direct dependencies
- Cap transitive depth at N
- Exclude packages with zero registered maintainers
- Prioritize packages with open CVEs or low contributor count

### Maintainer Registration

Maintainers must register to receive funds:
1. Claim ownership via OAuth (GitHub, GitLab, npm token, PyPI API key)
2. Add payout method
3. Unclaimed funds: held for 90 days, then redistributed or donated to a foundation (e.g., NLnet, Software Freedom Conservancy)

---

## Policy Engine

Organizations configure funding rules:

```yaml
# .fundingproxy.yaml
budget:
  monthly_usd: 500
  per_install_cap_usd: 0.10

allocations:
  depth_limit: 3          # only fund up to 3 hops of transitive deps
  min_payout_usd: 1.00    # ignore packages below this threshold
  direct_dep_weight: 0.5  # 50% of budget to direct deps

exclusions:
  - vendor/**             # skip vendored code
  - packages: [lodash]    # already sponsoring separately

priorities:
  - packages: [openssl, libssl-dev]
    weight: 2.0           # double allocation for security-critical deps
```

---

## Ecosystem-Specific Implementation

### npm / yarn / pnpm

- Registry protocol: simple HTTP JSON API (`/-/package`, `/package/-/package-version.tgz`)
- Proxy is a Fastify server implementing the npm registry spec
- Tarball delivery: proxy → verify SHA, stream from CDN → client
- Lockfile compatibility: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml` all use URLs that can point to the proxy

### pip / uv

- PEP 503 Simple API (`/simple/`, `/packages/`)
- PEP 658 metadata served unchanged
- Wheel and sdist delivered as-is; SHA256 verified
- Compatible with `uv`, `poetry`, `pdm`

### apt / dpkg

- Debian repository format: `Release`, `Packages.gz`, `*.deb`
- Proxy serves signed `Release` file (re-signed with proxy key + original key chain preserved)
- InRelease GPG verification requires distributing a proxy signing key

### cargo

- Sparse registry protocol (RFC 2789)
- Crate files served from `crates.io` CDN mirror
- `config.toml` `replace-with` points to proxy sparse index

---

## Business Model

| Tier | Target | Pricing | Features |
|---|---|---|---|
| Free | Individuals | $0 | Analytics only, no payments |
| Developer | Solo devs | $10/mo | $5 funding budget, basic reporting |
| Team | Small orgs | $99/mo + usage | $50 funding budget, org policy |
| Enterprise | Large companies | Custom | Private mirrors, compliance reports, audit logs, SSO |

Platform fee: **8% of funds distributed** (taken before maintainer payout)

Enterprise value-add: "OSS dependency funding report" for legal/compliance teams—demonstrates due diligence on OSS sustainability.

---

## Go-To-Market

### Phase 1 — npm MVP
- npm is the largest ecosystem by package count
- JSON API is well-documented and simple to proxy
- JavaScript developers are the most likely early adopters
- Build: proxy + payment engine + basic attribution

### Phase 2 — Maintainer Outreach
- Partner with prominent maintainers (Sindre Sorhus, TJ Holowaychuk-style accounts)
- Show real payout numbers; publish transparency reports
- Goal: 50 registered maintainers with real payout histories before launch

### Phase 3 — Org Sales
- Target mid-size companies (50–500 engineers) with heavy OSS usage
- Pitch: "fund your dependencies + get a compliance artifact for free"
- Integrate with procurement (PO-based billing)

### Phase 4 — Ecosystem Expansion
- pip (ML/data community has strong OSS values)
- cargo (Rust community cares about sustainability)
- apt (infrastructure teams, hardest sell but highest impact)

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| OSS community sees this as a paywall | Position as voluntary; free tier never blocks installs |
| Maintainers don't register | Hold funds publicly; pressure + incentive to claim |
| Proxy becomes a supply chain attack surface | Strict SHA256 verification; immutable artifact cache; no package modification |
| Large companies refuse to pay | Enterprise compliance angle; legal pressure as OSS funding expectations grow |
| Competing platforms emerge | Network effects; first-mover on attribution graph quality |
| Attribution disputes | Transparent on-chain ledger; dispute resolution process |

### Supply Chain Security (critical)

The proxy **must never modify packages**. Implementation:
- All upstream artifacts downloaded once, SHA256 verified against registry checksums
- Stored in immutable CDN (S3 Object Lock or equivalent)
- Proxy serves artifacts verbatim; no repackaging
- Independent reproducibility verification (anyone can check proxy SHA vs upstream)
- Open-source the proxy server; allow self-hosting for enterprises

---

## Legal Considerations

- Proxying public package registries is legally unambiguous (standard CDN/mirror behavior)
- No license modification; OSS licenses remain intact
- Payment is **voluntary funding**, not a fee for the software itself (which would conflict with permissive licenses)
- GDPR: usage data tied to org accounts; no personal data for anonymous installs
- npm, PyPI ToS: standard mirroring is permitted; check per-ecosystem terms for commercial proxies

---

## Open Questions

1. **What happens when a package has zero attribution data?** → Fund a foundation fallback (Apache, Linux Foundation, PSF) or pool for future attribution improvement.
2. **How to handle monorepos?** → `packages/*` treated as separate items; git blame at directory level.
3. **Can companies expense this?** → Yes; enterprise tier provides invoices coded as "software infrastructure."
4. **Self-hostable?** → Yes, especially for enterprises with private registries. Revenue model shifts to license fee.

---

## Future Vision

- **Funding graph becomes an industry standard:** companies report OSS funding alongside security audits
- **Maintainers earn sustainable income:** top maintainers of critical infrastructure make $50K–$200K/year passively
- **Dead package detection:** if a package has zero registered maintainers and high download count, the platform flags it as a critical risk and routes extra funding to encourage adoption/revival
- **Insurance product:** "OSS dependency insurance"—companies pay premiums, platform guarantees maintainer response SLAs
