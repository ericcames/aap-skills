---
name: aap-bootstrap
description: "Bootstrap a fresh Ansible Automation Platform (AAP) instance from Red Hat Demo Platform (RHDP). Guides the user through credential collection, generates an inventory for the environment, and runs the bootstrap playbook to create Hub credentials, Vault credential, project, and job template. TRIGGER when: user asks to bootstrap AAP, provision a new AAP instance, or set up a fresh RHDP environment. SKIP: if AAP is already bootstrapped and user wants to run a demo."
---

# AAP Bootstrap Skill

This skill bootstraps a fresh Ansible Automation Platform instance provisioned from the Red Hat Demo Platform (RHDP). It collects credentials, generates a named inventory, and runs `playbooks/bootstrap_dev.yml`.

## Overview

A bootstrap provisions these objects on a fresh AAP instance:
- `Automation Hub - certified` and `Automation Hub - validated` credentials (assigned to the Default Organization as its galaxy credentials)
- Vault credential
- `aap.as.code` project (synced from `main`)
- `Setup - AAP - CAC` job template

See [`skills/references/aap-as-code-context.md`](../references/aap-as-code-context.md) for the exact object names and API endpoints used.

When bootstrap completes the AAP instance is ready but no demo has been configured yet. To configure the demo, run `/aap-setup-demo`.

## Preflight Check

Before doing anything else, verify all local prerequisites are in place:

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

# user defaults
python3 -c "
import yaml, os
path = os.path.expanduser('~/.ansible/aap_defaults.yml')
try:
    d = yaml.safe_load(open(path)) or {}
    missing = [k for k in ['my_vault','my_remote_vault','my_remote_ssh_pub_key'] if not d.get(k,'')]
    if missing:
        print('❌ ~/.ansible/aap_defaults.yml missing: ' + ', '.join(missing))
    else:
        print('✅ ~/.ansible/aap_defaults.yml — user defaults present')
except FileNotFoundError:
    print('❌ ~/.ansible/aap_defaults.yml — not found. Run /aap-first-time.')
"
```

If any check fails, stop immediately and tell the user:

```
❌ Prerequisites missing: <list each failing item>

Run /aap-first-time to set up these prerequisites before bootstrapping.
```

Do not proceed to Step 1 until all checks pass.

## Step 1 — Identify the Environment

Ask the user for two values to name the inventory:
- **Customer account** — short name for the customer or account (e.g. `acme`, `redhat`, `ibm`)
- **Planned demo** — short name for the demo to run (e.g. `f5demo`, `linuxdemo`, `windowsdemo`)

The inventory will be created at:
```
inventories/rhdp-<customer>-<demo>/
```

Check if this inventory already exists in the repo. If it does, ask whether to reuse it or create a new one.

## Step 2 — Resolve Credentials

Resolve AAP credentials in this priority order:

### 2a. Environment variables (highest priority)
Check if these are already set in the shell:
```bash
echo $CONTROLLER_HOST
echo $CONTROLLER_USERNAME
echo $CONTROLLER_PASSWORD
```
If all three are non-empty, skip to Step 3.

### 2b. Local ansible.cfg
Read `~/.ansible/ansible.cfg`. If `[defaults]` contains `controller_host`, extract `controller_host`, `controller_username`, and `controller_password` from it.

### 2c. Prompt the user (fallback)
If credentials are not found via 2a or 2b, ask the user to paste:
- **AAP URL** — from the RHDP Services page (e.g. `https://aap-aap.apps.cluster-xxxx.dynamic.redhatworkshops.io`)
- **AAP Password** — from the RHDP Services page
- Username defaults to `admin` unless the user specifies otherwise

Write these to the shell session as env vars:
```bash
export CONTROLLER_HOST=<url>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=<password>
```

## Step 3 — Resolve Hub Token and Vault Password

### Hub token
Read from `~/.ansible/ansible.cfg` under `[galaxy_server.rh_certified]` → `token`. If not found, tell the user:
> "Your Automation Hub API token is missing. Get it from console.redhat.com → Automation Hub → Connect to Hub → API token, then add it to ~/.ansible/ansible.cfg under [galaxy_server.rh_certified] as: token=<your_token>"

### Vault password
Read from `~/.ansible/secrets2` (single line, trimmed). If not found, ask the user to provide the vault password and write it to `~/.ansible/secrets2`.

## Step 4 — Generate the Inventory

First, read the user-specific vars from `~/.ansible/aap_defaults.yml`:

```bash
python3 -c "
import yaml, os
d = yaml.safe_load(open(os.path.expanduser('~/.ansible/aap_defaults.yml'))) or {}
for k in ['my_vault','my_windows_catalog_short_description','my_remote_vault','my_remote_ssh_pub_key']:
    print(k + '=' + str(d.get(k,'')))
"
```

Use those values when writing the generated `all.yml` below.

Create the inventory directory and vars file:

```
inventories/rhdp-<customer>-<demo>/
  hosts                    ← empty hosts file (localhost only)
  group_vars/
    all.yml                ← environment variables (no secrets)
```

**`hosts` file:**
```ini
[all]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
```

**`group_vars/all.yml`:**
```yaml
---
# RHDP environment: rhdp-<customer>-<demo>
# Generated by /aap-bootstrap on <date>
aap_hostname: "{{ lookup('ansible.builtin.env', 'CONTROLLER_HOST') }}"
aap_username: "{{ lookup('ansible.builtin.env', 'CONTROLLER_USERNAME') }}"
aap_password: "{{ lookup('ansible.builtin.env', 'CONTROLLER_PASSWORD') }}"
aap_validate_certs: false
hub_token: "{{ lookup('ini', 'token section=galaxy_server.rh_certified file=~/.ansible/ansible.cfg') }}"
vault_password: "{{ lookup('ansible.builtin.file', '~/.ansible/secrets2') | trim }}"
hub_auth_url: "https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token"
my_vault: "<my_vault from rhdp-sample-demo all.yml>"
my_windows_catalog_short_description: "<my_windows_catalog_short_description from rhdp-sample-demo all.yml>"
my_remote_vault: "<my_remote_vault from rhdp-sample-demo all.yml>"
my_remote_ssh_pub_key: "<my_remote_ssh_pub_key from rhdp-sample-demo all.yml>"
```

Substitute the actual values read from `rhdp-sample-demo/group_vars/all.yml` — do not copy the placeholder text literally.

Remind the user that `group_vars/all.yml` contains no secrets — all sensitive values are resolved at runtime via lookups.

## Step 5 — Run the Bootstrap

Run the playbook against the generated inventory:

```bash
ansible-playbook -i inventories/rhdp-<customer>-<demo>/ playbooks/bootstrap_dev.yml
```

Show the command to the user before running it and ask for confirmation.

### Interpreting playbook output

The bootstrap modules use `state: present` and are fully idempotent. Interpret the output as follows:

| Result | Meaning | Action |
|--------|---------|--------|
| `ok` | Object already exists and is correct | Continue — this is not a warning |
| `changed` | Object was created or updated | Continue |
| `failed` | Real error | Stop and report the error |

If the playbook was previously interrupted and is being re-run, it will skip objects that already exist (`ok`) and only create what is missing (`changed`). This is expected and correct — do not treat a mix of `ok` and `changed` as a problem.

## Step 6 — Report Results

### On success

Query the AAP API to verify each object was actually created, then print a structured summary.

Create a session token for the verification queries:
```bash
TOKEN_JSON=$(curl -s -k -X POST \
  -H "Content-Type: application/json" \
  -u $CONTROLLER_USERNAME:$CONTROLLER_PASSWORD \
  "$CONTROLLER_HOST/api/gateway/v1/tokens/" \
  -d '{"description":"bootstrap-verify token","scope":"write"}')

TOKEN=$(echo $TOKEN_JSON | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
TOKEN_ID=$(echo $TOKEN_JSON | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
```

Verify each object and print status:
```bash
# Automation Hub - certified
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/credentials/?name=Automation+Hub+-+certified" \
  | python3 -c "import sys,json; c=json.load(sys.stdin)['count']; print('✅ Automation Hub - certified' if c>0 else '❌ Automation Hub - certified — NOT FOUND')"

# Automation Hub - validated
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/credentials/?name=Automation+Hub+-+validated" \
  | python3 -c "import sys,json; c=json.load(sys.stdin)['count']; print('✅ Automation Hub - validated' if c>0 else '❌ Automation Hub - validated — NOT FOUND')"

# Vault credential — look up the Vault credential type ID first, then query by it
VAULT_TYPE_ID=$(curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/credential_types/?name=Vault" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/credentials/?credential_type=$VAULT_TYPE_ID" \
  | python3 -c "
import sys,json
results=json.load(sys.stdin)['results']
if results:
    for r in results:
        print(f'✅ {r[\"name\"]} (vault)')
else:
    print('❌ Vault credential — NOT FOUND')"

# Project — also check sync status
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/projects/?name=aap.as.code" \
  | python3 -c "
import sys,json
results=json.load(sys.stdin)['results']
if results:
    status=results[0]['status']
    icon='✅' if status=='successful' else '⚠️'
    print(f'{icon} aap.as.code (sync: {status})')
else:
    print('❌ aap.as.code — NOT FOUND')"

# Job template
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/controller/v2/job_templates/?name=Setup+-+AAP+-+CAC" \
  | python3 -c "import sys,json; c=json.load(sys.stdin)['count']; print('✅ Setup - AAP - CAC' if c>0 else '❌ Setup - AAP - CAC — NOT FOUND')"
```

Delete the verification token:
```bash
curl -s -k -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "$CONTROLLER_HOST/api/gateway/v1/tokens/$TOKEN_ID/"
```

Print the final summary in this format:
```
Bootstrap complete. Created in AAP:

  Credentials:
    ✅ Automation Hub - certified
    ✅ Automation Hub - validated
    ✅ <vault credential name> (vault)

  Projects:
    ✅ aap.as.code (sync: successful)

  Job Templates:
    ✅ Setup - AAP - CAC

Next step: Run /aap-setup-demo to configure your full demo environment,
or launch "Setup - AAP - CAC" manually in the AAP UI.
```

If any object shows ❌, tell the user which object is missing and suggest re-running `/aap-bootstrap` to retry.

### On failure

- Show the error from the playbook output
- Suggest the most likely fix based on the error message
