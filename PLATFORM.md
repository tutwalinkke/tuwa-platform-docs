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

Visual redesign (2026-07-20): moved from a flat stat-card grid to a denser
instrument-panel layout, inspired by (not copied from) Grafana-style
monitoring dashboards — radial SVG health gauge, a real bandwidth history
chart (recharts) backed by genuinely stored SNMP data via a new
GET /dashboard/bandwidth-history NOC endpoint, and a vertical icon sidebar
(lucide-react) replacing the original horizontal top nav. Palette deepened
(ink-950 #0A0F1A, more saturated status colors for chart legibility) but
kept the same named tokens and type system established at first build —
this was a density/maturity pass, not a rebrand.

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
- Security headers: /etc/nginx/snippets/security-headers.conf, included
  in all three vhosts (identity/noc/portal) right after Certbot's
  ssl_dhparam directive. HSTS (1yr, includeSubDomains), X-Frame-Options
  (SAMEORIGIN), X-Content-Type-Options (nosniff), Referrer-Policy
  (strict-origin-when-cross-origin), X-XSS-Protection.
- Content-Security-Policy: /etc/nginx/snippets/csp-portal.conf, Portal
  only (Identity/NOC are pure JSON APIs with no HTML to protect this
  way). Built from an actual audit of index.html and api.js: script-src
  'self' with no unsafe-inline/unsafe-eval (the real XSS protection),
  style-src allows 'unsafe-inline' + fonts.googleapis.com (Tailwind's
  compiled CSS and inline SVG styles need this), font-src allows
  fonts.gstatic.com, connect-src allows identity.tuwalink.com and
  noc.tuwalink.com (the two APIs the SPA actually calls). Verified live
  in a real browser across all six Portal pages with DevTools console
  open — zero CSP violations. Nginx configs themselves aren't in any
  git repo (they're server config, not app code) — this document is
  their only record; if this server is ever rebuilt, these snippet
  files need to be recreated from the content documented here.
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

## Platform self-monitoring

/srv/monitoring/check-uptime.sh — runs every 5 minutes via cron, independent
of all three app services (so it keeps working even if all three are down
simultaneously). Checks each service's health endpoint (Identity /up, NOC
/up, Portal /), tracks state transitions in /srv/monitoring/state/*.state,
and emails an alert (via raw curl SMTP against Brevo, reusing the same
credentials as Identity/NOC's own mail sending) only on actual up<->down
transitions, not on every check — otherwise a prolonged outage would spam
an email every 5 minutes. Credentials in /srv/monitoring/.mail-credentials
(chmod 600). Tested with genuine failing-then-passing email delivery,
including catching and fixing a real bug where \n in a shell variable
does not become a literal newline the way it does in a printf format
string — fixed by using real multi-line variable assignment instead of
escape sequences.

## Staging environment

A full, isolated second copy of the platform for safe testing, separate from
production in every way that matters:

- Domains: staging-identity.tuwalink.com, staging-noc.tuwalink.com,
  staging-portal.tuwalink.com (DNS A records added manually via Bluehost,
  pointing at this same server's IP)
- Databases: tuwa_identity_staging, tuwa_noc_staging (separate MySQL users
  too, not just separate schemas)
- Codebases: /srv/platform/apps/identity-staging, noc-staging,
  portal-staging — fresh git clones from the same GitHub repos as
  production, not copies of the production directories. This doubled as a
  real test that the GitHub history is genuinely complete and
  self-sufficient (all three cloned, installed, migrated, and passed their
  full test suites with zero production-specific fixes needed).
- Staging NOC points to staging Identity (not production Identity) for
  cross-service auth — genuinely isolated, not silently depending on prod.
  Verified live: logged in against staging Identity, called staging NOC's
  /ping, got back the correct staging identity — not production's.
  Staging has its own service account (noc-service@staging.tuwalink.com)
  with its own token, separate from production's.
- Mail: MAIL_MAILER=log in both staging apps — deliberately never sends
  real email during testing, regardless of what SMTP credentials exist
  elsewhere on the server.
- No cron, no Supervisor queue workers for staging — QUEUE_CONNECTION=sync
  means jobs run inline, no background worker needed. Billing/polling
  commands can be run manually when actually testing those flows.
- Login: admin@staging.tuwalink.com / StagingAdmin2026!

### Real bug found while setting this up

Fresh git clones are owned solely by the tuwalink user; PHP-FPM runs as
www-data, which is not a member of tuwalink's group. This meant Laravel
could not write compiled Blade views (storage/framework/views), causing a
templnam(): file created in the system's temporary directory error on
every page load — a real, confirmed production-parity gap between "clone
the repo" and "the app actually runs." Fixed by:
  sudo chown -R tuwalink:www-data <app>/storage <app>/bootstrap/cache
  sudo chmod -R 775 <app>/storage <app>/bootstrap/cache
This same fix will be needed for any future fresh deployment of any of
these apps, including if this server is ever rebuilt — worth doing
proactively as a standard post-clone step, not discovering it again.

## Staging environment

A full, isolated second copy of the platform for safe testing, separate from
production in every way that matters:

- Domains: staging-identity.tuwalink.com, staging-noc.tuwalink.com,
  staging-portal.tuwalink.com (DNS A records added manually via Bluehost,
  pointing at this same server's IP)
- Databases: tuwa_identity_staging, tuwa_noc_staging (separate MySQL users
  too, not just separate schemas)
- Codebases: /srv/platform/apps/identity-staging, noc-staging,
  portal-staging — fresh git clones from the same GitHub repos as
  production, not copies of the production directories. This doubled as a
  real test that the GitHub history is genuinely complete and
  self-sufficient (all three cloned, installed, migrated, and passed their
  full test suites with zero production-specific fixes needed).
- Staging NOC points to staging Identity (not production Identity) for
  cross-service auth — genuinely isolated, not silently depending on prod.
  Verified live: logged in against staging Identity, called staging NOC's
  /ping, got back the correct staging identity — not production's.
  Staging has its own service account (noc-service@staging.tuwalink.com)
  with its own token, separate from production's.
- Mail: MAIL_MAILER=log in both staging apps — deliberately never sends
  real email during testing, regardless of what SMTP credentials exist
  elsewhere on the server.
- No cron, no Supervisor queue workers for staging — QUEUE_CONNECTION=sync
  means jobs run inline, no background worker needed. Billing/polling
  commands can be run manually when actually testing those flows.
- Login: admin@staging.tuwalink.com / StagingAdmin2026!

### Real bug found while setting this up

Fresh git clones are owned solely by the tuwalink user; PHP-FPM runs as
www-data, which is not a member of tuwalink's group. This meant Laravel
could not write compiled Blade views (storage/framework/views), causing a
templnam(): file created in the system's temporary directory error on
every page load — a real, confirmed production-parity gap between "clone
the repo" and "the app actually runs." Fixed by:
  sudo chown -R tuwalink:www-data <app>/storage <app>/bootstrap/cache
  sudo chmod -R 775 <app>/storage <app>/bootstrap/cache
This same fix will be needed for any future fresh deployment of any of
these apps, including if this server is ever rebuilt — worth doing
proactively as a standard post-clone step, not discovering it again.

## OS-level security (UFW, Fail2Ban)

Both were already installed and active from initial server provisioning,
before this session began — not something built during this project, but
verified and hardened here.

UFW: default-deny incoming, allowing only 22 (SSH), 80/443 (HTTP/HTTPS),
51820/udp (WireGuard, powering the wg0 interface visible in our own SNMP
data). Nothing unexpected open.

Fail2Ban: watching sshd since server provisioning. As of this session,
had recorded 92,365 failed SSH login attempts and issued 10,310 bans over
roughly two months — this server is under continuous, real brute-force
attack, not a hypothetical risk. Default policy (5 attempts / 10min ->
10min ban) was genuinely too weak against a sustained attacker who can
simply wait out a 10-minute ban. Hardened via /etc/fail2ban/jail.local:
bantime raised to 1h, with escalating bans (bantime.increment, factor 2,
capped at 24h) for repeat offenders. Config change verified safe with
fail2ban-client -t before restart, and the restart itself verified safe
by opening a genuinely fresh SSH connection afterward — the one place
in this whole session where a mistake could have caused an unrecoverable
lockout, so this was checked deliberately rather than assumed.

## Two-factor authentication (TOTP)

Optional, per-user, built on pragmarx/google2fa-laravel. Encrypted secret
and recovery codes (Eloquent 'encrypted' cast, layered on top of APP_KEY).
Two-step login: password success with 2FA enabled returns a short-lived
(5min), restricted-ability Sanctum token ('2fa-pending' only) instead of a
real session token — a real token is only issued after a valid TOTP code
or one-time recovery code (8 generated at enable time) is verified.

Two real bugs found and fixed via live testing before this shipped:
1. This app's auth:sanctum middleware alias is non-standard (aliased to
   EnsureFrontendRequestsAreStateful rather than the usual parameterized
   Authenticate::class pattern) — Laravel's built-in 'abilities'
   middleware alias was also simply missing entirely, causing a 500 on
   the verify endpoint and silently failing to enforce the pending-token
   restriction (confirmed live: a pending token could hit /me and get
   200 OK, when it should be 401).
2. After adding custom middleware to work around that, a second bug:
   Sanctum's Token::can() treats a '*' wildcard ability (which every
   normal, fully-privileged token has) as satisfying ANY ability check,
   including can('2fa-pending') — so real, legitimate session tokens
   were being incorrectly rejected everywhere. Fixed by checking the
   token's raw ability array directly instead of going through can().
   A dedicated regression test (test_a_normal_fully_privileged_token_is_
   not_incorrectly_rejected) exists specifically to catch this again.

Portal UI: Settings page (QR code via qrcode.react, manual-entry key
fallback, recovery codes shown once at confirm time) and a two-step
Login page. Verified fully working end-to-end in a real browser through
every state — enable, confirm, login-with-2FA, recovery code redemption
(consumed correctly, cannot be reused), and disable (requires password
re-confirmation).

13 backend tests (TwoFactorAuthTest), all passing alongside the full
existing 66-test Identity suite with zero regressions.

## Bandwidth threshold alerting

Optional, per-device, per-direction (in/out) configurable thresholds
(devices.alert_threshold_in_bps / alert_threshold_out_bps, nullable —
no threshold set means no checking). Checked during the existing
devices:poll-snmp cycle, against the device's total throughput summed
across all its interfaces (matching how the Dashboard already presents
bandwidth). State-transition tracking (via DeviceEvent, same table
up/down status changes use) avoids repeat alerts during a sustained
breach and distinguishes a real breach (warning severity + email) from
recovery (info severity, no email).

### Two real, previously-invisible production bugs found while verifying this live

1. DeviceEvent ordering ambiguity: two threshold events created within
   the same second (a real possibility, since created_at is set
   manually via now() rather than relying on database auto-increment
   timestamp precision) could not be reliably ordered by created_at
   alone, causing the breach/recovery state machine to occasionally
   read stale state. Fixed by adding id as an explicit tiebreaker.

2. Alert recipient scoping: AlertService::getResponsibleUsers() sends
   to every active tenant-admin/super-admin — which includes NOC's own
   service account (noc-service@tuwalink.com), since it holds
   super-admin for API access purposes. That address is not a real
   monitored inbox, so Brevo has been silently failing ("Deferred,
   connection timeout") to deliver every device-down and bandwidth
   alert sent to it since the account was created — a real regression
   affecting production alerting that had been present and unnoticed
   for days, only surfaced by deliberately checking Brevo's delivery
   logs rather than assuming "no error in our own logs" meant success.
   Fixed via a new config('services.identity.alert_excluded_emails')
   list (ALERT_EXCLUDED_EMAILS env var, defaults to the known service
   account) with a dedicated regression test
   (test_service_account_is_excluded_from_alert_recipients).

Also discovered and fixed in the same investigation: the NOC service
account's Sanctum token had expires_at = NULL, which — after this
session's earlier token-expiration hardening work — meant Sanctum's
global age-based expiration policy was silently applying to it too
(that policy only applies when expires_at is null), causing
authenticate-to-Identity failures that broke the entire alert pipeline
(not just bandwidth alerts — device-down alerts too) starting from
whenever that policy shipped. Fixed by reissuing the service token with
an explicit, far-future expires_at, which exempts it from the global
policy entirely. Worth remembering for any future service account: it
must always be created with an explicit expiration, never left as
implicit-null under a codebase that also has a global age-based policy.

Real login account note: admin@tuwalink.com was renamed to
tuwalink@gmail.com (a real, monitored inbox) at the person's request,
since alerts need to reach somewhere they'll actually see them. This is
also now the login email for that account.

Verified live end-to-end: real threshold set on a real device, real
SNMP poll cycle correctly detected and reported the breach, real event
logged, real email correctly delivered to a real inbox — confirmed via
screenshot, not assumed from logs alone.

6 new backend tests (threshold logic) + 1 new regression test (alert
exclusion), full NOC suite passing at 93 tests.

## Maintenance windows

Per-device scheduled downtime (maintenance_windows table: device_id,
starts_at, ends_at, reason). Device::isInMaintenance() checks for an
active window right now. Both PollDevices (device up/down) and
PollDeviceInterfaces (bandwidth thresholds) check this before sending
an alert email — but never before logging the event. During a window,
what would normally be a critical/warning severity event is logged as
info instead, with the message explicitly noting it happened during
scheduled maintenance. This means a full, honest historical record
always exists (a device did go down, or bandwidth genuinely was over
threshold) — maintenance mode changes how urgently that gets treated,
never whether it gets recorded at all.

CRUD API (POST to schedule, POST .../end-early to close a window before
its planned end without deleting the historical record, DELETE to
remove entirely), tenant-scoped, activity-logged, consistent with every
other resource in this platform.

Verified live: scheduled a real window on the real SNMP test device,
forced a real bandwidth breach during it, confirmed via direct database
query that the event was logged at info severity with the maintenance
context in its message, and that zero mail jobs were queued (not just
delayed — genuinely never sent).

9 new tests (MaintenanceWindowTest): API CRUD, tenant isolation,
Device::isInMaintenance() correctness (including the boundary case of
a future-scheduled window not counting as active yet), and suppression
behavior for both PollDevices and the bandwidth threshold checker.

## Incident tracking and escalation

Every critical or warning severity DeviceEvent automatically creates an
Incident (IncidentService::maybeCreateFromEvent) — open -> acknowledged
-> resolved. Maintenance-suppressed events never create an incident,
for free, since they're already downgraded to info severity before
IncidentService ever sees them (no separate maintenance-check needed
here — this was a deliberate design choice to avoid duplicating that
logic in a second place).

CheckIncidentEscalation command, scheduled every 5 minutes: finds
incidents still open and unacknowledged past a configurable threshold
(--minutes, default 30) and sends a second, more urgent email
(IncidentEscalationAlert) via the same AlertService/recipient-scoping
pipeline as every other alert. escalated_at is set once escalation
fires, so the same incident is never escalated twice even though the
check runs repeatedly.

API: GET /incidents (tenant-scoped, optional ?status= filter), POST
.../acknowledge, POST .../resolve — both activity-logged, both reject
an incident that's not in the expected state (can't acknowledge an
already-acknowledged incident, can't resolve an already-resolved one).

Verified live end-to-end on real production data: forced a genuine
device state transition, confirmed a real Incident was created linked
to the real triggering DeviceEvent, acknowledged it via the real API
with real user attribution, confirmed the escalation command correctly
skipped it once acknowledged (tested with --minutes=0 to rule out any
timing coincidence), then resolved it — every step confirmed via actual
API responses, not assumed from passing tests alone.

11 new tests (IncidentTest): IncidentService creation rules, API CRUD
and tenant scoping, and escalation command behavior (fires past
threshold, doesn't fire within threshold, doesn't fire for acknowledged
incidents, never fires twice for the same incident).

Portal UI: Incidents page (src/pages/Incidents.jsx), status-filtered
tabs (All/Open/Acknowledged/Resolved), acknowledge/resolve buttons,
sidebar nav entry. Verified live end-to-end with real production data:
forced a genuine device state transition, confirmed the fresh incident
appeared correctly in the real UI (not just the API), acknowledged and
resolved it through real button clicks, confirmed both actions
persisted correctly and moved the incident between tabs. 5 new frontend
tests, full Portal suite at 30 tests.

## Network topology

Manually-declared topology (device_links table: device_a_id, device_b_id,
tenant_id, link_type, description) with real live status shown on it —
deliberately NOT automatic discovery. Real LLDP/CDP/ARP-based discovery
needs actual protocol access to real hardware, which this environment
does not have (same underlying constraint as SNMP/SSH device monitoring
generally). An operator declares which devices connect to which; the
map then shows genuine current device status on that structure.

Link normalization: the lower device ID is always stored as
device_a_id regardless of which order the API caller declares them in,
so a link and its reverse-order duplicate are recognized as the same
connection — genuinely prevented by a real unique constraint, not just
application-level convention. Verified by dedicated tests declaring
the same link in both directions.

API: GET /topology (combined devices + links, tenant-scoped), POST
/topology/links (create, rejects self-links and cross-tenant links),
DELETE /topology/links/{id}.

Portal UI: Topology page, SVG diagram with circular auto-layout (no
saved per-device positions — recalculated fresh each render), nodes
colored by real live device status (green/red/gray), lines between
linked devices labeled with link type, a declared-links list below
with remove actions, an add-link form.

Verified live end-to-end with real production data: linked two real
devices, confirmed the combined topology endpoint returned correct
data, confirmed the Portal rendered a correct diagram — two genuinely
down devices shown red, one genuinely up device shown green, the real
fiber link correctly drawn between the right two nodes, the
unconnected third device correctly shown with no line.

9 backend tests (TopologyTest) + 4 frontend tests, including the two
most load-bearing ones: link normalization and duplicate-rejection
regardless of declared order.

## Legacy system removal (2026-07-23)

This VPS was found to be running a second, unrelated system alongside
Tuwa — discovered while investigating this server's existing WireGuard
setup (wg0, ~25 peers) for a planned zero-touch device provisioning
feature. Confirmed by the account owner to be their own prior
deployment, no longer needed, with full authority to remove.

What it was: a real, previously-live ISP management stack — GenieACS
(TR-069 auto-configuration server, npm package), MongoDB (GenieACS's
datastore), FreeRADIUS (PPPoE/hotspot authentication), a WireGuard
tunnel (wg0) with ~25 real peers, and a Laravel application serving
tuwalink.com, www.tuwalink.com, hotspot.tuwalink.com, and (via a
wildcard rule) any subdomain of tuwalink.com not otherwise configured.

Audited before touching anything: the Laravel app's actual code was
already gone (only an empty storage/logs directory remained), its
scheduler had been failing every single run ("Could not open input
file: artisan"), tuwalink.com and hotspot.tuwalink.com were both
already returning 404, and FreeRADIUS had processed zero
authentications in the prior 24 hours — genuinely dormant
infrastructure, not something actively serving real traffic, though
the WireGuard tunnel itself remained live at the network layer (one
peer had a real handshake within the prior 2 days).

Full backup taken before removal, at /srv/removed-system-backup:
app-code.tar.gz, nginx-tuwalink.conf, wg0.conf.backup, both systemd
unit file backups, tuwalink_db.sql (MySQL dump), and a full mongodump
of GenieACS's database.

Removed, in order, verifying Tuwa's health after each step: stopped
and disabled all services, removed the tuwalink nginx vhost, tore down
the WireGuard interface (a real gotcha here: systemctl stop reported
success but the interface stayed up at the kernel level — wg show
kept showing all peers live; wg-quick down was needed to actually tear
it down, including running the PostDown iptables cleanup rules that a
bare interface deletion would have skipped), removed systemd unit
files, removed the empty app directory, dropped tuwalink_db, uninstalled
the genieacs npm package and the mongodb-org/freeradius apt packages,
and removed the dangling genieacs nginx symlink that remained after
its target application was gone.

Verified clean: no matching enabled services, no matching processes,
nginx sites-enabled shows only Tuwa's seven real vhosts, all 122 NOC
tests still passing, Identity/NOC/Portal all confirmed healthy after
every single step — not just at the end.

## Device provisioning codes (zero-touch onboarding)

Tuwa's own fresh WireGuard interface, built from scratch on 10.20.0.0/24
(port 51821) after removing an unrelated legacy system that was found
sharing this VPS (see "Legacy system removal" above) — no relation to,
or reuse of, anything from that prior setup.

Flow: a Portal user (authenticated) generates a short-lived (15
minute), single-use code via POST /provisioning-codes. A router
redeems it via POST /provisioning-codes/redeem — deliberately outside
identity.auth, since a fresh unconfigured router has no bearer token
yet; the code itself is the credential, which is exactly why it's
short-lived, single-use, and the endpoint is rate-limited
(throttle:10,1). Redemption: allocates the next free 10.20.0.x IP,
adds a real WireGuard peer via WireGuardService (a thin, mockable
wrapper around the one genuinely untestable part — real `wg set` /
`wg-quick save` shell calls, run via a narrowly-scoped passwordless
sudo rule for www-data covering only those exact commands, verified to
actually work as that user before being trusted), creates the Device
record, and returns the server's public key + endpoint so the router
can finish configuring its own side. A failed provisioning attempt (no
IPs left, peer-add failure) never consumes the code — a transient
failure shouldn't permanently burn a code the person would otherwise
need to regenerate from scratch.

Verified live end-to-end, not just via tests: generated a real code,
redeemed it with a genuinely generated WireGuard keypair and no auth
header at all, confirmed a real peer appeared on the live interface
(`wg show`) and a real Device record was created with the correct
assigned IP, confirmed a second redemption attempt with the same code
was correctly rejected, confirmed Tuwa's core services stayed healthy
throughout.

10 new tests (DeviceProvisioningCodeTest): code generation requires
auth, redemption does not, device creation with correct fields, code
marked used on success, duplicate redemption rejected, expired code
rejected, unknown code rejected, and both real failure modes (no IPs
remaining, peer-add failure) correctly leave the code unconsumed.

## Fully automated one-paste provisioning (2026-07-23/24)

The manual two/three-step process above was superseded the same
night by genuine full automation, built and verified against a real
MikroTik hEX S — the first real device connected to Tuwa.

A single RouterOS command, generated per-device in the Portal (with
the real code embedded), does everything: pre-flight check for an
existing wg0 (fails with a readable message rather than a raw
RouterOS error if found), interface creation, calling the redemption
endpoint via /tool fetch, extracting assigned_ip/server_public_key/
server_endpoint from the JSON response via string search (RouterOS
has no built-in JSON parser), splitting the host:port endpoint, and
completing the tunnel — wrapped in :do/on-error so a bad or expired
code fails gracefully instead of crashing.

Backend: JSON_UNESCAPED_SLASHES added to the redeem response, since
WireGuard base64 keys routinely contain '/' and forcing every client
to unescape '\/' is needless complexity. New GET
/provisioning-codes/{code}/status endpoint (public, code-authenticated,
same trust model as redeem) — returns waiting_for_redemption,
waiting_for_connection, or connected, letting the Portal poll every 3s
and show live state without the person needing to do anything after
pasting the command once.

Real RouterOS scripting gotchas found and worked around, only
discoverable by testing against genuine hardware — no amount of
reasoning about RouterOS syntax in the abstract would have caught
these:
- :local variables in the interactive console only live for a single
  command line/submission — typing `:local x "y"` then pressing Enter
  and typing `:put $x` on a new line loses the variable entirely. The
  whole script has to be one chained line (or a saved script file).
- Functions defined with do={...} do NOT inherit outer :local
  variables — they only see their own explicitly-passed parameters.
  An early draft's extractField silently returned nothing because it
  referenced $json from the outer scope instead of a passed-in
  parameter.
- /tool fetch throws (aborting the calling script line) on any
  non-2xx response rather than returning the failure status inside
  its result value — genuinely necessitates :do/on-error wrapping for
  any real error handling.

Verified live, repeatedly, against the real hEX S: genuine one-paste
success (interface creation through tunnel completion, zero manual
key copying), a genuine real WireGuard handshake and working ICMP
ping through the tunnel, the Portal's live polling UI correctly
showing waiting-for-code → waiting-for-connection → connected, the
success banner and automatic panel-close/device-list-refresh, and the
new wg0-already-exists pre-flight check correctly triggering and being
correctly caught.

5 new backend tests (status endpoint) + updated/expanded frontend
tests (the one-paste command flow and live polling, including the
connected success state).

## Real incident: accidental deletion of the real hEX S device record

During the same session's testing, an untargeted cleanup command
(`for id in 9 10 11 12; do curl -X DELETE .../devices/$id; done`)
deleted the real, genuinely-connected tuwatest5 device record
alongside disposable earlier test devices, without first checking
which ID was the real one. A real mistake, not a system bug.

Impact was limited to the Device database row — the underlying
WireGuard tunnel and the router's own configuration were completely
unaffected (`wg show` continued showing a live, recently-handshaking
peer throughout). Recovered cleanly: recreated the Device record via
the real API, restored wireguard_ip/wireguard_public_key to match the
still-live peer, ran devices:poll manually and confirmed it
immediately reconnected correctly.

Worth remembering: any future cleanup involving multiple device IDs
should look up and confirm which records are genuinely disposable
before deleting, not delete by a blind ID range.

## SNMP added to zero-touch provisioning (2026-07-23/24)

A real gap identified directly by the person: the original one-paste
provisioning connected a device via WireGuard but never actually
enabled the rich monitoring (bandwidth, interfaces) the rest of the
platform provides — SNMP had to be set up manually afterward, which
defeats the point of "one paste, everything works." Fixed properly by
extending the automated flow, not by patching the one already-connected
device as a one-off.

Backend: redeem() now generates a real, random, per-device SNMP
community (never shared across the fleet — a single leaked community
would otherwise grant read access to every device) and returns it
alongside server_wireguard_ip. The RouterOS script uses both to enable
SNMP and restrict its community's `addresses` to exactly the
management tunnel IP — confirmed this restriction genuinely works by
testing an SNMP query from an unauthorized address and watching it
correctly time out.

Real, non-obvious finding during testing: RouterOS's default `public`
SNMP community cannot be deleted, only renamed and restricted — a
`/snmp/community/remove` on it fails outright. The correct approach,
used in the script, is `/snmp/community/set` to rename it to the real
generated community and apply the address restriction, rather than
trying to add a second community alongside an still-unrestricted
default.

A real debugging detour worth recording honestly: several live test
runs failed with a generic, unhelpful error message. Traced through
careful isolation (checking `/tool fetch` in isolation, checking
`X-RateLimit-Remaining` headers directly, testing the full redeem
request via curl bypassing the router entirely) to real rate-limiting
— our own extensive diagnostic testing had been consuming the
10-per-minute redemption throttle meant for genuine abuse protection,
not a bug in the actual provisioning logic. Fixed the error message to
honestly mention this as a real possible cause rather than leaving
the person to guess. A separate, unrelated, and genuinely transient
`wg-quick save` filesystem write failure was also investigated (real
kernel logs checked directly) and confirmed to be a one-off, not an
ongoing disk/filesystem problem.

Verified live, completely, end to end against the real hEX S with a
single clean paste: WireGuard tunnel established, SNMP enabled and
genuinely restricted to the tunnel IP (confirmed via direct
`/snmp/print` and `/snmp/community/print` on the router), the
scheduled devices:poll cron picked up connectivity on its own with no
manual trigger needed, and devices:poll-snmp correctly discovered
every real interface with accurate physical port states and began
returning genuine bandwidth figures within one poll cycle.

## Visual redesign, phase 1: tokens + Dashboard (2026-07-24)

Real design tokens extracted directly from a design reference's own
CSS custom properties (:root variables) — void-black surface scale,
blue/cyan technical accent, sharp 3-8px radii, JetBrains Mono/Inter
typography (already matched what Tuwa used). Applied to Tailwind
config using the SAME token names the whole Portal already relies on
(ink.950/900/etc, signal, mist, status) — meaning every existing page
picked up the new palette automatically the moment the config changed,
no per-page markup rewrite needed for the base swap.

Dashboard specifically got the fuller treatment beyond the base
palette: a CardHead component adding corner-bracket accents (small
technical framing marks on card headers, adapted from the reference),
outline-badge KPI cards (small bordered accent square + uppercase
micro-label + large mono value, "technical readout" style), updated
chart gradient colors matching the new blue/cyan story. Every real
data binding — health percent, device counts, bandwidth figures, event
feed — kept completely unchanged; this was a visual pass only, no
behavior or data logic touched.

Real bug found live during the visual sweep, not assumed: Topology's
device details panel had value text with no explicit color class,
relying on an implicit default that happened to be legible against the
old lighter navy palette but became invisible against the new,
noticeably darker void-black tokens. Fixed immediately with an
explicit text-mist-50. A full manual click-through of every other page
(Incidents, Devices, Customers, Subnets, Billing, Activity, Settings)
confirmed no other readability regressions.

Explicitly scoped as phase 1 — deliberately NOT a full re-theme of
every page's fuller visual treatment (KPI-card-style patterns,
corner-bracket accents) beyond the base token swap, given how late in
a long session this was picked up. Every other page has the new colors
correctly (via the shared tokens) but not yet the fuller Dashboard-style
treatment. Continuing that across Incidents/Devices/Topology/Customers/
Subnets/Billing/Activity/Settings is real, scoped, well-understood
follow-up work for next time — the hard design decisions (the actual
token values, the card-head pattern, the KPI card pattern) are already
made and proven, so extending them should be materially faster than
tonight's from-scratch design work.

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

## Git & CI

Each service is its own repo, main branch, pushed to a private GitHub repo
(tuwa-identity, tuwa-noc, tuwa-portal, tuwa-platform-docs, all under the
tutwalinkke account). Commit messages describe what changed and why; several
commits document real bugs found and fixed (SNMP OID key formatting, Carbon
diff sign, Laravel test-guard caching, Http::fake() mocking by URL not by
request header) — worth reading if similar issues recur.

All three app repos have GitHub Actions (.github/workflows/tests.yml) running
the full test suite on every push to main. NOC's workflow additionally
installs and configures a local snmpd agent to match the local test
environment exactly, since PollDeviceInterfacesTest exercises real SNMP
protocol behavior, not mocks. Portal's workflow also runs a production build
as a second signal beyond just tests passing. All three were verified with a
real failing-then-passing run, not just a single green checkmark — NOC's
first CI run caught a genuine test-design bug (Http::fake() mocks by URL
pattern, not by which token was actually sent, so a static fake response
returned the same identity regardless of actor — silently masked in local
runs, caught immediately by CI) which was fixed and reverified.
