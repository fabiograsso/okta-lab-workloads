The purpose of this lab is to demonstrate how to use Okta PAM Workloads with GitHub Actions OIDC to retrieve secrets and run SSH commands against a Linux target.

## Lab Details

- **OPA Tenant:** i.e. https://xyz.pam.okta.com
- **Team:** i.e. xyz
- **Target Secret:** i.e. folder `production_server_secrets`, name `MySql_root`, key `password`
- **CLI Tool:** `sft` (ScaleFT / Okta PAM client) — installed from `dist.scaleft.com`

## Documentation

- Main documentation: [Workloads](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-workloads.htm)
  - [Requirements and limitations](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-requirements-workloads.htm)
  - [Get Started](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-configure-workloads.htm)
  - [Configure Workload Connection](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-configure-workload-connection.htm)
  - [CLI command for workload authentication](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-configure-workload-cli.htm)
  - [Configure Workload Role](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-configure-workload-role.htm)
  - [Principal SSH access for automated workloads](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-ssh-access-workloads.htm)
- [How secrets work in Okta Privileged Access](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-secrets.htm)
- [Okta Privileged Access Overview](https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-overview.htm)
 
## Authentication Flow

GitHub Actions OIDC token → `sft wl authenticate` → OPA Bearer Token → `sft secrets reveal` to retrieve secret

The workflows are split and intentionally self-contained:

- `.github/workflows/opa-workloads-read-secret.yml` reveals one configured secret and includes OIDC, `sft` install, and Workloads authentication inline.
- `.github/workflows/opa-workloads-ssh-linux.yml` runs an SSH test against a Linux target and includes OIDC, `sft` install, and Workloads authentication inline.
- `.github/workflows/opa-workloads-read-secret.sampledocker` and `.github/workflows/opa-workloads-ssh-linux.sampledocker` are non-executable examples showing the same flows with `ghcr.io/xyz/opa-scaleft-client:latest`.

## Optional AWS Security Group Auto-Update

The SSH workflow can optionally open a temporary AWS Security Group ingress rule for the current GitHub-hosted runner public IP. This is only for labs where the Okta PAM / ScaleFT gateway or target VM is protected by AWS inbound rules.

The feature is disabled by default and must remain optional. It is enabled only when both of these GitHub Actions Variables are configured:

- `AWS_GROUP_ID`
- `AWS_ROLE_TO_ASSUME`

`AWS_REGION` is optional and defaults to `us-east-1`. `AWS_SECURITY_GROUP_PORT` is optional and defaults to TCP `7234`.

Future agents may modify this logic only when changing SSH/gateway connectivity behavior. Preserve these guardrails:

- use GitHub OIDC with `aws-actions/configure-aws-credentials`; do not add static AWS access keys
- do not open `0.0.0.0/0`
- keep the rule scoped to the current runner `/32`
- always remove the rule with an `if: always()` cleanup step
- tolerate duplicate rule creation and cleanup when the rule is already gone
- preserve backward compatibility for non-AWS usage and for the secret workflow

Key `sft` commands:
```bash
# Exchange GitHub JWT for OPA token
export OPA_TOKEN=$(sft wl authenticate \
  --team <team> \
  --connection <workload-connection-name> \
  --jwt-env <env-var-with-github-jwt> \
  --role-hint <workload-role-name> \
  2>&1 | tr -d '\n\r')

# OPA_TOKEN is picked up automatically by subsequent sft commands

# 1. List all accessible secret folders
sft secrets list \
  --team   <team> \
  --output json

# 2. List secrets inside a folder (resource-group/project/id come from step 1)
sft secrets list \
  --team           <team> \
  --resource-group <resource_group.name> \
  --project        <project.name> \
  --id             <folder-id> \
  --output json

# 3. Reveal key-value pairs of a secret (resource-group/project/id from step 2)
sft secrets reveal \
  --team           <team> \
  --resource-group <resource_group.name> \
  --project        <project.name> \
  --id             <secret-id> \
  --output json
```

### Example JSON responses

`sft secrets list --team ... --output json` → folders:
```json
[{ "id": "af34dee2-...", "name": "production_server_secrets",
   "resource_group": {"name": "production_servers"},
   "project": {"name": "secrets"}, "type": "folder" }]
```

`sft secrets list --team ... --id <folder-id> --output json` → secrets:
```json
[{ "id": "fafd42e6-...", "name": "MySql_root",
   "resource_group": {"name": "production_servers"},
   "project": {"name": "secrets"}, "type": "key_value_secret" }]
```

`sft secrets reveal --team ... --id <secret-id> --output json` → key-value pairs:
```json
[{ "key_name": "username", "secret_value": "mysql" },
 { "key_name": "password", "secret_value": "password123" },
 { "key_name": "host",     "secret_value": "mydb1" }]
```

## GitHub Actions Variables Required

All configuration is done via GitHub Actions **Variables** (repo Settings → Secrets and variables → Variables tab). No hardcoded values in the workflow.

| Variable | Example value | Notes |
|----------|--------------|--------------|
| `OPA_ADDR` | `https://xyz.pam.okta.com` | OPA Tenant URL |
| `SFT_TEAM` | `xyz` | Team name |
| `OPA_WORKLOAD_CONNECTION` | `github-actions` | Workload connection name; optional default |
| `OPA_WORKLOAD_ROLE` | `workload-role-1` | Workload role name; optional default |
| `SECRET_RESOURCE_GROUP` | `production_servers` | Resource group containing the secret folder |
| `SECRET_PROJECT` | `secrets` | Project containing the secret folder; optional default |
| `SECRET_FOLDER` | `production_server_secrets` | Folder containing the target secret(s); optional default |
| `SECRET_NAME` | `MySql_root` | Target secret name; optional default |
| `OPA_LINUX_TARGET` | `opa-linux-target` | Linux server name used for the SSH test; optional default |

Secret selection behavior:

| Scope variables | `SECRET_NAME` | Behavior |
|-----------------|---------------|----------|
| defaults or overrides | defaults or override | Reveal one configured secret directly |

`OPA_WORKLOAD_CONNECTION`, `OPA_WORKLOAD_ROLE`, `SECRET_RESOURCE_GROUP`, `SECRET_PROJECT`, `SECRET_FOLDER`, `SECRET_NAME`, and `OPA_LINUX_TARGET` have lab-friendly defaults. The workflow runs an SSH test with `script -q -e -c "sft ssh '$OPA_LINUX_TARGET' --command 'hostname && uname -a && whoami'" /dev/null` to provide a pseudo-terminal in GitHub Actions, then reveals one configured secret directly via `sft secrets reveal --resource-group "$SECRET_RESOURCE_GROUP" --project "$SECRET_PROJECT" --path "$SECRET_FOLDER" --name "$SECRET_NAME" --output json`.

`sft` setup is intentionally not cached as an apt package. The shared action checks whether `sft` already exists and installs `scaleft-client-tools` only when needed. If performance becomes a blocker, prefer a self-hosted runner or custom container image with `sft` preinstalled.

This lab workflow prints decoded GitHub OIDC token claims and decoded OPA token content when available. This is only for disposable labs and must not be enabled in production.

## Future Lab Ideas

- Expand SSH tests into a richer command matrix for workload-managed Linux targets


## Sample OIDC Token from GitHub Actions
OIDC_TOKEN_HEADER:
{
  "alg": "RS256",
  "kid": "38826b17-6a30-5f9b-b169-8beb8202f723",
  "typ": "JWT",
  "x5t": "ykNaY4qM_ta4k2TgZOCEYLkcYlA"
}
OIDC_TOKEN_PAYLOAD:
{
  "actor": "xyz",
  "actor_id": "798097",
  "aud": "https://xyz.pam.okta.com",
  "base_ref": "",
  "check_run_id": "82477645421",
  "event_name": "push",
  "exp": 1781953097,
  "head_ref": "",
  "iat": 1781952797,
  "iss": "https://token.actions.githubusercontent.com",
  "job_workflow_ref": "xyz/okta-lab-workloads/.github/workflows/opa-workloads-read-secret.yml@refs/heads/main",
  "job_workflow_sha": "7929bf4465790ad6f6b9fb3a4f1195518f99b212",
  "jti": "aba5e289-a34f-4585-b2f5-e48065d7ff84",
  "nbf": 1781952497,
  "ref": "refs/heads/main",
  "ref_protected": "false",
  "ref_type": "branch",
  "repository": "xyz/okta-lab-workloads",
  "repository_id": "1274399613",
  "repository_owner": "xyz",
  "repository_owner_id": "798097",
  "repository_visibility": "private",
  "run_attempt": "1",
  "run_id": "27868968625",
  "run_number": "20",
  "runner_environment": "github-hosted",
  "sha": "7929bf4465790ad6f6b9fb3a4f1195518f99b212",
  "sub": "repo:xyz/okta-lab-workloads:ref:refs/heads/main",
  "workflow": "OPA Workloads — Read Secrets",
  "workflow_ref": "xyz/okta-lab-workloads/.github/workflows/opa-workloads-read-secret.yml@refs/heads/main",
  "workflow_sha": "7929bf4465790ad6f6b9fb3a4f1195518f99b212"
}

## Sample OPA Token from `sft wl authenticate`

OPA_TOKEN_HEADER:
{
  "alg": "EdDSA",
  "jwk_proof": "***",
  "typ": "JWT+OAT"
}
OPA_TOKEN_PAYLOAD:
{
  "aud": [
    "okta.com"
  ],
  "exp": 1781954187,
  "iat": 1781953287,
  "iss": "161a2b1e-3e80-4531-b2ee-5065a3c0384b",
  "jti": "2498d486-a781-405e-be53-8b65e5d6c498",
  "nbf": 1781953227,
  "oktapa.cid": "eae5e0c4-c365-4e07-90ba-9ee116267412",
  "oktapa.cn": "github-actions",
  "oktapa.rh": "workload-role-1",
  "oktapa.tid": "da4712b1-3aae-45ef-bbf1-1907525f3aec",
  "oktapa.tn": "xyz",
  "oktapa.wl": {
    "actor": "xyz",
    "actor_id": "798097",
    "base_ref": "",
    "check_run_id": "82478151897",
    "event_name": "push",
    "head_ref": "",
    "job_workflow_ref": "xyz/okta-lab-workloads/.github/workflows/opa-workloads-read-secret.yml@refs/heads/main",
    "job_workflow_sha": "6e2e942f38b7a4a8bc5cdcde9119399c87c29c5f",
    "ref": "refs/heads/main",
    "ref_protected": "false",
    "ref_type": "branch",
    "repository": "xyz/okta-lab-workloads",
    "repository_id": "1274399613",
    "repository_owner": "xyz",
    "repository_owner_id": "798097",
    "repository_visibility": "private",
    "run_attempt": "1",
    "run_id": "27869155209",
    "run_number": "21",
    "runner_environment": "github-hosted",
    "sha": "6e2e942f38b7a4a8bc5cdcde9119399c87c29c5f",
    "workflow": "OPA Workloads — Read Secrets",
    "workflow_ref": "xyz/okta-lab-workloads/.github/workflows/opa-workloads-read-secret.yml@refs/heads/main",
    "workflow_sha": "6e2e942f38b7a4a8bc5cdcde9119399c87c29c5f"
  },
  "sub": "opa://xyz/github-actions/repo:xyz%2Fokta-lab-workloads:ref:refs%2Fheads%2Fmain"
}

## Okta Sample Log

```json
{
  "actor": {
    "id": "opa://xyz/github-actions/repo:xyz%2Fokta-lab-workloads:ref:refs%2Fheads%2Fmain",
    "type": "WorkloadPrincipal",
    "alternateId": null,
    "displayName": "opa://xyz/github-actions/repo:xyz%2Fokta-lab-workloads:ref:refs%2Fheads%2Fmain",
    "detailEntry": null
  },
  "client": {
    "userAgent": {
      "rawUserAgent": "scaleft.go/1.108.0 (sft; k:n o:la)",
      "os": null,
      "browser": null
    },
    "zone": null,
    "device": null,
    "id": null,
    "ipAddress": "9.234.149.82",
    "geographicalContext": null
  },
  "device": null,
  "authenticationContext": null,
  "displayMessage": "(PAM) Issue credentials to access servers",
  "eventType": "pam.user_creds.issue",
  "outcome": {
    "result": "SUCCESS",
    "reason": null
  },
  "published": "2026-06-21T13:25:29.746Z",
  "securityContext": null,
  "severity": "INFO",
  "debugContext": {
    "debugData": {
      "teamName": "xyz",
      "traceId": "1-6a37e649-1bac2f0570707c8d38d1ad22",
      "x509KeyFingerprint": "77:8b:07:78:c2:5a:8c:0c:9d:4c:69:1e:04:d5:af:78",
      "servers": "af1e2e28-1a72-454c-b15d-d6cbfe9a0064",
      "userAccessMethod": "",
      "clientIp": "9.234.149.82",
      "client": "WORKLOAD",
      "mfaChallengeCompleted": "false",
      "serverHostnames": "opa-linux-target",
      "sshKeyFingerprint": ""
    }
  },
  "legacyEventType": null,
  "transaction": {
    "type": "SERVICE",
    "id": "6fea2c91-7864-4ec8-b829-85ab9b651a03",
    "detail": {}
  },
  "uuid": "ac4c47d3-6d74-11f1-b822-f5d0a6c662bd",
  "version": "0",
  "request": {
    "ipChain": []
  },
  "target": [
    {
      "id": "e2282b27-05bd-467d-b7b3-0eb0de35ac83",
      "type": "Project",
      "alternateId": "e2282b27-05bd-467d-b7b3-0eb0de35ac83",
      "displayName": "web_servers",
      "detailEntry": null
    },
    {
      "id": "da4712b1-3aae-45ef-bbf1-1907525f3aec",
      "type": "Team",
      "alternateId": "da4712b1-3aae-45ef-bbf1-1907525f3aec",
      "displayName": "xyz",
      "detailEntry": null
    }
  ]
}
```

## TODO

1. Update article with limitations and requirements, including:
https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-requirements-workloads.htm

2. Explain that at the moment (22 june 2026) only the [Principal SSH access for automated workloads])https://help.okta.com/oie/en-us/content/topics/privileged-access/pam-ssh-access-workloads.htm) is supported and working.

3. The secrets Workload is not yet supported, and the `sft secrets` commands will fail with a 403 error. The secrets workload is expected to be supported in the future.
Errors
`OPA_TOKEN is not valid: Missing capability: secret_folder.item_list`
and
`OPA_TOKEN is not valid: Missing capability: secret.resolve`

explain that once it will be supported it should work :)
