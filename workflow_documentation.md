# 1Kosmos Digital Banking End-to-End Workflow

**File:** `1Kosmos_Digital_Banking_EndToEnd.json`  
**Workflow Name:** `1Kosmos_Digital_Banking_EndToEnd`  
**Type:** `idv-ial2`  
**Version:** `v1`

---

## Overview

A single, self-contained 1Kosmos Workflow Builder JSON that guides a user through:

1. Identity enrollment (welcome → consent → camera check → selfie → ID capture → OCR review → face match → account creation)
2. A persistent banking dashboard with three selectable services
3. A money transfer flow with automatic risk-gating (≤ $50,000 direct, > $50,000 triggers LiveID re-auth)
4. A device recovery flow with full re-verification (QR → LiveID → Government ID → Face Match)
5. Secure logout

All sensitive operations reuse `liveid_capture` and `js` nodes following patterns from the reference workflows. No external CSS libraries (Bootstrap, Tailwind) are used. Brand colors and the logo `src` attribute are the only changes needed to reuse for another bank.

---

## Branding Customisation

| Element | Where to Change |
|---|---|
| Logo URL | All `<img class='header-logo' src='...'>` attributes in every node's HTML |
| Bank name | All `<span class='bank-name'>1Kosmos Digital Bank</span>` text nodes |
| Primary color (dark blue `#1a3c6e`) | `styleSheet` — search/replace `#1a3c6e` |
| Button color (inherits primary) | `styleSheet` `.btn-continue` and `.bank-header` background |
| Support email | `recovery_face_match_failed` and `generic_error` HTML bodies |

---

## Supported Node Types Used

| Node Type | Count | Purpose |
|---|---|---|
| `form` | 22 | UI screens, retry screens, confirmation screens, error screens |
| `js` | 8 | Business logic, routing, mock API calls |
| `liveid_capture` | 3 | Selfie + liveness capture (enrollment, transfer auth, recovery auth) |
| `photo_id_capture` | 2 | Government document scan (enrollment, recovery) |
| `qr_for_device_handoff` | 2 | Device resolution handoff + recovery QR authentication |

---

## Complete Node Reference

### Enrollment & Verification Flow

#### `welcome_form` — form
**Purpose:** Landing screen describing the 3-step journey (Selfie, ID, Account Setup).  
**Routing:** `on_next` → `consent_form` | `on_error` → `generic_error`

---

#### `consent_form` — form
**Purpose:** Displays four privacy commitments (GDPR, CCPA, GLBA compliance; encryption; data deletion rights). User must click "I Agree & Continue."  
**Routing:** `on_next` → `camera_check` | `on_error` → `generic_error`

---

#### `camera_check` — form
**Properties:** `check_resolution: true`, `min_resolution_height: 1080`, `min_resolution_width: 1920`  
**Purpose:** Requests camera permission and validates that the device meets minimum resolution for biometric capture.  
**Routing:**  
- Pass → `on_next` → `liveid_capture`  
- Fail resolution → `on_check_resolution_fail` → `qr_device_handoff`  
- Error → `on_error` → `generic_error`

---

#### `selfie_try_again` — form
**Purpose:** Retry screen displayed when `liveid_capture` reports `on_fail`. Instructs user to improve lighting, remove glasses, and face the camera.  
**Routing:** `on_next` → `liveid_capture`

---

#### `liveid_capture` — liveid_capture
**Properties:** `render_style: iframe`, `authtype: none`, `purpose: enroll`, `scopes: wallet,selfie,liveness_score,face_compare_score`, resolution gate enabled  
**Purpose:** Primary biometric selfie + liveness capture for account enrollment.  
**Routing:**  
- Success → `doc_intro_form`  
- Fail → `selfie_try_again`  
- Low resolution → `qr_device_handoff`  
- Error → `generic_error`

---

#### `doc_intro_form` — form
**Purpose:** Informs the user of accepted government documents (Passport, Driver's License, Military ID) before scanning.  
**Routing:** `on_next` → `doc_capture`

---

#### `doc_scan_retry` — form
**Purpose:** Retry screen for `photo_id_capture` failures. Gives tips for a clean scan.  
**Routing:** `on_next` → `doc_capture`

---

#### `doc_capture` — photo_id_capture
**Properties:** `render_style: full_frame`, `check_resolution: true`, resolution gate enabled  
**Purpose:** Captures front (and back if applicable) of the government ID via the 1Kosmos scanning engine.  
**Routing:**  
- Success → `ocr_review`  
- Fail → `doc_scan_retry`  
- Low resolution → `qr_device_handoff`  
- Error → `generic_error`

---

#### `ocr_review` — form
**Purpose:** Confirmation screen displayed after successful document scan. Informs user that OCR extraction is complete and a face match is pending. In production, dynamic data from the `photo_id_capture` node can be injected here using workflow `{placeholder_*}` tokens once the engine exposes an OCR summary placeholder.  
**Routing:** `on_next` → `js_face_match`

---

#### `js_face_match` — js
**Function Name:** Biometric Face Match  
**JS Variables Set:**

| Variable | Type | Description |
|---|---|---|
| `faceMatchScore` | number | Simulated match confidence (0–100). Production: replace with real API call result. |
| `livenessScore` | number | Simulated liveness confidence. |
| `faceMatchPassed` | boolean | `true` if both scores meet thresholds (face ≥ 85, liveness ≥ 80). |
| `matchedAt` | number | Unix timestamp in ms. |
| `next` | string | Dynamic routing target. |

**Dynamic Routing via `next`:**  
- `faceMatchPassed === true` → `next = 'js_create_customer'`  
- `faceMatchPassed === false` → `next = 'face_match_failed'`

`on_success` is intentionally `""` because routing is done via the `next` variable (same pattern as `route_user_based_on_credentials` in the Enrollment reference).

---

#### `face_match_failed` — form
**Purpose:** Terminal error for enrollment if biometrics cannot be matched. Instructs user to restart.  
**Routing:** `on_next` → `welcome_form` (full restart)

---

#### `js_create_customer` — js
**Function Name:** Create Banking Customer Account  
**JS Variables Set:**

| Variable | Type | Description |
|---|---|---|
| `customerId` | string | `CUST-XXXXXXXXX` — simulated unique customer ID |
| `accountNumber` | string | `1K` + 10-digit number — simulated account number |
| `routingNumber` | string | `021000021` — hardcoded mock routing number |
| `accountBalance` | number | `0.00` — new account starts at zero |
| `customerStatus` | string | `ACTIVE` |
| `verificationLevel` | string | `IAL2` — reflects IAL2 identity assurance |
| `createdAt` | number | Unix timestamp in ms |

**Routing:** `on_success` → `banking_dashboard`  

Production replacement: replace the mock logic with a `fetch()` POST to your customer management API. Follow the pattern in `check_user_credentials` from `Enrollment.json` for authenticated API calls with license/keySecret headers.

---

### Dashboard

#### `banking_dashboard` — form
**Purpose:** Main hub after enrollment. Displays a numbered menu of services. Uses a `text` field named `action` so the user enters `1`, `2`, or `3`.

> **Note on `text` field type:** The reference workflows use `html`, `email`, and `ssn` field types. `text` is a natural extension (standard HTML `<input type="text">`). If the engine does not support `text`, replace with a hidden `html` field and a custom button-to-JS approach, or contact the workflow engine team to enable `text` input.

**Fields:**

| Field | Type | Name | Required |
|---|---|---|---|
| HTML menu display | html | — | — |
| Option selector | text | `action` | true |

**Routing:** `on_next` → `dashboard_router`

---

#### `dashboard_router` — js
**Function Name:** Route Dashboard Selection  
**Reads:** `params.nodes.banking_dashboard.action`  
**Dynamic Routing via `next`:**

| Action value | Destination |
|---|---|
| `"1"` | `transfer_recipient_form` |
| `"2"` | `recovery_email_form` |
| `"3"` | `logout_confirm` |
| anything else | `banking_dashboard` (loops back) |

`on_success` is `""` because routing is fully dynamic.

---

### Transfer Money Flow

```
transfer_recipient_form
  → transfer_amount_form
    → js_transfer_risk_route
        ├── (amount ≤ $50,000) → js_execute_transfer → transaction_success → banking_dashboard
        └── (amount > $50,000) → transfer_high_value_auth → transfer_liveid_approval
                                    ├── (success) → js_execute_transfer → transaction_success → banking_dashboard
                                    └── (fail)    → transfer_liveid_retry → transfer_liveid_approval
```

---

#### `transfer_recipient_form` — form
**Purpose:** Collects recipient account number or email.  
**Fields:** `text` input named `recipient`  
**Routing:** `on_next` → `transfer_amount_form`

---

#### `transfer_amount_form` — form
**Purpose:** Collects USD transfer amount. Warns user that amounts above $50,000 require biometric authorization.  
**Fields:** `text` input named `amount`  
**Routing:** `on_next` → `js_transfer_risk_route`

---

#### `js_transfer_risk_route` — js
**Function Name:** Transfer Risk Assessment and Routing  
**Reads:**
- `params.nodes.transfer_amount_form.amount`
- `params.nodes.transfer_recipient_form.recipient`

**JS Variables Set:**

| Variable | Type | Description |
|---|---|---|
| `amount` | number | Parsed float from amount field (strips non-numeric chars) |
| `recipient` | string | Recipient identifier |
| `riskLevel` | string | `LOW` / `MEDIUM` / `HIGH` / `CRITICAL` based on amount tiers |
| `requiresQR` | boolean | `true` if amount > 50000 |
| `transactionId` | string | `TXN-XXXXXXXXX` — unique transaction reference |
| `initiatedAt` | number | Unix timestamp in ms |
| `next` | string | `transfer_high_value_auth` or `js_execute_transfer` |

Risk tiers:

| Amount | Risk Level | QR Required |
|---|---|---|
| ≤ $10,000 | LOW | No |
| $10,001 – $50,000 | MEDIUM | No |
| $50,001 – $100,000 | HIGH | **Yes** |
| > $100,000 | CRITICAL | **Yes** |

---

#### `transfer_high_value_auth` — form
**Purpose:** Warning/briefing screen before the LiveID liveness check for high-value transfers. Explains why additional verification is required.  
**Routing:** `on_next` → `transfer_liveid_approval`

---

#### `transfer_liveid_retry` — form
**Purpose:** Retry screen for LiveID failure during transfer authorization.  
**Routing:** `on_next` → `transfer_liveid_approval`

---

#### `transfer_liveid_approval` — liveid_capture
**Properties:** `purpose: auth` (authentication, not enrollment), same resolution requirements, `scopes: wallet,selfie,liveness_score,face_compare_score`  
**Purpose:** Real-time liveness check to authorize the high-value transfer.  
**Routing:**  
- Success → `js_execute_transfer`  
- Fail → `transfer_liveid_retry`  
- Low resolution → `qr_device_handoff`

---

#### `js_execute_transfer` — js
**Function Name:** Execute Bank Transfer  
**Reads:**
- `params.nodes.js_transfer_risk_route.amount`
- `params.nodes.js_transfer_risk_route.recipient`
- `params.nodes.js_transfer_risk_route.transactionId`

**JS Variables Set:**

| Variable | Type | Description |
|---|---|---|
| `transactionId` | string | Reused from risk assessment node |
| `amount` | number | Transfer amount |
| `recipient` | string | Recipient identifier |
| `transferStatus` | string | `SUCCESS` (mock) |
| `confirmationCode` | string | `CNF-XXXXXX` — receipt reference |
| `estimatedArrival` | string | `1-2 Business Days` |
| `executedAt` | number | Unix timestamp in ms |

Production replacement: replace with `fetch()` to your core banking API transfer endpoint.  
**Routing:** `on_success` → `transaction_success`

---

#### `transaction_success` — form
**Properties:** `return_summary: true`  
**Purpose:** Displays transfer confirmation with a summary checklist. "Return to Dashboard" loops back to `banking_dashboard`.

---

#### `transfer_error_form` — form
**Purpose:** Transfer failure screen with instructions to check details.  
**Routing:** `on_next` → `transfer_recipient_form` (restart transfer flow)

---

### Device Recovery Flow

```
recovery_email_form
  → js_verify_recovery_email
      ├── (invalid email) → recovery_email_invalid → recovery_email_form
      └── (valid)         → recovery_qr_auth (QR display)
                               → recovery_liveid_capture
                                   ├── (fail) → recovery_selfie_retry → recovery_liveid_capture
                                   └── (success) → recovery_doc_intro
                                                      → recovery_doc_capture
                                                          ├── (fail) → recovery_doc_retry → recovery_doc_capture
                                                          └── (success) → js_face_match_recovery
                                                                            ├── (fail) → recovery_face_match_failed → banking_dashboard
                                                                            └── (success) → js_register_device → device_registered_success → banking_dashboard
```

---

#### `recovery_email_form` — form
**Purpose:** Entry point for recovery. Collects the user's registered email.  
**Fields:** `email` field named `recovery_email`  
**Routing:** `on_next` → `js_verify_recovery_email`

---

#### `js_verify_recovery_email` — js
**Function Name:** Verify Recovery Email Against Account  
**Reads:** `params.nodes.recovery_email_form.recovery_email`  
**JS Variables Set:**

| Variable | Type | Description |
|---|---|---|
| `email` | string | Trimmed email input |
| `emailValid` | boolean | Basic format check (contains `@` and `.`) |
| `recoveryToken` | string\|null | `REC-XXXXXXXX` if valid, null if not |
| `initiatedAt` | number | Unix timestamp |
| `next` | string | Dynamic route |

Production replacement: call your user management API (see `check_user_credentials` in `Enrollment.json`) to confirm the email is registered before generating a recovery token.

---

#### `recovery_email_invalid` — form
**Purpose:** Error screen for unrecognized email. Directs user back to try again.  
**Routing:** `on_next` → `recovery_email_form`

---

#### `recovery_qr_auth` — qr_for_device_handoff
**Purpose:** Displays a QR code the user scans with their existing trusted device to initiate recovery authentication. Uses the `{placeholder_for_qrcode}` runtime token.  
**Note:** This node intentionally adds `on_next` and `on_next_button_label` to provide a manual "Continue" button after the user completes mobile QR scanning. The standard `qr_for_device_handoff` pattern only has `on_error`, but adding `on_next` here extends it for the banking authentication use case.  
**Routing:** `on_next` → `recovery_liveid_capture`

---

#### `recovery_selfie_retry` — form
**Purpose:** Retry for LiveID failure during recovery.  
**Routing:** `on_next` → `recovery_liveid_capture`

---

#### `recovery_liveid_capture` — liveid_capture
**Properties:** `purpose: auth` (authentication mode)  
**Purpose:** Liveness check confirming the person initiating recovery matches the enrolled identity.  
**Routing:**  
- Success → `recovery_doc_intro`  
- Fail → `recovery_selfie_retry`

---

#### `recovery_doc_intro` — form
**Purpose:** Explains that a government ID re-scan is required for device recovery (accepts Passport or Driver's License only; Military ID intentionally excluded to reduce attack surface for recovery).  
**Routing:** `on_next` → `recovery_doc_capture`

---

#### `recovery_doc_retry` — form
**Purpose:** Retry for document scan failure during recovery.  
**Routing:** `on_next` → `recovery_doc_capture`

---

#### `recovery_doc_capture` — photo_id_capture
**Properties:** Same resolution requirements as enrollment scan  
**Routing:**  
- Success → `js_face_match_recovery`  
- Fail → `recovery_doc_retry`

---

#### `js_face_match_recovery` — js
**Function Name:** Recovery Biometric Face Match  
**Logic:** Identical thresholds as `js_face_match` (face ≥ 85, liveness ≥ 80). Separate node to keep recovery data isolated from enrollment data in `params.nodes`.  
**Dynamic Routing:**  
- Pass → `js_register_device`  
- Fail → `recovery_face_match_failed`

---

#### `recovery_face_match_failed` — form
**Purpose:** Hard stop — instructs user to contact support. Biometric failure during recovery is a high-security event and should not loop.  
**Routing:** `on_next` → `banking_dashboard`

---

#### `js_register_device` — js
**Function Name:** Register New Trusted Device  
**Reads:**
- `params.nodes.recovery_email_form.recovery_email`
- `params.nodes.js_verify_recovery_email.recoveryToken`

**JS Variables Set:**

| Variable | Type | Description |
|---|---|---|
| `email` | string | Account email |
| `deviceId` | string | `DEV-XXXXXXXX` — new device identifier |
| `registrationCode` | string | `REG-XXXXXX` — one-time registration code |
| `recoveryToken` | string | Token from email verification step |
| `deviceRegistered` | boolean | `true` (mock) |
| `registeredAt` | number | Unix timestamp |

Production replacement: call your device registration API with the `recoveryToken` and the new device's push token/public key.

---

#### `device_registered_success` — form
**Properties:** `return_summary: true`  
**Purpose:** Confirms device registration with a checklist of all recovery steps completed.  
**Routing:** `on_next` → `banking_dashboard`

---

### Logout

#### `logout_confirm` — form
**Properties:** `is_terminal_node: true`  
**Purpose:** Confirms secure session termination. "Start New Session" sends the user back to `welcome_form`, effectively restarting the workflow.  
**Routing:** `on_next` → `welcome_form`

---

### Shared Utility Nodes

#### `qr_device_handoff` — qr_for_device_handoff
**Purpose:** Triggered by any `on_check_resolution_fail` event. Displays a QR code for the user to switch to a capable mobile device. Uses `{placeholder_for_qrcode}` runtime token.  
**Used by:** `camera_check`, `liveid_capture`, `doc_capture`, `transfer_liveid_approval`, `recovery_liveid_capture`, `recovery_doc_capture`

---

#### `generic_error` — form
**Purpose:** Catch-all error screen for unexpected system errors. Instructs the user to return to their original browser tab and request a new session. All nodes set `on_error: generic_error`.  
**Routing:** `on_next` → `welcome_form`

---

## JavaScript Function Patterns

All JS nodes follow the reference pattern established in `Enrollment.json`:

```javascript
// Pattern 1: Static success/fail routing (on_success / on_fail set on node)
var result = { ... };
return result;

// Pattern 2: Dynamic routing via next variable (on_success must be "")
var result = { ... };
if (condition) { next = 'node_a'; } else { next = 'node_b'; }
result.next = next;
return result;
```

Nodes using Pattern 2 (dynamic routing): `js_face_match`, `dashboard_router`, `js_transfer_risk_route`, `js_verify_recovery_email`, `js_face_match_recovery`.

Nodes using Pattern 1 (static routing): `js_create_customer`, `js_execute_transfer`, `js_register_device`.

### Helper Function: `generateId(prefix, len)`
Used in all JS nodes requiring unique IDs. Generates a `PREFIX-XXXXXXX` string using alphanumeric characters. Replace with your platform's UUID or KSUID generation in production.

---

## `params.nodes` Data Flow

The following table shows which downstream JS nodes read data produced upstream:

| Produced by | Field | Consumed by |
|---|---|---|
| `banking_dashboard` | `action` | `dashboard_router` |
| `transfer_recipient_form` | `recipient` | `js_transfer_risk_route`, `js_execute_transfer` |
| `transfer_amount_form` | `amount` | `js_transfer_risk_route` |
| `js_transfer_risk_route` | `amount`, `recipient`, `transactionId`, `riskLevel` | `js_execute_transfer` |
| `recovery_email_form` | `recovery_email` | `js_verify_recovery_email`, `js_register_device` |
| `js_verify_recovery_email` | `recoveryToken` | `js_register_device` |
| `js_face_match` | `faceMatchScore`, `faceMatchPassed` | (informational; not consumed downstream in this workflow) |
| `js_create_customer` | `customerId`, `accountNumber` | (available for dashboard HTML injection in future) |

---

## Routing Map (All Paths)

```
welcome_form
  └─► consent_form
        └─► camera_check ──[low res]──────────────────────────────────────┐
              └─► liveid_capture ──[fail]──► selfie_try_again ─────────────┤
                    │                           └─► liveid_capture          │
                    └─► doc_intro_form                                       │
                          └─► doc_capture ──[fail]──► doc_scan_retry ───────┤
                                │                        └─► doc_capture     │
                                └─► ocr_review                              │
                                      └─► js_face_match                     │
                                            ├─[fail]──► face_match_failed   │
                                            │             └─► welcome_form  │
                                            └─[pass]──► js_create_customer  │
                                                          └─► banking_dashboard◄──────────────────────────────────────────────────────────────┐
                                                                │ action=1                                                                    │
                                                     dashboard_router                                                                        │
                                                          ├─ action=2 ──► recovery_email_form                                               │
                                                          │               └─► js_verify_recovery_email                                      │
                                                          │                     ├─[invalid]──► recovery_email_invalid ──► recovery_email_form│
                                                          │                     └─[valid]───► recovery_qr_auth                              │
                                                          │                                     └─► recovery_liveid_capture                  │
                                                          │                                           ├─[fail]──► recovery_selfie_retry ──►  │
                                                          │                                           └─[pass]──► recovery_doc_intro         │
                                                          │                                                          └─► recovery_doc_capture │
                                                          │                                                                ├─[fail]──► recovery_doc_retry ──►│
                                                          │                                                                └─[pass]──► js_face_match_recovery│
                                                          │                                                                              ├─[fail]──► recovery_face_match_failed ──►│
                                                          │                                                                              └─[pass]──► js_register_device            │
                                                          │                                                                                            └─► device_registered_success─┤
                                                          ├─ action=3 ──► logout_confirm                                                                                            │
                                                          │               └─► welcome_form                                                                                          │
                                                          └─ action=1 ──► transfer_recipient_form                                                                                   │
                                                                           └─► transfer_amount_form                                                                                 │
                                                                                 └─► js_transfer_risk_route                                                                         │
                                                                                       ├─[≤50k]──► js_execute_transfer                                                             │
                                                                                       │             ├─[fail]──► transfer_error_form ──► transfer_recipient_form                   │
                                                                                       │             └─[pass]──► transaction_success ─────────────────────────────────────────────┤
                                                                                       └─[>50k]──► transfer_high_value_auth                                                        │
                                                                                                     └─► transfer_liveid_approval                                                   │
                                                                                                           ├─[fail]──► transfer_liveid_retry ──► transfer_liveid_approval           │
                                                                                                           └─[pass]──► js_execute_transfer ──► transaction_success ─────────────────┘

[low res from any node] ──► qr_device_handoff (terminal for that session leg)
[on_error from any node] ──► generic_error ──► welcome_form
```

---

## Production Hardening Checklist

- [ ] Replace all `generateId()` mock functions with cryptographically secure UUID generation from your backend API
- [ ] Replace `js_face_match` and `js_face_match_recovery` static scores with real calls to your biometric matching API
- [ ] Replace `js_create_customer` with authenticated POST to your customer management service
- [ ] Replace `js_execute_transfer` with authenticated POST to your core banking transfer API
- [ ] Replace `js_verify_recovery_email` email check with your user lookup API (see `check_user_credentials` pattern in `Enrollment.json`)
- [ ] Replace `js_register_device` with your device registration / ACR code API (see `create_acr_code` pattern in `Enrollment.json`)
- [ ] Update `dvcId` from `banking_dvc_sandbox` to your production DVC ID
- [ ] Replace all static logo URLs with your bank's CDN-hosted logo URL
- [ ] Update support email in `recovery_face_match_failed` and `generic_error` nodes
- [ ] Set `isPublished: false` during development; flip to `true` for production release
- [ ] Add rate-limiting and fraud score thresholds to `js_transfer_risk_route`
