# Falcon Fusion demo: email on IOM and Cloud risk

Two Falcon Fusion SOAR workflows that send an HTML email each time Falcon Cloud Security raises:

- an **Indicator of Misconfiguration (IOM)**, via the `CloudSecurityAssessment/Configuration` trigger
- a **Cloud risk**, via the `CloudRisk` trigger

GitHub Actions validates the workflows, then imports and releases them into your CID. No addresses or credentials are stored in the repo.

## Quickstart

1. Create a Falcon API client with the **Workflow** scope (read + write).
2. Add these under **Settings > Secrets and variables > Actions**:

   | Name | Type | Value |
   |---|---|---|
   | `FALCON_CLIENT_ID` | secret | API client ID |
   | `FALCON_CLIENT_SECRET` | secret | API client secret |
   | `RECIPIENT_EMAIL` | secret | Where to send the emails. It must be a Falcon user or on a domain approved for the CID. |
   | `FALCON_BASE_URL` | variable (optional) | For example `https://api.us-2.crowdstrike.com`. The default is us-1. |

   Or do it from the CLI:

   ```bash
   gh secret set FALCON_CLIENT_ID
   gh secret set FALCON_CLIENT_SECRET
   gh secret set RECIPIENT_EMAIL
   gh variable set FALCON_BASE_URL --body https://api.us-2.crowdstrike.com   # only if you're not on us-1
   ```

3. Run the deploy:

   ```bash
   gh workflow run "Deploy Fusion workflows" && gh run watch
   ```

After that, every push to `main` that changes `workflows/` redeploys, and pull requests are validated only. Workflows are matched by name, so redeploying replaces them instead of creating duplicates.

> **Volume warning:** neither workflow filters on severity, so they send an email for **every** new IOM and cloud risk in the CID. To cut that down, add a condition node between the trigger and the email action.

### Deploying without GitHub Actions

```bash
git clone https://github.com/CrowdStrike/fusion-skills /tmp/fusion-skills
pip install -r /tmp/fusion-skills/requirements.txt
export FALCON_CLIENT_ID=... FALCON_CLIENT_SECRET=...
mkdir -p build && for f in workflows/*.yaml; do sed 's|__RECIPIENT_EMAIL__|you@yourco.com|' "$f" > "build/${f##*/}"; done
python /tmp/fusion-skills/skills/authoring/scripts/validate.py build/*.yaml
python /tmp/fusion-skills/skills/deployment/scripts/import_workflows.py build/iom-to-email.yaml --replace   # prints "Imported — ID: <id>"
python /tmp/fusion-skills/skills/deployment/scripts/release_workflow.py --id <id>                         # run once per workflow
```

## What's in the email

| IOM email | Cloud risk email |
|---|---|
| Policy, severity, disposition | Rule name and description, severity, status |
| Cloud provider and service | Account **name** and ID |
| AWS account / Azure subscription / GCP parent ID | Cloud provider and region |
| Region, resource ID and type | Asset name and type |
| Finding, remediable flag, first detected | Risk factors, risk summary, first seen |
| Link to the Falcon console | Links to the asset and the Falcon console |

## Trigger payload limits

These come from the live trigger schemas (`trigger_search.py --fields`):

- The **IOM trigger** has account IDs but **no account name** and **no compliance framework or benchmark**.
- The **Cloud risk trigger** has the account name but **no compliance framework**.
- `Trigger.CloudRisk.Remediations.Title` is on a list item. Referencing it directly fails at release with `unknown variable`.

To get framework or benchmark data, add a CrowdStrike HTTP Request action that calls `GET /cloud-security-evaluations/entities/ioms/v1` for the event.

## Known rough edges

- For IOMs, `Policy` sometimes shows a numeric ID and `PolicyID` shows `0`.
- The ID fields for other clouds show `null` (for example, the Azure and GCP rows on an AWS finding).
- On Cloud risks, `RiskSummary` comes through as raw JSON, and `RiskFactorNames` can be `null`.
