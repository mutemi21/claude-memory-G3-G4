# PAYGO / Token / Credit System — Detailed Summary

## Overview
The G4 cooker implements a Pay-As-You-Go (PAYGO) system that controls whether the cooker can be used based on whether the customer has paid. The system uses the **Nexus Keycode** library (from Angaza) to validate 15-digit tokens entered by users. Tokens encode a "due date" — the date until which the cooker is authorized to operate.

## Two Credit Management Modes
The system supports two modes, controlled by `STATUS_FLAG_CRD_MGMT_MODE`:

1. **RTC Mode** (default, flag NOT set): Compares current RTC time against a stored due date. If current time > due date, cooker is locked.
2. **Device On/Off Mode** (flag SET): Server sends `"device_on"` or `"device_off"` via shadow update. No time comparison.

The mode is set by the `"Cred_Mgt"` field in the shadow JSON from the server (`"RTC"` or `"credit"`).

## Architecture & Data Flow

### Token Entry Flow (User enters 15-digit code on keypad)
```
UIPresenter (TOKEN_ENTRY state, keypad input)
  → UIView_ReturnCurrentTokenArray() (collects 15 digits)
  → UIModel_ValidateToken()
    → Token_EnterToken() (converts byte array to string, enqueues)
      → Token_Churn() (main loop dequeues)
        → nx_keycode_handle_complete_keycode() (Nexus library)
          → nxp_keycode_payg_credit_set() callback (SET token)
             OR nxp_keycode_payg_credit_unlock() callback (UNLOCK token)
          → nxp_keycode_feedback_start() callback (result feedback)
```

### Token Processing Result → UI Feedback
```
nxp_keycode_payg_credit_set() / nxp_keycode_payg_credit_unlock()
  → UIPresenter_TknRsltACK()    → UI_PRESENTER_STATE_TOKEN_ACCEPTED
  → UIPresenter_TknRsltNACK()   → UI_PRESENTER_STATE_TOKEN_REJECTED
  → UIPresenter_TknRsltUsedTkn()→ UI_PRESENTER_STATE_TOKEN_ALRDY_USED

nxp_keycode_feedback_start()
  → UIPresenter_TknRsltRateLimit() → UI_PRESENTER_STATE_TOKEN_RATE_LIMITED
  → UIPresenter_TknRsltNACK()      → UI_PRESENTER_STATE_TOKEN_REJECTED
```

### Credit Check Flow (every time user tries to cook)
```
UIPresenter action function (e.g., user presses power)
  → UIModel_IsUserInCredit()
    → CreditManagementIsCreditStatusOkay()
      → If STATUS_FLAG_CRD_MGMT_MODE set: Shadow_IsDeviceStatusDeviceOn()
      → If STATUS_FLAG_DEVICE_ALWAYS_ON set: return true (fully paid off)
      → Otherwise (RTC mode): !IsLoanOverDue() (compare RTC vs deadline)
```

If not in credit, UIPresenter redirects to `UI_PRESENTER_STATE_TOKEN_ENTRY` instead of allowing cooking.

### Server-Side Credit Update Flow (via Bluetooth or cellular)
```
Bluetooth/MQTT receives shadow JSON
  → Shadow_Parse()
    → HandleShadowUpdate() parses "due_date", "status", "Cred_Mgt"
      → CreditManagement_UpdateDeadline() (updates cached deadline)
      → Flash_SaveData() (persists shadow to internal flash)
```

## Key Files

| File | Purpose |
|------|---------|
| `ProductFeatures/Token/Src/Token.c` | Token queue, Nexus library init/churn, callbacks for `nxp_common_*` |
| `ProductFeatures/Token/Src/TokenKeycode.c` | Nexus keycode callbacks: `nxp_keycode_payg_credit_set()`, `nxp_keycode_payg_credit_unlock()`, `nxp_keycode_feedback_start()`, `nxp_keycode_get_secret_key()` |
| `ProductFeatures/Token/Src/TokenNv.c` | Non-volatile storage for Nexus library state (SPI flash, 2 blocks) |
| `ProductFeatures/CreditManagement/Src/CreditManagement.c` | Central credit decision: `CreditManagementIsCreditStatusOkay()`, `IsLoanOverDue()` |
| `ProductFeatures/Shadow/Src/Shadow.c` | Shadow (device twin) — parses server JSON, stores due date, calls `CreditManagement_UpdateDeadline()` |
| `ProductFeatures/SecData/Src/SecDataStore.c` | Stores token secret key and production date in MCU internal flash (page 63) |
| `ProductFeatures/DeviceSettings/Src/DeviceSettings.c` | Stores PAYGO version string and serial number in SPI flash |
| `ProductFeatures/UI/Src/UIPresenter.c` | State machine with TOKEN_ENTRY and result states |
| `ProductFeatures/UI/Src/UIModel.c` | `UIModel_IsUserInCredit()`, `UIModel_ValidateToken()` |
| `ExternalPeripherals/InductionCooker/Src/IndCooker.c` | `CookerShouldBeLocked()` — currently hardcoded to `false` (TODO) |
| `Drivers/Nexus/` | Angaza Nexus Keycode library (third-party, do not modify) |

## How Tokens Work (Nexus SET Credit)

1. Token is a 15-digit numeric code entered on the keypad
2. Nexus library decodes it using the device's 16-byte secret key (`nxp_keycode_get_secret_key()`)
3. Token type is **SET** (not ADD): credit value = absolute seconds since production date
4. `nxp_keycode_payg_credit_set(Credit)` receives credit in **seconds since epoch**
5. New due date = `ProductionDate + Credit` (converted via Unix time)
6. If new due date < current time → reject as expired (PG10 error)
7. If valid → `Shadow_SetDueDate()` → persists to flash → updates CreditManagement deadline

## How Tokens Work (Nexus UNLOCK)

1. Special token that permanently unlocks the device
2. Sets due date to 31/12/2099 00:00:00
3. `CreditManagement_UpdateDeadline()` detects Year=99 → sets `STATUS_FLAG_DEVICE_ALWAYS_ON`

## Security Data (MCU Internal Flash, Page 63, Address 0x0801F800)

- **TokenKey**: 16-byte secret key for Nexus keycode decryption (with modulo-256 checksum)
- **DataEncryptionKey**: 16-byte key for data encryption (separate from token key)
- **ProductionDate**: 6 bytes (Y/M/D/H/M/S) — the epoch for SET token credit calculation
- All validated with mod-256 checksums

## Nexus Library NV Storage (SPI Flash)

- Two blocks (Block 0, Block 1) stored at `SPI_FLASH_TOKEN_DATA_STORAGE_ADDRESS`
- 31 bytes total: 4-byte header sequence (0xAA55AA55) + length + data for each block
- Stores keycode processing state (message IDs, rate limiting counters, etc.)
- Written lazily via `TokenNV_Churn()` in main loop (not in callback context)

## Shadow JSON Format
```json
{"state":{"FW":"","due_date":"25/12/12,12:56:07","hbf":"5","dtm":"0","status":"device_on","Cred_Mgt":"RTC","rtc_tamper_threshold":"10"}}
```
- `due_date`: Format "dd/MM/yy,HH:mm:ss"
- `status`: "device_on" or "device_off"
- `Cred_Mgt`: "RTC" (time-based) or "credit" (on/off mode)

## UI States for Token Flow

| State | View | Timeout |
|-------|------|---------|
| `UI_PRESENTER_STATE_TOKEN_ENTRY` | `UI_VIEW_TOKEN_ENTRY` | 30s |
| `UI_PRESENTER_STATE_AWAITING_TOKEN_VALIDATION_RSLT` | `UI_VIEW_BLANK_SCREEN` | 5s |
| `UI_PRESENTER_STATE_TOKEN_ACCEPTED` | `UI_VIEW_TOKEN_ACCEPTED` | 5s |
| `UI_PRESENTER_STATE_TOKEN_REJECTED` | `UI_VIEW_TOKEN_REJECTED` | 5s |
| `UI_PRESENTER_STATE_TOKEN_ALRDY_USED` | `UI_VIEW_TOKEN_ALRDY_USED` | 5s |
| `UI_PRESENTER_STATE_TOKEN_RATE_LIMITED` | `UI_VIEW_TOKEN_RATE_LIMITED` | 5s |

## Error Codes (DeviceErrors)

| Code | ID | Meaning |
|------|----|---------|
| PG8 | 19 | Invalid Token |
| PG9 | 20 | Duplicate Token (valid but already applied) |
| PG10 | 21 | Expired Token (due date in the past) |
| PG11 | 22 | Token Rate Limited |
| PG12 | 23 | Invalid Production Date (can't read from flash) |

## Important Gotchas

1. **`CookerShouldBeLocked()` is currently disabled** — hardcoded to `return false` with a TODO. The credit check that actually prevents cooking is done in `UIModel_IsUserInCredit()` at the UI level, not at the IndCooker level.
2. **`nxp_keycode_payg_credit_add()` is unsupported** — only SET and UNLOCK token types work. ADD returns false.
3. **Token feedback sent from callback, not from `nxp_keycode_feedback_start()`** — because Nexus doesn't use the return value of `credit_set`/`credit_unlock`, the ACK/NACK is sent directly from those functions.
4. **Token queue is size 2** — only 2 tokens can be buffered at a time.
5. **Bluetooth uses PAYGO serial as device name** — `DeviceSettings_GetPaygoSerial()` provides the BLE advertising name.
6. **Shadow updates come from two sources**: Bluetooth (direct JSON) and cellular/MQTT (via AT commands).
7. **Always-on detection**: Year == 99 in due date triggers `STATUS_FLAG_DEVICE_ALWAYS_ON`.
