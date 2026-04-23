# aap-as-code Context Reference

Shared reference for the `aap-skills` plugin. Skills cite this file rather than duplicating the information.

Source repo: [github.com/ericcames/aap.as.code](https://github.com/ericcames/aap.as.code)

---

## AAP Object Names

These are the exact names the skills create or depend on in AAP:

| Object | Type | Name |
|--------|------|------|
| Hub certified credential | Ansible Galaxy/Automation Hub API Token | `Automation Hub - certified` |
| Hub validated credential | Ansible Galaxy/Automation Hub API Token | `Automation Hub - validated` |
| Vault credential | Vault | varies by user — query by type |
| Project | Git | `aap.as.code` |
| Job template | Job Template | `Setup - AAP - CAC` |

---

## AAP API Endpoints (2.5+)

AAP 2.5+ uses a gateway architecture. Use these base paths:

| Purpose | Endpoint |
|---------|----------|
| Token create/delete | `$CONTROLLER_HOST/api/gateway/v1/tokens/` |
| Credentials | `$CONTROLLER_HOST/api/controller/v2/credentials/` |
| Credential types | `$CONTROLLER_HOST/api/controller/v2/credential_types/` |
| Projects | `$CONTROLLER_HOST/api/controller/v2/projects/` |
| Job templates | `$CONTROLLER_HOST/api/controller/v2/job_templates/` |
| Jobs | `$CONTROLLER_HOST/api/controller/v2/jobs/` |
| Job stdout | `$CONTROLLER_HOST/api/controller/v2/jobs/<id>/stdout/?format=txt` |

To verify the API layout on an unknown instance:
```bash
curl -s -k "$CONTROLLER_HOST/api/" | python3 -m json.tool
```

---

## Vault URL Patterns

The vault file must be hosted at a **public HTTPS raw URL**. Valid vs invalid patterns:

```
✅ https://raw.githubusercontent.com/user/repo/main/vault.yml
❌ https://github.com/user/repo/blob/main/vault.yml   (returns HTML)
❌ https://github.com/user/repo/raw/main/vault.yml    (redirects, may fail)
```

The same rules apply to the hosted SSH public key:

```
✅ https://raw.githubusercontent.com/user/repo/main/id_rsa.pub
❌ https://github.com/user/repo/blob/main/id_rsa.pub  (returns HTML)
```

---

## Vault Variable Reference

Full list of variables expected in the remote vault file. Source: [aap.as.code README](https://github.com/ericcames/aap.as.code#readme).

| Key | Description | Purpose |
|-----|-------------|---------|
| `default_passwd` | Team default password; also used in the F5 demo | Default password |
| `redhat_username` | RH Service Account username | Used to register the system |
| `redhat_passwd` | RH Service Account password | Used to register the system |
| `ddw_username` | Windows server username | Used to grant access to Windows servers |
| `ddw_password` | Windows server password | Used to grant access to Windows servers |
| `snow_url` | ServiceNow instance URL | Used for SNOW automation |
| `snow_username` | ServiceNow username | Used for SNOW automation |
| `servicenow_passwd` | ServiceNow password | Used for SNOW automation |
| `ssh_priv_key` | Laptop private key | Gives access to provisioned infrastructure |
| `red_hat_auto_hub_token` | Automation Hub API token | Used to pull collections |
| `admin_password` | Network gear admin password | Used for F5, Palo Alto, Infoblox |
| `reg_rh_io_username` | registry.redhat.io username | Used to access Red Hat images |
| `reg_rh_io_passwd` | registry.redhat.io password | Used to access Red Hat images |
| `quay_username` | quay.io username | Used to access Quay images |
| `quay_passwd` | quay.io password | Used to access Quay images |
| `customer_portal_username` | Red Hat Customer Portal username | Access to software downloads |
| `customer_portal_password` | Red Hat Customer Portal password | Access to software downloads |
| `rh_activation_key` | Red Hat activation key | Used by the rhsm role |
| `rh_org_id` | Red Hat Org ID | Used by the rhsm role |
| `dyna_key` | Dyna key | Usage TBD |
| `vault_passwd` | Vault password | Usage TBD |
| `my_ctrl_username` | Controller username | Usage TBD |
| `my_ctrl_admin_password` | Controller admin password | Usage TBD |

Example vault file: [vault_example.yml](https://github.com/ericcames/sourcefiles/blob/main/vault_example.yml)
