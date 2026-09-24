# Gather: the safety work

*Private draft for the owner's review. Each claim cites a file or test in this repository.
Anything not confirmed there is marked (unverified).*

## 1. What Gather is

Gather is a web app for a church district and its churches: kids' check-in, volunteer serving,
and online payments and giving. It holds children's medical notes, the list of who may pick each
child up, and money. Safety therefore means three things: the wrong adult never gets a child or a
child's records, money is handled honestly, and an older, non-technical volunteer can still sign
in.

## 2. Sign-in and sessions

- **Passwords** are hashed with scrypt and checked in constant time. A wrong password and an
  unknown email get the same response and take about the same time. [1]
- **Second factor without an app:** a 6-digit emailed code (10 minutes, 5 tries, one live code)
  plus "remember this device for 90 days." Only the three money and district roles need it. No
  passkeys or SMS; a real older volunteer could not use an authenticator app. [2][3]
- **"I can't get in"** alerts the person's leaders and always gives the same reply, so it reveals
  nothing about which accounts exist. [4]
- **Sessions** are random 256-bit tokens in HttpOnly, Secure cookies. The server stores only a
  hash, so a stolen database file signs no one in. [1][5]
- **Session length follows role.** Parents and volunteers stay signed in up to 90 days; money and
  district roles time out after 12 idle hours. A "shared computer" box ends the session at browser
  close or after 15 idle minutes. [5][6]
- **Re-authentication:** 21 sensitive routes (sign-in resets, exports, money, granting View As)
  re-ask for the password mid-session. A test sends a real request to each. [7][8]
- **Rate limits:** 5 wrong passwords lock that account *from that IP* for 15 minutes, never the
  account alone, so knowing a staff email is not enough to lock the real person out. Every route
  also has a request-volume limit. [9][10]
- **CSRF:** double-submit token. Tests fail if a form route is unprotected or a public route is
  missing from the reasoned allowlist. [11][12]

## 3. Authorization

- There are nine roles and one permission table; anything not granted is denied. [13]
  `authorize()` is called 503 times across 96 files. Code scans fail the build if a data
  function skips the outpost or ministry scope check. [14]
- **Authorization audit** (merged 2026-09-18). It fixed 21 findings plus 8 must-fix items from
  review. It left 51 attack-test files (105 tests) covering cross-church reads, cross-outpost
  forgery, and second-factor bypass. [15][16]
- **View As** is read-only. The server blocks every write, and pages disable their buttons.
  Granting, starting, and exiting View As are all recorded in the audit log. [17][18]
- **District isolation.** Each district has its own database file and encryption key. [19] One
  rule says an empty church shows an empty page and never another church's data. That rule is
  written down but not yet enforced by a test. [20]

## 4. Data safety

- **Audit log:** each entry is chained to the hash of the one before it, so editing or deleting
  any past entry can be detected. [21]
- **Medical records** are encrypted with AES-256-GCM using a separate key per district. That key
  is stored encrypted under a master key. [22]
- **Messages are hidden, never deleted.** Hiding affects only your own inbox; no code updates or
  deletes a message row. [23]
- **Opening an email link changes nothing.** A mail scanner once burned a sign-in link before the
  person clicked it. Every emailed link now shows a landing page; only Continue (a POST) uses the
  token. A test opens each link type twice and fails on any unregistered new type. [24]
- **Schema changes** run as a dry run first, then on one test district. Running them needs a
  one-time approval tied to that exact plan. [25]
- **Backups** are proven by restoring them. A script opens each backup read-only and runs an
  integrity check. [26] The plan calls for a weekly drill (unverified that it runs weekly).
- **Beta-tester logs** record screens, buttons, and field *names*, never typed text. [27]

## 5. Money

Stripe's hosted checkout page takes the card, so card numbers never reach Gather. Stripe
notifications need a valid signature and are refused if they are more than 5 minutes old. [28] If
any Stripe key is missing, card payments are turned off everywhere. [29] Fees appear as two
labeled lines, the Gather fee and card processing, and the church receives the full event price.
[30] A script scans the repository for anything that looks like a secret key. [31]

## 6. Child safety at check-out

Pickup works like a coat check: one code per family per event, compared in constant time, never
logged or shown to staff. [32] No code? Staff resend it; failing that, a photo ID must match an
adult on the child's file and a second leader must approve, logged and emailed to the parent.
Anyone not on file is refused. [33] Families can pre-authorize other pickup adults. [34] A QR-code
checkout has been designed but not approved or built.

## 7. Operations

- One live server; all building happens in separate copies, checked by a script. [35]
- Restarts kill by process ID and verify the new process; a bare "200 OK" is not proof. [36]
- The hosting runbook disables SSH password and root login and blocks all inbound traffic behind
  an outbound tunnel. [37] Whitelisting the operator before fail2ban, and reading the SSH config
  rather than testing it, are operator practice (unverified in the repository).

## 8. Testing

The last full run passed 4,606 tests, 0 failed. [38] A run passes only with exit code 0 and zero
✖ marks; the output is not TAP, so grepping for TAP lines proves nothing. A rehearsal harness
boots a fresh district, opens one real browser per role, and plays an event end to end; any
button whose effect doesn't match its label becomes a screenshot and a defect file. [39]

## 9. Numbers

| Measure | Count |
|---|---|
| Tests in last full run | 4,606 passed, 0 failed |
| Authorization audit findings fixed + review must-fixes | 21 + 8 |
| Authorization-audit test files / tests | 51 / 105 |
| Routes requiring re-authentication | 21 |
| Registered routes / public (all allowlisted) | 387 / 40 |
| `authorize()` call sites | 503 in 96 files |
| Merges with security-related titles | 83 of 658 |
| Owner rules written / enforced by a test | 46 / 12 |

## 10. How it was built

I direct a studio of AI coding agents. I set the rules (kept in a dated, append-only file), walk
every screen, and approve anything visible before it ships. Agents write the code under those
rules; separate reviewer agents check each batch against my original request, and attack tests
must fail before a fix goes in. 34 of my 46 rules still lack a test; that is my to-do list.

---
[1] docs/auth.md, src/auth/password.ts · [2] src/auth/mfaEmailCode.ts · [3] src/http/routes/authRoutes.ts ·
[4] src/dal/signInHelp.ts · [5] src/http/session.ts · [6] src/config/sessionLifetime.ts,
tests/sessionLifetimeRoleRefresh.test.ts · [7] src/auth/stepUp.ts, src/http/stepUpGuard.ts ·
[8] tests/authzStepUpRouteTable.test.ts · [9] src/auth/rateLimit.ts, tests/rateLimitPairing.test.ts ·
[10] src/http/generalRateLimit.ts · [11] tests/formCsrfRouteCoverage.test.ts ·
[12] tests/publicRouteAllowlist.test.ts · [13] src/policy/rbac.ts, docs/rbac_matrix.md ·
[14] tests/dalOutpostScopeScan.test.ts, tests/dalMinistryScopeScan.test.ts · [15] merge 47a9fc4b ·
[16] tests/authzAudit/ · [17] src/http/previewGuard.ts, tests/viewAsPreviewNoteSweep.test.ts ·
[18] src/dal/viewAs.ts · [19] docs/adr/0001-one-sqlite-file-per-tenant.md, tests/tenantIsolation.test.ts ·
[20] docs/LAWS.md · [21] src/audit/auditLog.ts · [22] src/crypto/envelope.ts ·
[23] src/dal/messages.ts, tests/messages.test.ts · [24] tests/emailLinkLaw.test.ts ·
[25] docs/runbooks/migrate_all.md · [26] scripts/restore-drill.ps1, docs/adr/0006-backups-proven-by-restore.md ·
[27] src/http/activityContext.ts · [28] src/payments/stripeClient.ts · [29] src/config/stripeConfig.ts ·
[30] docs/LAWS.md, branch fee-model (merged) · [31] scripts/check-secrets.mjs · [32] src/dal/pickupCodes.ts ·
[33] src/dal/pickupOverrides.ts · [34] src/dal/authorizedPickupAdults.ts · [35] scripts/check-one-checkout.ps1 ·
[36] scripts/restart-live.mjs, scripts/safe-kill.ps1 · [37] docs/HOSTING_MOVE_PLAN.md ·
[38] docs/suite_reports/stack-2026-09-18f3_report.md · [39] tests/rehearsal/run-local.ts
