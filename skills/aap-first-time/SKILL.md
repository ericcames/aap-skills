---
name: aap-first-time
description: >
  First-time setup for AAP skills. Run this before aap-bootstrap or aap-setup-demo.
  Use when the user is new to these skills, asks how to get started, says prerequisites
  are missing, or hits errors about ansible.cfg or secrets2 not found. Walks through
  every prerequisite interactively and validates each one before continuing.
---

# AAP First-Time Setup Skill

This skill walks a new user through every local prerequisite needed to run `/aap-bootstrap` or `/aap-setup-demo`. It checks what already exists, guides creation of anything missing, and ends with a full preflight validation.

Run this once on a new machine. If you've already done this setup, skip straight to `/aap-bootstrap`.

## Orientation

Print this once at the start — do not repeat on subsequent steps:

```
You're setting up the prerequisites to bootstrap an AAP demo environment
from your laptop using Claude Code. This takes about 10 minutes once.

We'll set up:
  1. Your Automation Hub API token (~/.ansible/ansible.cfg)
  2. Your vault password file (~/.ansible/secrets2)
  3. Required Ansible collections (ansible.platform, ansible.controller)
  4. Your SSH key pair (local key + public key hosted at a URL)
  5. Your vault file (secrets the demo uses at runtime)

After this, you'll be ready to run /aap-bootstrap or /aap-setup-demo.
```

Before proceeding, verify the user is running from inside the `aap.as.code` repo:

```bash
test -f playbooks/bootstrap_dev.yml && echo "✅ In aap.as.code directory" || echo "❌ Wrong directory"
```

If the check fails, stop and tell the user:

```
❌ You must run this skill from inside the aap.as.code repo.

Clone it first:
  git clone https://github.com/ericcames/aap.as.code.git
  cd aap.as.code
  claude .

Then re-run /aap-first-time.
```

## Step 1 — Check What Already Exists

Before doing anything, audit current state:

```bash
# ansible.cfg
test -f ~/.ansible/ansible.cfg && echo "EXISTS" || echo "MISSING"

# secrets2
test -f ~/.ansible/secrets2 && echo "EXISTS" || echo "MISSING"

# collections
ls ./collections/ansible_collections/ansible/ 2>/dev/null || echo "MISSING"

# SSH key
test -f ~/.ssh/id_rsa && echo "EXISTS" || echo "MISSING"
```

For each item found, validate it has the right content:
- **ansible.cfg**: must contain `[galaxy_server.rh_certified]` and a non-placeholder `token=` line
- **secrets2**: must be non-empty
- **collections**: must include both `platform` and `controller` subdirectories
- **SSH key**: must be RSA format

Skip any section below where the item already exists and is valid. Tell the user what was found before proceeding:

```
✅ ~/.ansible/ansible.cfg — found with Hub token
✅ ~/.ansible/secrets2 — found, non-empty
❌ ansible.platform collection — MISSING
❌ ~/.ssh/id_rsa — MISSING
```

## Step 2 — ansible.cfg Setup

Skip this step if ansible.cfg already exists and contains a valid Hub token.

If missing, show the user exactly what it should look like and ask them to create it:

```ini
[defaults]
collections_paths = ./collections

[galaxy_server.rh_certified]
url=https://console.redhat.com/api/automation-hub/content/published/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=PASTE_YOUR_TOKEN_HERE

[galaxy_server.rh_validated]
url=https://console.redhat.com/api/automation-hub/content/validated/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=PASTE_YOUR_TOKEN_HERE
```

Then prompt:
> "Paste your Automation Hub API token. Get it at: https://console.redhat.com/ansible/automation-hub/token"

Write the token into both `[galaxy_server.rh_certified]` and `[galaxy_server.rh_validated]` token fields. Validate the written value is not the placeholder string `PASTE_YOUR_TOKEN_HERE`.

If ansible.cfg exists but the token is missing or placeholder, ask the user to paste their token and update it in place.

## Step 3 — secrets2 Setup

Skip this step if `~/.ansible/secrets2` exists and is non-empty.

Explain what this file is:

```
~/.ansible/secrets2 should contain one line: your vault password.
This is the password you chose when encrypting your vault file.
It must match whatever password was used on the vault file you're using.
```

Create the file with the correct permissions:

```bash
mkdir -p ~/.ansible
touch ~/.ansible/secrets2
chmod 600 ~/.ansible/secrets2
```

Then prompt the user for their vault password and write it as a single line to `~/.ansible/secrets2`.

Confirm the file is non-empty after writing.

## Step 4 — Collections Install

Skip this step if both `./collections/ansible_collections/ansible/platform` and `./collections/ansible_collections/ansible/controller` exist.

Install the required collections:

```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg \
  ansible-galaxy collection install \
    ansible.platform \
    ansible.controller \
  -p ./collections
```

Show this command to the user and ask for confirmation before running it.

After running, validate:

```bash
test -d ./collections/ansible_collections/ansible/platform && \
  echo "✅ ansible.platform — installed" || echo "❌ ansible.platform — MISSING"

test -d ./collections/ansible_collections/ansible/controller && \
  echo "✅ ansible.controller — installed" || echo "❌ ansible.controller — MISSING"
```

If either is missing after install, report the error and stop — do not proceed.

## Step 5 — SSH Key Check

The demo provisions cloud infrastructure (AWS). Your SSH key gives you access to what gets built. The private key goes into your vault file; the public key must be hosted at a public HTTPS raw URL so AWS can install it on provisioned nodes.

### 5a. Check for a local RSA key

```bash
test -f ~/.ssh/id_rsa && echo "EXISTS" || echo "MISSING"
```

If missing, offer to generate one:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
```

Show this command and ask for confirmation before running. Note that Amazon Web Services requires RSA keys.

If the user has an RSA key at a different path, ask them to provide it and use that path throughout this step.

### 5b. Verify the key is RSA

```bash
ssh-keygen -l -f ~/.ssh/id_rsa
```

Confirm the output shows `(RSA)`. If the key is a different type (ed25519, ecdsa), warn the user that AWS requires RSA and offer to generate a new RSA key alongside it.

### 5c. Check for a hosted public key URL

Ask:
> "Is your public SSH key hosted at a public HTTPS raw URL? (e.g. https://raw.githubusercontent.com/youruser/yourrepo/main/id_rsa.pub)"

If yes: ask for the URL and validate it in 5d.

If no: direct the user to host it:
> "Add your public key to a public GitHub repo and use the raw URL. Example:
> https://raw.githubusercontent.com/youruser/sourcefiles/main/id_rsa.pub
> The URL must use raw.githubusercontent.com — not github.com/blob/ or github.com/raw/"

### 5d. Validate the hosted URL

```bash
curl -s "<hosted_pub_key_url>" | head -1
```

Confirm the response starts with `ssh-rsa` — not HTML, not a redirect page.

Valid vs invalid URL patterns:
```
✅ https://raw.githubusercontent.com/user/repo/main/id_rsa.pub
❌ https://github.com/user/repo/blob/main/id_rsa.pub   (returns HTML)
❌ https://github.com/user/repo/raw/main/id_rsa.pub    (redirects, may fail)
```

### 5e. Verify the key pair matches

Derive the public key from the private key and compare it against the hosted public key:

```bash
# Derive public key from private key
DERIVED=$(ssh-keygen -y -f ~/.ssh/id_rsa 2>/dev/null)

# Fetch hosted public key (key material only — first two fields, no comment)
HOSTED=$(curl -s "<hosted_pub_key_url>" | awk '{print $1, $2}')

# Compare (strip comment from derived too)
DERIVED_MATERIAL=$(echo "$DERIVED" | awk '{print $1, $2}')

if [ "$DERIVED_MATERIAL" = "$HOSTED" ]; then
  echo "✅ Key pair matches — hosted public key matches local private key"
else
  echo "❌ Key pair mismatch — hosted public key does not match ~/.ssh/id_rsa"
fi
```

If `ssh-keygen -y` fails (the private key has a passphrase), it will exit non-zero. In that case, fall back to comparing the local public key against the hosted key:

```bash
LOCAL_PUB=$(awk '{print $1, $2}' ~/.ssh/id_rsa.pub 2>/dev/null)
HOSTED=$(curl -s "<hosted_pub_key_url>" | awk '{print $1, $2}')

if [ "$LOCAL_PUB" = "$HOSTED" ]; then
  echo "✅ Hosted public key matches ~/.ssh/id_rsa.pub"
  echo "   (Private key has a passphrase — full key pair match could not be verified)"
else
  echo "❌ Key mismatch — hosted public key does not match ~/.ssh/id_rsa.pub"
fi
```

If there is a mismatch, stop and ask the user to either update the hosted URL or regenerate the key pair before continuing.

## Step 6 — Vault File Check

The vault file holds the secrets the demo uses at runtime (credentials, tokens, passwords). It must be hosted at a public HTTPS raw URL.

Ask:
> "Do you have a vault file hosted at a public HTTPS URL?"

### If yes:

Ask for the URL, then validate it returns YAML:

```bash
curl -s -L "<vault_url>" | head -5
```

Check that the response:
- Is not HTML (no `<html>` tag)
- Returns HTTP 200 (not 404 or redirect loop)
- Contains YAML content (`key: value` pairs or starts with `---`)

Valid vs invalid URL patterns:
```
✅ https://raw.githubusercontent.com/user/repo/main/vault.yml
❌ https://github.com/user/repo/blob/main/vault.yml   (returns HTML)
❌ https://github.com/user/repo/raw/main/vault.yml    (redirects, may fail)
```

### If no:

Direct the user to create one:
> "Copy the example vault from https://github.com/ericcames/sourcefiles/blob/main/vault_example.yml,
> fill in your values, encrypt it with ansible-vault, and host it in a public GitHub repo at a raw URL."

Full vault variable reference (keys, purposes, and example file link) is in [`skills/references/aap-as-code-context.md`](../references/aap-as-code-context.md#vault-variable-reference).

Once they have a URL, validate it as described above.

## Step 7 — Final Validation

Run a complete preflight check and print green/red status for every item:

```bash
# ansible.cfg
grep -q "galaxy_server.rh_certified" ~/.ansible/ansible.cfg 2>/dev/null && \
  grep -v "PASTE_YOUR_TOKEN_HERE" ~/.ansible/ansible.cfg | grep -q "token=" && \
  echo "✅ ~/.ansible/ansible.cfg — found with Hub token" || \
  echo "❌ ~/.ansible/ansible.cfg — missing or Hub token not set"

# secrets2
test -s ~/.ansible/secrets2 && \
  echo "✅ ~/.ansible/secrets2 — found, non-empty" || \
  echo "❌ ~/.ansible/secrets2 — missing or empty"

# collections
test -d ./collections/ansible_collections/ansible/platform && \
  echo "✅ ansible.platform collection — installed" || \
  echo "❌ ansible.platform collection — MISSING"

test -d ./collections/ansible_collections/ansible/controller && \
  echo "✅ ansible.controller collection — installed" || \
  echo "❌ ansible.controller collection — MISSING"

# SSH key
test -f ~/.ssh/id_rsa && \
  echo "✅ ~/.ssh/id_rsa — found" || \
  echo "❌ ~/.ssh/id_rsa — MISSING"
```

Also confirm the results from Steps 5 and 6:
```
✅ SSH key pair — local key matches hosted public key
✅ Vault URL — accessible, returns YAML
```

If everything is green, print:
```
All prerequisites are in place.
Ready to run /aap-bootstrap.
```

If anything is red, tell the user which step to re-run before proceeding. Do not proceed to `/aap-bootstrap` until all items are green.
