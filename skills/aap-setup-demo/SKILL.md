---
name: aap-setup-demo
description: "Bootstrap a fresh AAP instance AND run the Setup - AAP - CAC job template as part of a live demo story. Use this when the setup process itself is the demo. TRIGGER when: user wants to demonstrate the AAP setup/CaC process to a customer, or run setup as part of a demo. SKIP: if the user just wants to bootstrap quietly before running a separate demo — use /aap-bootstrap instead."
---

# AAP Setup Demo Skill

This skill bootstraps a fresh Ansible Automation Platform instance and then launches the `Setup - AAP - CAC` job template — making the setup process itself a live demo story for a customer.

## Overview

This skill extends `/aap-bootstrap` with one additional step: after bootstrap completes it launches `Setup - AAP - CAC` in AAP and monitors the job until completion.

The full sequence:
1. Bootstrap AAP (Hub creds, Vault cred, project, job template)
2. Launch `Setup - AAP - CAC` from AAP
3. Monitor and report job status

## AAP API Paths (2.5+)

AAP 2.5+ uses a gateway architecture. Always use these base paths:

| Purpose | Base path |
|---------|-----------|
| Token create/delete | `$CONTROLLER_HOST/api/gateway/v1/tokens/` |
| Job templates | `$CONTROLLER_HOST/api/controller/v2/job_templates/` |
| Jobs | `$CONTROLLER_HOST/api/controller/v2/jobs/` |

Verify the API layout first if unsure:
```bash
curl -s -k "$CONTROLLER_HOST/api/" | python3 -m json.tool
```

## Steps 1–6: Bootstrap First

Follow all steps from the `/aap-bootstrap` skill exactly (credential resolution, inventory generation, playbook execution).

Do not proceed to Step 7 until the bootstrap playbook exits successfully.

## Step 7 — Launch Setup - AAP - CAC

After a successful bootstrap, launch the job template via the AAP API.

First, create a session token via the gateway:
```bash
TOKEN_JSON=$(curl -s -k -X POST \
  -H "Content-Type: application/json" \
  -u admin:$CONTROLLER_PASSWORD \
  "$CONTROLLER_HOST/api/gateway/v1/tokens/" \
  -d '{"description":"setup-demo token","scope":"write"}')

TOKEN=$(echo $TOKEN_JSON | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['token'])")
TOKEN_ID=$(echo $TOKEN_JSON | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['id'])")
```

Get the job template ID and launch it:
```bash
# Get the job template ID
JT_ID=$(curl -s -k \
  -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/job_templates/?name=Setup+-+AAP+-+CAC" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Launch it
JOB_ID=$(curl -s -k \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$CONTROLLER_HOST/api/controller/v2/job_templates/$JT_ID/launch/" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
```

Show the job URL to the user:
```
$CONTROLLER_HOST/#/jobs/$JOB_ID/output
```

## Step 8 — Monitor the Job

Poll the job status every 15 seconds:
```bash
curl -s -k \
  -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/jobs/$JOB_ID/" \
  | python3 -c "import sys,json; j=json.load(sys.stdin); print(j['status'], j.get('finished','running'))"
```

Report status updates to the user. Stop polling when status is `successful` or `failed`.

## Step 9 — Clean Up Token

Always delete the session token after the job completes or fails:
```bash
curl -s -k -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/gateway/v1/tokens/$TOKEN_ID/"
```

## Step 10 — Report Results

On success:
- Confirm that `Setup - AAP - CAC` completed
- Tell the user AAP is fully configured and the demo is ready to run

On failure:
- Show the job failure message
- Provide the direct link to the job output in AAP for troubleshooting
