# OCI Always Free Auto-Retry

This is a tool for automatically launching a `VM.Standard.A1.Flex` (ARM Ampere) instance within the OCI (Oracle Cloud Infrastructure) Always Free tier.

If the instance cannot be launched immediately due to Availability Domain resource exhaustion ("Out of host capacity"), GitHub Actions cron will attempt to launch it **every 5 minutes** and stop once successful.

---

## 🎯 Problem This Solves

- OCI Always Free ARM Ampere (`VM.Standard.A1.Flex`) instances are highly popular and often cannot be created immediately due to AD resource exhaustion.
- Manually repeatedly clicking the “Launch” button is impractical.
- This tool automatically retries every 5 minutes on GitHub Actions and secures the instance the moment resources become available.

---

## 📁 File Structure

```text
oci-freenium-retry/
├── .github/workflows/oci-retry.yml   # GitHub Actions definition (cron + workflow_dispatch)
├── scripts/
│   ├── launch.py                      # Main script
│   └── requirements.txt              # Python dependencies (oci, requests)
├── state.example.json                 # Sample state.json
├── .gitignore                         # Excludes state.json from Git tracking
├── README.md                          # This file
└── docs/
    └── architecture.md                # Mermaid architecture diagram + detailed design
```

---

## 🔑 Steps to Generate the Required OCI API Key

Generate an API key in the OCI Console and prepare the following information.

### 1. Confirm Tenancy OCID / User OCID

1. In the top-right profile menu of the OCI Console, open **Tenancy**, then copy the **OCID** (`ocid1.tenancy.oc1.....`).
2. Open Profile → **My profile** → **My profile**, then copy the user **OCID** (`ocid1.user.oc1.......`).

### 2. Add API Key

1. Open Profile → **My profile** → **API keys**.
2. Click **Add API key** → **Generate API key pair**.
3. Download the PEM file using **Download private key** (example: `oci_api_key.pem`). **This file cannot be reissued. Store it securely.**
4. Click **Add**, and the **Configuration file preview** will be displayed. Note the following values:
   - `tenancy` (Tenancy OCID)
   - `user` (User OCID)
   - `fingerprint` (API key fingerprint)
   - `region` (example: `ap-tokyo-1`)
   - `key_file` (local path, not used in CI)

### 3. Confirm Fingerprint

Save the `fingerprint` shown after adding the API key (`xx:xx:xx:...` format). This will be registered later in Secrets.

### 4. Get the Contents of the PEM Private Key

Open the downloaded `.pem` file in a text editor and copy **all lines** from `-----BEGIN PRIVATE KEY-----` to `-----END PRIVATE KEY-----`. Register this as the GitHub Secret `OCI_API_KEY_CONTENT`.

> ⚠️ Never commit the private key to the Git repository. `*.pem` is already excluded in `.gitignore`.

---

## 🔐 GitHub Secrets List

Register the following in the repository under **Settings → Secrets and variables → Actions → New repository secret**.

| Secret Name | Value | Example |
|---|---|---|
| `OCI_TENANCY_OCID` | Tenancy OCID | `ocid1.tenancy.oc1..aaaa...` |
| `OCI_USER_OCID` | User OCID | `ocid1.user.oc1..aaaa...` |
| `OCI_COMPARTMENT_OCID` | Compartment OCID (same as Tenancy OCID if using root) | `ocid1.tenancy.oc1..aaaa...` |
| `OCI_REGION` | Region identifier | `ap-tokyo-1` |
| `OCI_VCN_OCID` | VCN OCID | `ocid1.vcn.oc1.ap-tokyo-1.aaaa...` |
| `OCI_SUBNET_OCID` | Subnet OCID | `ocid1.subnet.oc1.ap-tokyo-1.aaaa...` |
| `OCI_API_KEY_CONTENT` | **All lines** of the PEM private key | `-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----` |
| `OCI_API_KEY_FINGERPRINT` | API key fingerprint | `12:34:56:78:...` |
| `OCI_SSH_PUBLIC_KEY` | SSH public key to register with the instance | `ssh-ed25519 AAAA... user@host` |
| `DISCORD_WEBHOOK_URL` | Discord Webhook URL for success/error notifications | `https://discordapp.com/api/webhooks/...` |

### Variables (Optional — OK to leave unregistered if default values are acceptable)

These can be registered under **Settings → Secrets and variables → Actions → Variables**.

| Variable Name | Default | Description |
|---|---|---|
| `OCI_SHAPE` | `VM.Standard.A1.Flex` | Instance shape |
| `OCI_OCPUS` | `2` | Number of OCPUs |
| `OCI_MEMORY_GB` | `12` | Memory (GB) |
| `OCI_DISPLAY_NAME` | `auto-retry-freenium` | Instance display name |
| `OCI_OS` | `Canonical Ubuntu` | OS name |
| `OCI_OS_VERSION` | `24.04` | OS version |
| `OCI_ARCH` | `ARM` | Architecture |
| `OCI_BOOT_VOLUME_SIZE_GB` | `50` | Boot volume size (GB) |

---

## 🚀 Initial Deployment Steps

1. **Push to repository**

   ```bash
   git init
   git add .
   git commit -m "feat: OCI Always Free auto-retry"
   git branch -M main
   git remote add origin <YOUR_REPO_URL>
   git push -u origin main
   ```

2. **Register Secrets**: Register all items listed in the “GitHub Secrets List” above.

3. **Enable Actions**
   - Open the repository’s **Actions** tab.
   - If the workflow is disabled on first use, click **"I understand my workflows, go ahead and enable them"**.

4. **Verify operation with a manual run**
   - Click Actions → **OCI Always Free Auto-Retry** → **Run workflow**.
   - Check the logs. If it exits with `Out of host capacity` and exit code 0, it is working correctly and will retry on the next cron run.

---

## 🔧 Operation

### Automatic execution every 5 minutes

It runs automatically with `cron: "*/5 * * * *"`. No action is required.

### Manual execution

You can run it at any time from the Actions tab → **Run workflow**.

### Reset state (if you want to retry again)

If you delete the instance and want the tool to retry again, reset `state.json`:

```bash
# Reset state.json to the initial state
cp state.example.json state.json
git add -f state.json
git commit -m "chore(state): reset state.json"
git push
```

On the next cron execution, it will attempt to launch again.

---

## 🩺 Troubleshooting

### "Out of host capacity" does not resolve

- **Change region**: `ap-tokyo-1` is especially congested. Consider `us-ashburn-1` / `us-phoenix-1` / `ap-osaka-1`, etc.
- **Increase AD options**: Regions with multiple ADs have a higher chance than single-AD regions.
- **Change time of day**: Late night to early morning in Japan time may be relatively less congested.
- **Try launching manually in the OCI Console**: There may be a restriction on the account itself. The Always Free A1 allocation is up to 4 OCPU / 24GB per tenancy.

### Workflow does not run

- Check whether the workflow is enabled in the Actions tab.
- `cron` may be delayed by up to 10–15 minutes depending on GitHub load. Immediate execution is not guaranteed.
- Check whether it can be run manually with `workflow_dispatch`.

### state.json is not committed

- Check whether `permissions: contents: write` is set in the workflow. This repository already has it set.
- Check whether branch protection rules are blocking pushes from `GITHUB_TOKEN`. This is usually not a problem for personal repositories.
- Check whether `persist-credentials: true` exists in the checkout step. This repository already has it set.

### Discord notifications are not received

- Check whether `DISCORD_WEBHOOK_URL` is correct.
- Check whether the Webhook URL has been disabled, and verify the Discord channel settings.

### API authentication error

- Check whether `OCI_API_KEY_FINGERPRINT` and `OCI_API_KEY_CONTENT` are the correct pair.
- Check whether the group containing the user has the appropriate policy, such as `allow group ... to manage compute in tenancy`.

---

## 📚 Related Documentation

- [Design / Architecture](docs/architecture.md)
- [OCI Python SDK Documentation](https://docs.oracle.com/en-us/iaas/tools/python/latest/)
- [OCI Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
