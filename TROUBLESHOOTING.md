# Troubleshooting & Operations Runbook

This guide resolves 100% of known edge cases across **n8n**, **Meta WhatsApp Cloud API**, **Google Sheets OAuth2**, and **Docker/WSL2**.

---

## 1. n8n Workflow & Execution Errors

### ❌ Error: `Module 'luxon' is disallowed [line 1]`
- **Cause:** Modern n8n Docker images sandbox the Code node and block `require('luxon')` by default unless `NODE_FUNCTION_ALLOW_BUILTIN` / `NODE_FUNCTION_ALLOW_EXTERNAL` are overridden.
- **Resolution:** Already resolved in all release workflows (`W1`–`W6`). All Code nodes use 100% native JavaScript (`Intl` / `toLocaleString("en-US", { timeZone: "Africa/Cairo" })`) with zero external module dependencies.

### ❌ Error: `Credentials not found` after CLI Import
- **Cause:** When workflows are imported via `n8n import:workflow`, n8n strips or disconnects internal credential IDs if they don't match the target database's generated UUIDs.
- **Resolution:**
  1. Open each workflow (`W1`–`W6`) in the n8n editor (`http://localhost:5678`).
  2. Double-click the **Google Sheets** node(s) and select your **Google Sheets OAuth2 API** credential from the dropdown.
  3. Double-click each **HTTP Request (`Send ...`)** node and select your **WhatsApp API** credential under *Predefined Credential Type -> WhatsApp API*.
  4. Save the workflow (`Ctrl + S`).

### ❌ Error: `"The execution took a different path and didn't reach this node. This typically happens with conditional nodes like If or Switch."`
- **Resolution:**
  - All Switch nodes in `./workflows/` have been explicitly patched with `"operation": "equal"` and explicit `value2` integers.
  - All Code nodes include a custom `parseDateStr()` parser supporting Egyptian `DD/MM/YYYY`, `YYYY-MM-DD`, and Excel serial dates.

### ❌ Error: `Access to env vars denied` or `{{ $env.GOOGLE_SHEET_ID }}` evaluates to `undefined`
- **Resolution:**
  1. Ensure `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` is set in `.env`.
  2. Ensure `GOOGLE_SHEET_ID`, `GOOGLE_SHEET_NAME`, `WHATSAPP_PHONE_NUMBER_ID`, and `WHATSAPP_API_VERSION` are filled in `.env`.
  3. Recreate the container (`docker compose up -d --force-recreate`).

---

## 2. Meta WhatsApp Cloud API Errors

### ❌ Error `#132001`: `Template name does not exist in the translation`
- **Resolution:**
  1. In [WhatsApp Manager](https://business.facebook.com/wa/manage/message-templates/), ensure you are creating templates under the exact WABA that owns your `WHATSAPP_PHONE_NUMBER_ID`.
  2. Ensure the template name matches the workflow Sticky Note character-for-character and language is set to **`ar` (Arabic)**.

### ❌ Error `#131030`: `Recipient phone number not in allowed list`
- **Resolution:**
  - Verify the test phone number via WhatsApp OTP in **Meta App Dashboard -> WhatsApp -> API Setup**.
  - Ensure the `Phone` cell in Google Sheets is formatted as `2010XXXXXXXX` (no `+`, no leading `00`).

### ❌ Error `#132000`: `Number of parameters does not match the expected number of params`
- **Resolution:**
  - All templates use a single `{{1}}` variable (`Name`), **except** `w3_send_7_days_before_7`, which uses `{{1}}` (`Name`) and `{{2}}` (`Expiry Date`).

### ❌ Error `#190`: `Error validating access token: Session has expired`
- **Resolution:** Generate a non-expiring System User token in Meta Business Settings with `whatsapp_business_messaging` and `whatsapp_business_management` permissions.

---

## 3. Google Sheets OAuth2 & Data Errors

### ❌ Error `400: redirect_uri_mismatch` during Google Sign-In
- **Resolution:** Ensure `http://localhost:5678/rest/oauth2-credential/callback` is listed under **Authorized redirect URIs** in Google Cloud Console.

### ❌ Error `invalid_grant` (Token Expired after 7 Days)
- **Resolution:** Go to **Google Cloud Console -> APIs & Services -> OAuth consent screen** and click **Publish App** to set status to **In production**.

---

## 4. Webhook Verification & WSL2 Networking

### ❌ Container Fails to Start with `EACCES: permission denied, open '/home/node/.n8n/config'`
- **Resolution:**
  ```bash
  docker compose run --rm -u root n8n chown -R node:node /home/node/.n8n
  docker compose up -d
  ```
