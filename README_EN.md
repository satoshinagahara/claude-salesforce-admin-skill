# Claude Code — Salesforce Admin Skill

A Salesforce administration skill for Claude Code. Using Salesforce CLI (`sf`), you can perform data operations, metadata operations, LWC creation, and more simply by giving natural language instructions.

---

## ⚠️ Recommended Environments (Important)

> **This skill is recommended for use in Developer Edition or Sandbox environments.**

This skill performs direct operations on your Salesforce org via the Salesforce CLI. **Using it in a Production environment requires a thorough understanding and careful operation.**

| Environment | Recommendation | Notes |
|---|---|---|
| **Developer Edition** | ✅ Recommended | Free to obtain. No impact on production data. |
| **Sandbox** | ✅ Recommended | A test environment that mirrors production settings. |
| **Production** | ⚠️ Advanced users only | Mistakes have direct business impact. Auto safety checks apply, but proceed with caution. |

You can get a free Developer Edition org at [Salesforce Developer Signup](https://developer.salesforce.com/signup).

---

## Key Features

| Category | Capabilities |
|---|---|
| **Data Operations (DML)** | Create, update, delete records (single / bulk CSV) |
| **Custom Objects & Fields** | Create, modify, delete, configure relationships |
| **Page Layouts & Record Types** | Modify layouts, add sections, arrange fields |
| **Permission Sets & Profiles** | FLS settings, object permissions, user assignments |
| **Apex, Flows & Automation** | Create classes/triggers, enable/disable flows |
| **Lightning Web Components** | Create and deploy components |

---

## Prerequisites

1. [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) (`sf`) must be installed.
2. A **Connected App** must be created in your Salesforce org.
3. A configuration file (`sf-config.json`) for JWT authentication must be prepared.

---

## Salesforce Setup

This skill connects to Salesforce using **JWT Bearer Flow**. The following setup is required only once.

### Step 1: Generate an RSA Key Pair

Run the following in your terminal:

```bash
# Create a working directory
mkdir -p ~/sf-jwt-keys && cd ~/sf-jwt-keys

# Generate a private key and self-signed certificate (valid for 365 days)
openssl req -x509 -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -days 365 -nodes \
  -subj "/CN=salesforce-jwt"
```

Generated files:
- `server.key` — Private key (**never share this publicly**)
- `server.crt` — Certificate (to be uploaded to Salesforce)

### Step 2: Create a Connected App in Salesforce

1. Open **Setup** in Salesforce.
2. Search for **App Manager** in the Quick Find box.
3. Click **New Connected App**.
4. Fill in the following:

| Field | Value |
|---|---|
| Connected App Name | `Claude Code JWT` (or any name) |
| API Name | `Claude_Code_JWT` |
| Contact Email | Your email address |
| Enable OAuth Settings | ✅ Check |
| Callback URL | `http://localhost:1717/OauthRedirect` |
| OAuth Scopes | Add `Full access (full)` |
| Use Digital Signatures | ✅ Check → Upload `server.crt` |

5. After saving, note the **Consumer Key**.

### Step 3: Configure Connected App Policies

1. Open the app you created and click **Manage Policies**.
2. Under **OAuth Policies**, set "Permitted Users" to `Admin approved users are pre-authorized`.
3. In the **Profiles** tab, add the profiles allowed to connect (e.g., `System Administrator`).

> ⚠️ Without this policy setting, JWT authentication will result in an `INVALID_LOGIN` error.

### Step 4: Create sf-config.json

```json
{
  "consumer_key": "paste your consumer key here",
  "key_file_path": "/Users/yourname/sf-jwt-keys/server.key",
  "username": "your-username@example.com",
  "instance_url": "https://your-domain.my.salesforce.com"
}
```

> **instance_url** must be your My Domain URL (`https://xxx.my.salesforce.com`). Using `login.salesforce.com` will cause JWT authentication to fail.

Place the config file in one of the following locations (checked in order):
1. `~/sf-config.json` (home directory)
2. `sf-config.json` in the current directory

### Step 5: Test the Connection

```bash
sf org login jwt \
  --client-id "YOUR_CONSUMER_KEY" \
  --jwt-key-file ~/sf-jwt-keys/server.key \
  --username your-username@example.com \
  --instance-url https://your-domain.my.salesforce.com

# On success, org info will be displayed
sf org display --target-org your-username@example.com
```

---

## Installation

Clone this repository into Claude Code's skills directory.

```bash
git clone https://github.com/satoshinagahara/claude-salesforce-admin-skill \
  ~/.claude/skills/salesforce-admin
```

---

## Usage

Type `/salesforce-admin` in Claude Code, or give natural language instructions like:

```
Add a text field called "Department" to the Account object.

Import 100 records from a CSV into the Account object.

Add a new section to the Opportunity page layout and arrange 3 custom fields.

Create a custom object named BOM.
```

---

## File Structure

```
salesforce-admin/
├── SKILL.md                       # Skill definition (loaded by Claude Code)
├── README.md                      # Japanese README
├── README_EN.md                   # This file (English README)
└── references/
    ├── data-dml.md                # Data operation reference
    ├── metadata-objects.md        # Object & field operations
    ├── metadata-ui.md             # Page layouts & record types
    ├── metadata-security.md       # Permission sets & profiles
    ├── metadata-automation.md     # Apex, flows & automation
    ├── metadata-lwc.md            # Lightning Web Components
    └── safety-production.md       # Production safety protocol
```

---

## Production Safety Features

When operating on a production org, the protocol in `safety-production.md` is automatically applied:

- **Validate** is run before any metadata changes and results are presented to the user.
- **Backups** are taken before destructive operations (deletions, bulk updates).
- **Impact scope is presented to the user** before execution, with explicit confirmation required.

---

## Known Limitations

- **Product2 cannot be a Master-Detail parent** → Use Lookup + SetNull instead.
- **`--metadata` flag cannot be used with source-backed components** → Use `--source-dir`.
- **FLS must be configured after deploying custom fields** → Grant via permission sets.
- **Master-Detail fields cannot be included in fieldPermissions** → Exclude them.

---

## Disclaimer

This skill performs direct operations on your Salesforce org via the Salesforce CLI. By using this skill, you agree to the following:

- The author assumes **no responsibility** for any data loss, corruption, unintended modifications, or any other damages resulting from the use of this skill.
- As this skill involves automated operations by AI (Claude), unintended actions may be executed. Use in a **Production environment is entirely at your own risk** and should only be done with a thorough understanding of the implications.
- This skill is provided **as-is**, with no warranties of any kind regarding accuracy, completeness, or fitness for a particular purpose.
- The skill may stop working without notice due to Salesforce specification changes or API updates.

**Use at your own risk.**

---

## License

MIT
