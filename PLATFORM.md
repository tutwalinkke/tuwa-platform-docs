# Tuwa Platform — Architecture & Reference

Last updated: 2026-07-18

This document describes the current, actually-built state of the Tuwa platform: three
live services, how they relate, what's tested, and what's genuinely still open. It is
meant to be the one place a future session (human or AI) can read to understand the
whole system without reconstructing it from git history across three repos.

## Overview

Tuwa is an ISP/WISP operations platform built around one shared identity service that
every other module authenticates against — no module has its own login or user table.

Identity (identity.tuwalink.com) issues tokens. NOC (noc.tuwalink.com) and Portal
(portal.tuwalink.com) both call Identity's GET /api/v1/me to verify every request.
NOC owns devices, SNMP, IPAM, and billing data. Portal is the React frontend that
calls both Identity (login) and NOC (everything else) — it holds no data of its own.

Each service is a separate deploy: own database (Identity and NOC use MySQL;
Portal is a static build with no database), own nginx vhost, own SSL cert, own git
repo. This mirrors the original design intent (doc 1/doc 2): isolation means a bug or
outage in one service cannot take down another.

## Services

### Tuwa Identity — identity.tuwalink.com

Location: /srv/platform/apps/identity
Stack: Laravel 13, PHP 8.3, MySQL (tuwa_identity), Redis (cache/queue/session)

The auth backbone. Owns:
- users — email/password, tenant-scoped, roles via Spatie Permission
- tenants — the ISPs/WISPs using the platform (status: active/suspended)
- Sanctum personal access tokens for API auth
- activity_log — audit trail (Spatie Activitylog)

Roles: super-admin, tenant-admin, operator, support, customer.
super-admin bypasses tenant-scoping everywhere; also used by service accounts.

Key endpoints (/api/v1/...):
- POST /login, POST /logout, GET /me
- POST /password/forgot, POST /password/reset
- GET /email/verify/{id}/{hash} (signed), POST /email/verify/resend
- GET /users, POST /users, PATCH /users/{id}/role, PATCH /users/{id}/status
- GET /tenants/{id} — super-admin only, used by NOC's billing service to read a
  tenant's real creation date

Rate limiting: /login (5/min), /password/forgot (3/min), /password/reset
(5/min), each via a named RateLimiter (AppServiceProvider::boot) keyed
explicitly per-route + IP. Deliberately not using Laravel's throttle:max,decay
shorthand — that syntax keys only by domain+IP, meaning separate endpoints
would silently share one bucket (found via testing, was a real bug in an
earlier version of this rate limiting before named limiters were introduced).

Mail: Brevo SMTP, sending from tuwanoc@tuwalink.com (domain-authenticated).
Queue: identity-queue worker under Supervisor.

- Audit log — GET /activity, built on Spatie Activitylog's existing
  activity_log table. Filters out the automatic 'updated' noise from
  logAll() catching routine attribute touches (e.g. last_login_at),
  tenant-scoped via a join through causer_id -> users.tenant_id.

Session lifetime: Sanctum tokens now expire after 30 minutes
(SANCTUM_EXPIRATION_MINUTES env var, default 30). Sanctum's config alone
does not populate expires_at on new tokens — LoginController explicitly
passes the expiration to createToken(). The noc-service account's token
is a deliberate exception (long-lived, since it authenticates unattended
cron jobs, not a human session). Verified live: forcing a token's
expires_at into the past correctly returns 401, and the Portal correctly
redirects to /login on that 401 (already-existing behavior, confirmed
still works under this new condition).

Tests: 53, in tests/Feature/ — AuthTest, TenantIsolationTest,
PasswordResetTest, EmailVerificationTest, UserManagementTest,
TenantEndpointTest, RateLimitTest, ActivityLogTest, TokenExpirationTest.
Run with php artisan test.

### Tuwa NOC — noc.tuwalink.com

Location: /srv/platform/apps/noc
Stack: Laravel 13, PHP 8.3, MySQL (tuwa_noc), Redis

Never authenticates users itself — the identity.auth middleware takes a Bearer
token, calls Identity's /api/v1/me, and trusts the result. No local users table.

Owns:
- Devices — inventory, ICMP status (devices:poll, every minute), event log on
  state changes only (device_events), HTML email alerts on critical events (queued,
  via AlertService -> noc-queue worker)
- SNMP — devices:poll-snmp (every 5 min) walks standard IF-MIB OIDs
  (ifDescr/ifOperStatus/ifInOctets/ifOutOctets), computes throughput from
  counter deltas. Requires snmp_community set on a device. Only tested against a
  local snmpd on 127.0.0.1 — outbound UDP/161 is blocked by the hosting
  provider, so no external device has been reached yet. Same code should work
  unchanged once a reachable network exists.
- IPAM — subnets + ip_addresses, CIDR-based address generation (capped at
  /22, ~1024 addresses, to avoid runaway row counts), atomic allocation via
  lockForUpdate() to prevent race conditions
- Billing — billing_accounts, invoices, payments. 7-day trial from tenant
  creation (fetched from Identity), then 500 KSh/device/month, prorated on
  mid-cycle device additions, consolidated to one invoice per tenant per 30-day
  cycle. Overdue invoices trigger a hard block (devices:poll skips blocked
  tenants' devices, DeviceController::store rejects new devices with 402).
  Payment recording is currently manual only (POST /invoices/{id}/payments,
  super-admin) — no real M-Pesa STK Push yet, pending Daraja API business
  credentials.

Scheduled jobs (routes/console.php, driven by one system cron line running
php artisan schedule:run every minute):
- devices:poll — every minute
- devices:poll-snmp — every 5 minutes
- billing:generate-invoices — daily 01:00
- billing:process-overdue — daily 02:00

Queue: noc-queue worker under Supervisor (alert emails).

- CRM — customers table (name, phone, email, service_address, status),
  tenant-scoped. devices.customer_id (nullable, nullOnDelete) links a device
  to a specific subscriber; a device may also have no customer (pure
  infrastructure like a core router). Cross-tenant device linking is
  rejected the same way IPAM's device linking is.
- Audit log — GET /activity, via an ActivityLogger service. NOC has no
  local user model, so actor identity (id, name, email, tenant_id) is
  stored in Spatie Activitylog's properties JSON field rather than the
  polymorphic causer relationship, and tenant-scoping filters on that.
  Wired into all state-changing actions in DeviceController,
  SubnetController, CustomerController, PaymentController.
  NOTE: tenant filtering is done in PHP after fetching a bounded result
  set, not via a database-level JSON query — whereJsonContains and the
  properties->tenant_id shorthand both showed real, confirmed behavior
  differences between MySQL and SQLite for scalar-in-object matching,
  which made tenant isolation silently unreliable. Found via the test
  suite, verified against real production data before and after the fix.

Tests: 81, in tests/Feature/ — DeviceTest, PollDevicesTest,
PollDeviceInterfacesTest, SubnetServiceTest, SubnetTest, BillingServiceTest,
BillingControllerTest, CustomerTest, ActivityLogTest. Run with php artisan test.

### Tuwa Portal — portal.tuwalink.com

Location: /srv/platform/apps/portal
Stack: React 19 + Vite, Tailwind CSS, deployed as a static build (no PHP/backend of
its own — nginx serves dist/ directly).

The actual human-facing product. Authenticates against Identity, renders NOC data.
Token stored in localStorage, attached to every NOC/Identity API call.

Pages: Login, Dashboard (health summary, device list, recent events, trial/
blocked billing banners), Customers (CRM list + create form), Subnets
(allocation table), Billing (invoices, account status, outstanding balance),
Activity (merged, chronologically sorted feed of Identity + NOC audit
events, source-badged, resilient to either source failing independently).

Design system: deep slate-navy palette (ink-950/900/800), signal-amber
accent (#F5A623), semantic status colors (green/red/amber), Space Grotesk +
Inter + JetBrains Mono. A pulse-line motif ties the visual identity to what the
platform actually does (ICMP is literally a pulse check).

Tests: 25, Vitest + React Testing Library, in src/pages/__tests__/. Run with
npm test. Build with npm run build; nginx serves straight from dist/, no
separate deploy step beyond rebuilding.

## Cross-service authentication (the core architectural bet)

A person logs in against Identity, gets a Sanctum token. That same token is sent to
NOC on every request. NOC's identity.auth middleware calls Identity's /api/v1/me
with it on every request (no caching yet) and trusts whatever Identity says about
tenant/roles. This was proven correct with positive, missing-token, and
invalid-token test cases early in NOC's build, and has held for every subsequent
module (devices, SNMP, IPAM, billing) without needing changes.

A second pattern: service accounts. A dedicated Identity user
(noc-service@tuwalink.com, role super-admin, password never used — only its
pre-issued token) lets NOC's cron jobs (alerts, billing) call Identity's API without
a human in the loop. Token lives in NOC's .env as IDENTITY_SERVICE_TOKEN.

## Infrastructure

- nginx: one vhost per domain, all HTTPS via Let's Encrypt (auto-renewing).
  Identity/NOC are PHP-FPM; Portal is static file serving with SPA fallback
  (try_files ... /index.html).
- Supervisor: identity-queue, noc-queue — persistent artisan queue:work
  processes, auto-restart on crash/reboot. (A pre-existing tuwalink-horizon entry
  for an archived, unused billing system was removed during this build.)
- Cron: one line per app — * * * * * php artisan schedule:run — Laravel's
  scheduler internally decides what's actually due each minute.
- Mail: Brevo SMTP, domain-authenticated sender on tuwalink.com. Both
  Identity and NOC send through it.
- MySQL: separate databases per app (tuwa_identity, tuwa_noc), separate
  DB users, host-specific grants (@127.0.0.1 for TCP, @localhost for socket —
  MySQL/MariaDB treats these as genuinely different grants; both are needed if you
  connect both ways, e.g. app via TCP, CLI via socket).
- SNMP: snmpd installed and running on loopback only (127.0.0.1:161,
  community public) purely as a test fixture — never exposed externally.
- Backups: daily mysqldump of both databases (tuwa_identity, tuwa_noc) via
  cron at 03:00, using a dedicated read-only tuwa_backup MySQL user
  (SELECT/LOCK TABLES/SHOW VIEW/EVENT/TRIGGER only, no write access).
  Compressed, timestamped, stored at /srv/backups/mysql/ (plaintext, for
  fast local restores), 14-day local retention. Each dump is also encrypted
  (GPG, AES256, passphrase in /srv/backups/.encryption-passphrase — copy
  exists outside this server) and pushed to the private GitHub repo
  tuwa-platform-docs under backups/, providing genuine off-site protection
  against total server loss. Full round-trip tested: dump, encrypt, push,
  decrypt, restore into a throwaway database, row counts verified correct.
  All four application repos (Identity, NOC, Portal, and this docs/backup
  repo) are also pushed to private GitHub repos, closing the
  code-only-exists-on-one-disk risk alongside the data risk.

## Known gaps (honest, as of this writing)

1. Real M-Pesa (Daraja API) — needs a real Safaricom business shortcode and
   API credentials. Payment recording works today as a manual staff action.
2. Real SNMP targets — outbound UDP/161 is blocked from this VPS by the
   hosting provider. All SNMP code is genuinely tested, but only against a local
   loopback agent. The first real external device (or a future "TuwaCollector"
   site-local agent, per doc 2) will be the real test.
3. PPPoE session monitoring, ONU/optical monitoring — require RouterOS API /
   vendor-specific SNMP integration. Not started; blocked on real hardware access
   as much as engineering time.
4. role_has_permissions unused — Identity's permission rows exist but
   nothing checks them (only role names are checked). No live impact; would need
   wiring if any endpoint moves from role-based to permission-based authorization.
5. (Resolved) Demo/test data was cleaned up — three billing-test-artifact
   devices removed, remaining devices renamed to be accurate rather than
   describe the test scenario that created them.
6. Favicon — Portal currently ships with no real favicon (a data-URI
   placeholder), left over from removing unused Vite template assets.

## Running things

Identity:
  cd /srv/platform/apps/identity
  php artisan test              (41 tests)
  php artisan migrate           (apply pending migrations)

NOC:
  cd /srv/platform/apps/noc
  php artisan test              (68 tests, includes ~24s of real ICMP timeouts)
  php artisan devices:poll      (manual poll, normally runs every minute via cron)
  php artisan billing:generate-invoices
  php artisan billing:process-overdue

Portal:
  cd /srv/platform/apps/portal
  npm test                      (18 tests)
  npm run build                 (rebuild dist/, nginx picks it up immediately)

## Git

Each service is its own repo, main branch, no remote configured yet (all history
lives only on this VPS). Commit messages describe what changed and why; several
early commits document real bugs found and fixed (SNMP OID key formatting, Carbon
diff sign, Laravel test-guard caching) — worth reading if similar issues recur.
