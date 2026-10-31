# Gym WhatsApp Retention CRM (n8n + Meta Cloud API + Google Sheets)

An automated, production-ready WhatsApp retention and lifecycle CRM built on **n8n**, **Google Sheets**, and the **Meta WhatsApp Cloud API**. Engineered specifically for gyms and fitness centers in Egypt (`Africa/Cairo` timezone, native Egyptian `DD/MM/YYYY` date parsing, and cost-optimized Arabic Utility/Marketing templates).

---

## Architecture & Workflow Suite

| Workflow | Trigger Schedule | Target Segment | Meta Templates Used | Tier |
| :--- | :--- | :--- | :--- | :--- |
| **W1: Onboarding** | Daily at `10:00 AM` | Joined `1`, `3`, or `7` days ago | `day1_welcome`, `day3_tips`, `day7_followup` | **Utility** |
| **W2: At-Risk / Absence** | Daily at `11:00 AM` | Active subscription, absent `7`, `14`, or `30` days | `w2_send_day_7`, `w2_send_day_14`, `w2_send_day_30` | **Utility** |
| **W3: Expiry Reminders** | Daily at `12:00 PM` | Expires in `7` days, `0` days (today), or `-3` days ago | `w3_send_7_days_before_7`, `w3_send_day_of_expiry_0`, `w3_send_3_days_after_3` | **Utility** |
| **W4: Win-Back Offers** | Daily at `02:00 PM` | Expired `45`, `48`, or `50` days ago | `w4_send_day_45_offer`, `w4_send_day_48_reminder`, `w4_send_day_50_urgency` | **Marketing** |
| **W5: Referral Engine** | Daily at `03:00 PM` | `Visits_This_Month >= 10` & `Referral_Asked != TRUE` | `w5_send_referral_ask` | **Marketing** |
| **W6: Google Reviews** | Fridays at `04:00 PM` | `Visits_This_Month >= 12` & `Review_Asked != TRUE` | `w6_send_review_request` | **Marketing** |

---

## 1. Quick Start (Ubuntu / WSL2 with Docker Compose)

### Prerequisites
- Windows 10/11 with **WSL2 (Ubuntu)** and **Docker Desktop** (with WSL2 integration enabled), or a native Ubuntu Linux server.

### Step 1: Configure Environment Variables
```bash
cp .env.example .env
```
Open `.env` and fill in your values:
1. Generate an encryption key:
   ```bash
   openssl rand -hex 16
   ```
   Paste the output into `N8N_ENCRYPTION_KEY`.
2. Set `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_BUSINESS_ACCOUNT_ID`, and `GOOGLE_SHEET_ID`.

### Step 2: Launch the Stack
```bash
docker compose up -d
```
Verify the container is healthy:
```bash
docker compose ps
```

### Step 3: Import the Sanitized Workflows
Import all 6 workflows into n8n with a single command:
```bash
docker exec -u node n8n-gym-crm n8n import:workflow --input=/workflows/all_gym_workflows.json
```
*(Or import individual files from `./workflows/W1.json` .. `W6.json` via the n8n UI at `http://localhost:5678`)*

---

## 2. Google Sheets Database Schema & OAuth2 Setup

### Required Spreadsheet Columns
Create a Google Sheet with a tab named `Sheet 1` (or match your `GOOGLE_SHEET_NAME` in `.env`) and the following exact column headers in **Row 1**:

| Column Header | Format / Example | Used By | Notes |
| :--- | :--- | :--- | :--- |
| `Name` | `أحمد محمد` | W1–W6 | Injected into `{{1}}` template parameter |
| `Phone` | `201012345678` | W1–W6 | **Must include country code without `+` or leading `00`** |
| `Join Date` | `21/09/2026` | W1 | Supports `DD/MM/YYYY`, `YYYY-MM-DD`, or Excel serials |
| `Last Visit Date` | `15/09/2026` | W2 | Supports `DD/MM/YYYY`, `YYYY-MM-DD`, or Excel serials |
| `Expiry Date` | `29/09/2026` | W2, W3, W4 | Supports `DD/MM/YYYY`, `YYYY-MM-DD`, or Excel serials |
| `Visits_This_Month` | `12` | W5, W6 | Integer count of check-ins this month |
| `Referral_Asked` | `TRUE` / empty | W5 | Automatically updated to `TRUE` after W5 sends |
| `Review_Asked` | `TRUE` / empty | W6 | Automatically updated to `TRUE` after W6 sends |

### Google Cloud OAuth2 Setup
1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project and navigate to **APIs & Services -> Library**.
3. Enable both **Google Sheets API** and **Google Drive API**.
4. Go to **APIs & Services -> OAuth consent screen**:
   - Select **External**.
   - Fill in the app name (`Gym CRM n8n`) and your support email.
   - Under **Test users**, click **Add Users** and add your Gmail address. *(Tip: Click "Publish App" to move from Testing to Production so your OAuth token never expires after 7 days).*
5. Go to **APIs & Services -> Credentials -> Create Credentials -> OAuth client ID**:
   - Application type: **Web application**
   - **Authorized redirect URIs**:
     ```text
     http://localhost:5678/rest/oauth2-credential/callback
     ```
     *(If using a custom domain/tunnel, replace `http://localhost:5678` with your `WEBHOOK_URL`)*
6. Copy the **Client ID** and **Client Secret**.
7. In n8n (`http://localhost:5678`), go to **Credentials -> Add Credential -> Google Sheets OAuth2 API**, paste the Client ID and Client Secret, and click **Sign in with Google**.

---

## 3. Meta WhatsApp Cloud API Setup

### Step 1: Create a Meta Developer App
1. Go to [developers.facebook.com](https://developers.facebook.com/) -> **My Apps -> Create App**.
2. Select **Other -> Business** and connect it to your Meta Business Portfolio.
3. Add the **WhatsApp** product to your app.

### Step 2: Sandbox vs. Production Numbers
- **Free Sandbox Testing ($0 Cost):**
  - Navigate to **WhatsApp -> API Setup**.
  - Copy the **Phone number ID** and **WhatsApp Business Account ID** into your `.env` file.
  - Under the **To** field, click **Manage phone number list** and add up to 5 personal phone numbers (verified via OTP). *Note: Sandbox numbers can ONLY send messages to verified test numbers.*
- **Production Setup:**
  - Add your gym's real phone number in **WhatsApp Manager -> Phone Numbers** and add a valid payment method inside Meta Business Settings.

### Step 3: Generate a Permanent System User Access Token
Temporary tokens in the Meta API Setup page expire every **24 hours**. For production:
1. Open [Meta Business Settings](https://business.facebook.com/settings/) -> **Users -> System Users**.
2. Click **Add**, create an **Admin** System User (e.g., `n8n_crm_bot`).
3. Click **Assign Assets**:
   - Select **Apps** -> Choose your App -> Toggle **Manage app (Full control)**.
   - Select **WhatsApp Accounts** -> Choose your WABA -> Toggle **Manage WhatsApp Account (Full control)**.
4. Click **Generate New Token**:
   - Select your App.
   - Set Token Expiration to **Never**.
   - Check permissions: `whatsapp_business_messaging` and `whatsapp_business_management`.
5. Copy the token. In n8n, go to **Credentials -> Add Credential -> WhatsApp API**, and paste your **WhatsApp Business Account ID** and **Permanent Access Token**.

### Step 4: Create & Approve Message Templates
Every workflow (`W1`–`W6`) includes embedded yellow/green **Sticky Notes** directly on the n8n canvas containing the exact Arabic copy, variable placeholders (`{{1}}`, `{{2}}`), and recommended category (**Utility** vs **Marketing**).
1. Open [WhatsApp Manager -> Message Templates](https://business.facebook.com/wa/manage/message-templates/).
2. Create each of the 14 templates listed in the table above using language **Arabic (`ar`)** and the exact template name.
3. Wait for status to turn **Active (Green)** (usually 1–5 minutes for Utility/Marketing templates).

---

## 4. Webhook Verification (Inbound Replies & Delivery Receipts)

If you extend the workflows to listen for inbound customer replies (e.g., when a member replies `"موافق"` to W4 or `"جاي"` to W3):

1. **Expose Local n8n over HTTPS** (Required by Meta):
   Run ngrok or Cloudflare Tunnel inside WSL2:
   ```bash
   ngrok http 5678
   ```
   Copy the HTTPS URL (e.g., `https://abcd-1234.ngrok-free.app`) and update `WEBHOOK_URL` in `.env`, then restart Docker:
   ```bash
   docker compose up -d
   ```
2. **Configure Meta Webhook Callback**:
   - In Meta App Dashboard -> **WhatsApp -> Configuration -> Webhook**, click **Edit**.
   - **Callback URL**: Paste your n8n WhatsApp Trigger / Webhook Production URL.
   - **Verify Token**: Enter the value of `WHATSAPP_WEBHOOK_VERIFY_TOKEN` from your `.env`.
   - Click **Verify and save**.
   - Click **Manage** under Webhook fields and subscribe to **`messages`**.

---

## 5. Final Activation Checklist
After importing workflows via CLI, n8n unlinks credential dropdowns for security:
1. Open each workflow (`W1` through `W6`) in the n8n UI.
2. Open the **Get Members** (and **Mark as Asked in Sheet**) nodes and select your **Google Sheets account** credential from the dropdown.
3. Open each **Send ...** HTTP Request node and select your **WhatsApp API** credential from the dropdown.
4. Click **Save** and toggle the workflow **Active** switch in the top-right corner.
