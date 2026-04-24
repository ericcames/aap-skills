---
name: aap-setup-demo
description: "Bootstrap a fresh AAP instance AND run the Setup - AAP - CAC job template as part of a live demo story. Use this when the setup process itself is the demo. TRIGGER when: user wants to demonstrate the AAP setup/CaC process to a customer, or run setup as part of a demo. SKIP: if the user just wants to bootstrap quietly before running a separate demo — use /aap-bootstrap instead."
---

# AAP Setup Demo Skill

This skill bootstraps a fresh Ansible Automation Platform instance and then launches the `Setup - AAP - CAC` job template — making the setup process itself a live demo story for a customer.

## Overview

This skill extends `/aap-bootstrap` with one additional step: after bootstrap completes it launches `Setup - AAP - CAC` in AAP and monitors the job until completion.

The full sequence:
1. Preflight — verify local prerequisites and check AAP bootstrap state
2. Bootstrap AAP if needed (Hub creds, Vault cred, project, job template)
3. Launch `Setup - AAP - CAC` from AAP
4. Monitor and report job status

## Preflight Check — Part 1: Local Prerequisites

Before doing anything else, verify local prerequisites are in place:

```bash
# ansible.cfg with Hub token
grep -q "galaxy_server.rh_certified" ~/.ansible/ansible.cfg 2>/dev/null && \
  grep -v "PASTE_YOUR_TOKEN_HERE" ~/.ansible/ansible.cfg | grep -q "token=" && \
  echo "✅ ansible.cfg" || echo "❌ ansible.cfg"

# secrets2
test -s ~/.ansible/secrets2 && echo "✅ secrets2" || echo "❌ secrets2"

# collections
test -d ./collections/ansible_collections/ansible/platform && \
  echo "✅ ansible.platform" || echo "❌ ansible.platform"

test -d ./collections/ansible_collections/ansible/controller && \
  echo "✅ ansible.controller" || echo "❌ ansible.controller"
```

If any check fails, stop immediately:

```
❌ Prerequisites missing: <list each failing item>

Run /aap-first-time to set up these prerequisites before continuing.
```

## Preflight Check — Part 2: AAP Bootstrap State

Resolve AAP credentials using the same priority order as `/aap-bootstrap` (env vars → ansible.cfg → prompt). Then check whether AAP has already been bootstrapped:

```bash
# Check for project
PROJECT_COUNT=$(curl -s -k \
  -u $CONTROLLER_USERNAME:$CONTROLLER_PASSWORD \
  "$CONTROLLER_HOST/api/controller/v2/projects/?name=aap.as.code" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['count'])")

# Check for job template
JT_COUNT=$(curl -s -k \
  -u $CONTROLLER_USERNAME:$CONTROLLER_PASSWORD \
  "$CONTROLLER_HOST/api/controller/v2/job_templates/?name=Setup+-+AAP+-+CAC" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['count'])")
```

- If both exist (`PROJECT_COUNT >= 1` and `JT_COUNT >= 1`): tell the user "AAP is already bootstrapped — skipping to job launch" and jump directly to Step 7.
- If either is missing: proceed through Steps 1–6 to run the bootstrap first.

## AAP API Paths (2.5+)

AAP 2.5+ uses a gateway architecture. Full endpoint list is in [`skills/references/aap-as-code-context.md`](../references/aap-as-code-context.md#aap-api-endpoints-25). This skill uses:

| Purpose | Base path |
|---------|-----------|
| Token create/delete | `$CONTROLLER_HOST/api/gateway/v1/tokens/` |
| Job templates | `$CONTROLLER_HOST/api/controller/v2/job_templates/` |
| Jobs | `$CONTROLLER_HOST/api/controller/v2/jobs/` |

## Steps 1–6: Bootstrap First

Follow all steps from the `/aap-bootstrap` skill exactly (credential resolution, inventory generation, playbook execution).

Do not proceed to Step 7 until the bootstrap playbook exits successfully.

## Audience Selection

Before launching the job, ask the presenter:

> "Who is your audience today?"
> 1. Technical (architect / engineer)
> 2. Management (IT director / VP)
> 3. Mixed room

Based on their answer, provide the appropriate narration track to use while the job runs:

### Technical track
Emphasize:
- **Declarative config** — the entire AAP environment is defined as code in Git; no clicking through the UI
- **Idempotency** — run this playbook ten times and you get the same result every time
- **Dependency ordering** — the `infra.aap_configuration.dispatch` role creates objects in the correct order automatically (orgs before projects, credentials before templates)
- **GitOps pattern** — any change to the environment goes through a pull request; the Git history is your audit trail
- **Version control** — roll back the entire AAP configuration with a `git revert`

### Management track
Emphasize:
- **Time savings** — what used to take hours of clicking through the UI now takes minutes
- **Consistency** — every environment is built exactly the same way, every time; no "it works on my AAP" problems
- **No manual errors** — humans don't typo YAML; the config is reviewed in Git before it's applied
- **Self-service** — once this is set up, the team can provision new demo environments without waiting on anyone
- **Audit trail** — every change is tracked in Git with who made it and why

### Mixed room
Lead with outcome, follow with one technical detail, close with business impact:
- **Open**: "Watch how fast we can stand up a fully configured AAP environment from scratch."
- **Middle**: "Everything you're seeing get created is defined as code in a Git repo — it's not a script, it's a declaration of what the environment should look like."
- **Close**: "When this job finishes, your team can do this themselves in minutes instead of spending hours in the UI — and it'll be identical every time."

## Step 7 — Launch Setup - AAP - CAC

After a successful bootstrap, launch the job template via the AAP API.

First, create a session token via the gateway:
```bash
TOKEN_JSON=$(curl -s -k -X POST \
  -H "Content-Type: application/json" \
  -u $CONTROLLER_USERNAME:$CONTROLLER_PASSWORD \
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
