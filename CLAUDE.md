# CLAUDE.md

Guidance for Claude Code working in this repo.

## Project type

UiPath Studio Process project, REFramework 25.10 layout, Portable target, VB expression language.
Consumer half of the New-Hire Benefits Posting solution (PDD ACB-BEN-PDD-0007). Drains the
`NewHireBenefitsPosting` Orchestrator queue and posts each new hire's elections to the
BenefitConnect partner API.

The producer half is `../ACB.NewHireOnboarding.Dispatcher` (random demo-data generator,
1+ items per run, asset-driven queue name).

## Design philosophy: API-first, UI Automation as the documented fallback

Every C-step in `Framework/Process.xaml` is implemented as **HTTP calls** against the
BenefitConnect REST surface (`/api/*`) rather than UI-driving the portal. The API is
undocumented (reverse-engineered from the portal's network traffic) and is the same surface the
web app uses — chosen because it's sub-second per call, has no selectors to maintain, no
rendering races, no headed browser.

Because the API is undocumented, treat every response field as optional and guard every JObject
dereference (see "JObject access pattern" below).

### UI Automation fallback

**The fallback is not theoretical — it's the documented contingency in PDD §3.2 / Appendix C
and the only path for the two endpoints that are UI-session-only by partner policy (see "APIs
we don't call" below).** Every step the consumer performs has a corresponding portal flow at
`https://accrual-bp-hri.azurewebsites.net`. `UiPath.UIAutomation.Activities` is already in the
dependency list — the fallback is wired-up-ready, just not implemented in v1.

When to switch a C-step from API → UI Automation:

- the `/api/*` route returns `403 API_NOT_PUBLIC` (tenant policy revoke),
- the response schema drifts and field-by-field guarding gets out of hand,
- you need to perform an operation that has no API verb (dependents / beneficiaries — UI-only
  by policy).

**Whole-step swap, not per-call fallback.** Replace the HTTP call inside a Process step with a
UI Automation sequence; keep the queue contract, business rules, and exception codes
unchanged. Do **not** wrap an HTTP call in TryCatch with a UI fallback inside the same step —
each step is one path or the other.

C-step → portal screen mapping (PDD Appendix C):

| C-step | API today | UI equivalent |
|---|---|---|
| C-03 IdentityAssertion | `GET /api/employees/me` | `/admin/onboarding/{employeeId}` — Candidate panel (verify Preferred Name, Email, DOB, Employee Number) |
| C-04 DependentPreCheck | `GET /api/dependents` | Enrollment wizard step 2 — Dependents |
| C-05 BeneficiaryPreCheck | `GET /api/beneficiaries` | Beneficiary management screen (under Employee profile) |
| C-06 DuplicateActiveCheck | `GET /api/enrollments/current` | `/admin/onboarding/{employeeId}` — Status badge (`Active` → SUCCESS_NO_OP) |
| C-07 PostElectionLines | `POST /api/plans/select` × N | Enrollment wizard steps 3 → 5 (Medical / Dental-Vision-Life / Savings-401k); one drop-down per line, in BR-05 order |
| C-08 ReconcileDraft | `GET /api/enroll/draft` | Wizard step 6 — Review (visual reconcile vs intake form) |
| C-09 CreatePendingEnrollment | `POST /api/enroll` | Wizard step 6 — **Submit** button (Draft Saved → Pending Confirmation) |
| C-10 ConfirmToActive | `POST /api/onboarding/pending/{id}/approve` | `/admin/onboarding/{employeeId}` — green **Approve** button (Pending Confirmation → Active, queue item resolved) |

Notes for whoever wires a UI fallback:

- Use `NApplicationCard` + `uia-configure-target` for selector capture; never hand-write
  selectors. See the `uipath-rpa` skill's UI automation guide.
- Sign-in at `/login` happens once per session; cache the session via cookies in the same
  `NApplicationCard` scope, don't sign in per transaction.
- The portal sorts the Onboarding Queue most-urgent-first by `Days Since Hire`; for a single
  candidate, navigate directly to `/admin/onboarding/{employeeId}` rather than scraping the list.
- The Dependents and Beneficiaries screens are the **only place** to create/edit those records —
  the UI is not just a fallback there, it's the canonical path (PDD OI-02, OI-03).

## APIs we call (the meat)

All calls carry header `X-API-Key: <ACB.BenefitConnect.ApiKey asset>`. Per-employee calls also
carry `X-Employee-Id: <candidate.id>` (or `?employeeId=` query param equivalent). Base URL is
the `ACB.BenefitConnect.BaseUrl` asset. The shared HTTP wrappers live in
`Framework/HttpGet.xaml` / `Framework/HttpPost.xaml` and inject those headers
automatically — never hand-add them in a Process step.

### C-03 · `GET /api/employees/me` — `Process/IdentityAssertion.xaml`

Returns the employee record for `X-Employee-Id`. **Auto-vivify**: any `E<digits>` id creates a
placeholder employee on first reference (no HR pre-creation required). Used purely to assert
that the queue item's `EmployeeId` matches what the partner sees; mismatch → `IDENTITY_MISMATCH`.

### C-04 · `GET /api/dependents` — `Process/DependentPreCheck.xaml`

Returns JArray. Combined with the medical tier from the queue item, enforces **BR-03**
(`EE+Spouse`/`Family` requires a spouse on file; `EE+Child`/`Family` requires a child).
Failure → `BR03_TIER_INCONSISTENT`.

### C-05 · `GET /api/beneficiaries` — `Process/BeneficiaryPreCheck.xaml`

Returns a **JArray of beneficiary records** (not the `{primary:[…], contingent:[…]}` JObject
the PDD sketch suggests — parse with `DeserializeJsonArray`). If a life line is elected and the
array is empty → `BENEFICIARIES_MISSING` (per BR-13).

### C-06 · `GET /api/enrollments/current` — `Process/DuplicateActiveCheck.xaml`

Returns the current `Active` enrollment for the plan year, or `null`. If non-null **and**
status=`Active` **and** matching `planYear` → idempotency hit (BR-11): mark
`SUCCESS_NO_OP` and short-circuit the rest of the pipeline. No further mutating calls.

### C-07 · `POST /api/plans/select` × N — `Process/PostElectionLines.xaml`

Posts each elected line in **BR-05 order** (medical → dental → vision → basicLife →
voluntaryLife → ad_and_d → std → ltd → fsaHealth → fsaDependent → hsa → commuter →
retirement). Payload shape varies per plan type — see PDD §8.3. BRs enforced before each post:
BR-04 (vol-life cap), BR-06 (IRS limits), BR-07 (HSA-vs-HDHP), BR-08 (DCFSA eligible dep). Any
4xx aborts the whole transaction → `PLAN_SELECTION_REJECTED`.

**Plan IDs in the live catalog are bare strings** (not `PLN-*` like the PDD examples).
Discoverable via `GET /api/public/plans`. Current set:
`MED-HMO`, `MED-PPO`, `MED-HDHP`, `DEN-BASIC`, `DEN-PLUS`, `VIS-BASIC`, `VIS-PREMIUM`,
`LIFE-VOL`, `LIFE-SPOUSE`, `LIFE-CHILD`, `STD`, `LTD`, `FSA-HC`, `FSA-DC`, `HSA`, `COMMUTER`,
`401K`.

Tier codes in the catalog are `EE` / `ES` / `EC` / `EF` (employee / spouse / child / family).

### C-08 · `GET /api/enroll/draft` — `Process/ReconcileDraft.xaml`

Returns `{employeeId, tier, elections: { medical: {…}, dental: {…}, … }, updatedAt}`.
**`elections` is a JObject** keyed by plan-type name — count its **properties**, not array
items. Mismatch between draft property count and `linesPosted` from C-07 →
`DRAFT_RECONCILE_MISMATCH`.

### C-09 · `POST /api/enroll` — `Process/CreatePendingEnrollment.xaml`

Promotes the draft to `Pending Confirmation`. Capture the returned `enrollmentId` for the
transaction log. 4xx → `ENROLLMENT_REJECTED`.

### C-10 · `POST /api/onboarding/pending/{employeeId}/approve` — `Process/ConfirmToActive.xaml`

The **preferred** C-10 path: confirms the pending enrollment **and** resolves the HR onboarding
queue item in one call. Capture `confirmationNumber`. 4xx → `CONFIRM_REJECTED`.

Alternative (not used here): `POST /api/confirm` confirms enrollment only and leaves the queue
item open — would require a separate `POST /api/onboarding/pending/:id/approve` call.

## APIs we don't call (lightly)

These exist on the surface but the consumer deliberately skips them:

- **`POST/PATCH/DELETE /api/dependents`, `PUT /api/beneficiaries`** — documented as
  UI-session-only (PDD OI-02, OI-03). The orientation coach captures dependents and beneficiaries
  through the BenefitConnect UI before the candidate ever appears in the queue. The consumer
  reads these endpoints (C-04, C-05) but never writes to them.
- **`GET /api/admin/onboarding-queue`, `POST /api/admin/onboarding-queue/:id/approve`** — UI
  shell views; equivalent to the API surface above (`GET /api/onboarding/pending`,
  `POST .../pending/:id/approve`) which we do use.
- **`GET /api/documents`** — public partner endpoint for confirmation statements / SBCs. Not
  fetched in v1 (PDD §12.3 says the BenefitConnect platform audit log plus the posting log are
  sufficient evidence; OI-06 tracks this).
- **`GET /api/public/status`, `GET /api/public/version`, `GET /api/public/plan-types`,
  `GET /api/public/carriers`** — liveness / metadata endpoints. The original PDD §5.2 dispatcher
  used `/api/public/health` + `/api/public/status` as a pre-check. The current Dispatcher is a
  random demo-data generator with no upstream poll, so these aren't called from this solution.
- **`GET /api/public/plans`** — plan catalogue. Used at design time to lock in plan IDs for the
  generator and the BR-06 limits, but not called at runtime.

Out-of-scope process flows (won't be added here): QLE processing (SCN-11), Open Enrollment
batch (SCN-08), payroll deduction feed (PI-2024-02), carrier eligibility files. These are
separate scenarios per PDD §1.2 / §13.3.

## Architecture (relative paths)

Stock REFramework state machine in `Main.xaml` (Init → GetTransactionData → Process → End). The
business pipeline lives outside the stock files:

| Path | Role |
|---|---|
| `Main.xaml` | REF state machine entry. Reads `Data/Config.xlsx` `OrchestratorQueueName = NewHireBenefitsPosting`. **Do not modify**, with one Portable-target exception already applied: the "Log Message screen resolution" activity was removed because `Screen.PrimaryScreen` (`System.Windows.Forms`) does not resolve off Windows and broke `uip rpa build` on Linux CI runners. |
| `Framework/InitAllSettings.xaml` | Stock REF — reads Config.xlsx into `in_Config`. |
| `Framework/InitAllApplications.xaml` | Stock REF — empty in v1 (asset load lives in `InitWorkflow.xaml`). |
| `Framework/GetTransactionData.xaml` | Stock REF — `GetTransactionItem` against the queue. **Do not modify.** |
| `Framework/Process.xaml` | Bespoke — extracts `SpecificContent.*` from the queue item, calls `InitWorkflow`, dispatches to C-03 → C-11. On `outcomeCode = BUSINESS_EXCEPTION` throws `UiPath.Core.BusinessRuleException` so REF routes the item to `BusinessException`. Other exceptions bubble to REF as system exceptions. |
| `Framework/SetTransactionStatus.xaml` | Stock REF, **modified for Portable target**: the "Try taking screenshot" block and its `ScreenshotPath` variable were removed (TakeScreenshot is a Windows-only UIAutomation activity); the queue's `SetTransactionStatus.Details` no longer composes a screenshot path. Retry/exception semantics still belong in Config.xlsx (`MaxRetryNumber`, `MaxConsecutiveSystemExceptions`). |
| `Framework/RetryCurrentTransaction.xaml`, `KillAllProcesses.xaml`, `CloseAllApplications.xaml` | Stock REF — leave alone. (`TakeScreenshot.xaml` was deleted during the Portable conversion.) |
| `Framework/InitWorkflow.xaml` | Bespoke — loads the 4 Orchestrator assets (`ACB.BenefitConnect.BaseUrl`, `ACB.BenefitConnect.ApiKey`, `ACB.NewHireOnboarding.PostingOrder`, `ACB.NewHireOnboarding.IRSLimits.2026`). |
| `Framework/HttpGet.xaml`, `HttpPost.xaml`, `HttpInvoke.xaml` | Bespoke — wrap legacy `HttpClient`, inject `X-API-Key` + `X-Employee-Id`, classify 401/403/5xx into `out_ErrorCode`. **All API calls go through these — don't drop a raw `HttpClient` into a Process step.** |
| `Framework/HandleBusinessException.xaml`, `HandleApplicationException.xaml`, `End.xaml` | Bespoke — exception logging + escalation hooks. |
| `Process/IdentityAssertion.xaml` … `LogOutcome.xaml` (9 files) | One per C-step (C-03 … C-11). |
| `Rules/BR01_HireDateWindow.xaml` … `BR11_Idempotency.xaml` (9 files) | One per business rule. Each takes typed in-args, emits `(out_IsValid, out_ExceptionCode, out_Detail)`. |
| `Tests/*` | REFramework stock test scaffolds. Not populated with real cases in v1. |
| `Data/Config.xlsx` | `OrchestratorQueueName=NewHireBenefitsPosting`. Add new robot-level settings here, not in XAML. |

### Data flow per transaction (1 queue item)

```
NewHireBenefitsPosting queue
  └── GetTransactionData → in_TransactionItem (QueueItem)
        └── Framework/Process.xaml
              ├── extract SpecificContent.{TransactionId, EmployeeId, PlanYear, …}
              ├── Framework/InitWorkflow.xaml (4 assets)
              ├── C-03 IdentityAssertion        → asserts EmployeeId
              ├── C-04 DependentPreCheck        → BR-03 tier check
              ├── C-05 BeneficiaryPreCheck      → BR-13 if life line elected
              ├── C-06 DuplicateActiveCheck     → BR-11 SUCCESS_NO_OP short-circuit
              ├── C-07 PostElectionLines × N    → BR-04..BR-08, BR-05 order
              ├── C-08 ReconcileDraft           → DRAFT_RECONCILE_MISMATCH check
              ├── C-09 CreatePendingEnrollment  → captures enrollmentId
              ├── C-10 ConfirmToActive          → captures confirmationNumber
              └── C-11 LogOutcome + throw BusinessRuleException on business failure
```

## Commands (relative paths from the project root)

```bash
# Per-file validation (run after every XAML edit)
uip rpa get-errors --file-path "Framework/Process.xaml" --project-dir "."

# Project-level build (catches member/enum errors get-errors misses)
uip rpa build "."

# Pack + deploy
uip rpa pack "." "../nupkg"
uip or packages upload "../nupkg/ACB.NewHireOnboarding.Performer.<version>.nupkg"
uip or processes update-version <process-key> --package-version "<version>"

# Start a job in DevCon26
uip or jobs start --folder-path "DevCon26" <process-key>
```

`get-errors` is the single source of truth for "did my edit break the workflow" — always run it
after touching any `.xaml`. Combine with `build` because `get-errors` doesn't catch unknown
member names or invalid enum values.

## JObject access pattern

Responses are deserialized into `Newtonsoft.Json.Linq.JObject` (variables `meObj`, `enrollObj`,
`confirmObj`, `draft`, etc.). The indexer `meObj("key")` returns `Nothing` for missing keys —
every dereference must be guarded:

```vb
If(meObj("id") IsNot Nothing, meObj("id").ToString, "(default)")
```

Treat any new `meObj(...)` / `enrollObj(...)` / `confirmObj(...)` access as defective until
guarded.

## Activity packages (locked versions)

| Package | Version |
|---|---|
| `UiPath.System.Activities` | `25.10.0` |
| `UiPath.UIAutomation.Activities` | `26.4.1-preview` |
| `UiPath.WebAPI.Activities` | `2.4.0` |
| `UiPath.Excel.Activities` | `3.2.1` |
| `UiPath.Mail.Activities` | `2.8.10` |
| `UiPath.Testing.Activities` | `25.10.0` |
