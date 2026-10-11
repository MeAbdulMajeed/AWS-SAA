# IAM (Identity and Access Management) — Section Notes

Course: Ultimate AWS Certified Solutions Architect Associate (Maarek)
Tier: **1 — heavily tested, know this deeply**

---

## 1. IAM Overview

- IAM = Identity and Access Management
- **Global service** — not tied to any region. A user created in IAM is available everywhere.
- Root account: created automatically when you set up AWS. Use it **only** to initially set up the account — never for daily work, never share it.
- Instead of using root, create **IAM Users** — one user = one physical person in your organization.
- Users can be grouped into **Groups** for easier permission management.

### Groups — key rules
- Groups can only contain **users**, not other groups (no nested groups).
- A user doesn't have to belong to a group (not best practice, but allowed).
- A user **can belong to multiple groups** (e.g. Developers + Audit Team).

---

## 2. IAM Policies

- Permissions are defined via **JSON documents called Policies**, attached to users or groups.
- AWS enforces the **principle of least privilege** — don't grant more access than a user actually needs.

### Policy structure
```json
{
  "Version": "2012-10-17",
  "Id": "policy_id (optional)",
  "Statement": [
    {
      "Sid": "statement_id (optional)",
      "Effect": "Allow | Deny",
      "Principal": "account/user/role this applies to",
      "Action": ["list of API calls"],
      "Resource": ["list of resources"],
      "Condition": {} 
    }
  ]
}
```
- **Effect** — Allow or Deny.
- **Principal** — who the policy applies to.
- **Action** — which API calls are allowed/denied (supports wildcards, e.g. `Get*`, `List*`).
- **Resource** — which resources the actions apply to.
- **Condition** — optional, further restricts when the statement applies.
- `*` means "anything" — `Action: *, Resource: *` = full admin access.

### Policy attachment & inheritance
- Policies can be attached **directly to a user** (inline policy) or **to a group** (inherited by all members).
- A user's effective permissions = sum of everything directly attached + everything inherited from every group they belong to.
- **Removing a user from a group immediately removes any permissions that were only inherited through that group** — takes effect right away, not after re-login.

---

## 3. Account Security: Password Policy & MFA

### Password Policy
Configurable options:
- Minimum password length
- Require uppercase / lowercase / number / non-alphanumeric character
- Allow users to change their own password
- Password expiration (e.g. every 90 days)
- Prevent password reuse

### MFA (Multi-Factor Authentication)
- Combines **something you know** (password) + **something you own** (a device).
- Protects even if the password is compromised — attacker would also need the physical device.
- **Must protect the root account at minimum**; recommended for all IAM users too.

**MFA device options (know these for the exam):**
| Type | Notes |
|---|---|
| Virtual MFA device | Google Authenticator (1 device per app instance) or Authy (multiple tokens on one device) |
| U2F Security Key | Physical key, e.g. YubiKey (third-party, not AWS). Supports multiple root/IAM users on one key. |
| Hardware key fob | Third-party (e.g. Gemalto) |
| GovCloud hardware key fob | Special device for AWS GovCloud (US), by SurePassID |

---

## 4. Accessing AWS — 3 Ways

| Method | Protected by |
|---|---|
| **Management Console** | Username + password (+ MFA) |
| **CLI** (Command Line Interface) | Access Key ID + Secret Access Key |
| **SDK** (Software Development Kit) | Same access keys, used programmatically from within application code |

- **Access keys** are generated via the console, downloaded **once only** (secret access key cannot be retrieved again after creation).
- Access keys = secret, like a password. Never share them.
- CLI is open-source, built on the **AWS SDK for Python (Boto)**.
- SDK supports many languages: JavaScript, Python, PHP, .NET, Ruby, Java, Go, Node.js, C++, plus mobile (Android/iOS) and IoT SDKs.
- CLI permissions = exactly the same as console permissions for that same user (removing group access blocks both equally).

### CLI setup
`aws configure` prompts for:
1. Access Key ID
2. Secret Access Key
3. Default region
4. Default output format

### AWS CloudShell
- Browser-based terminal, built into the AWS Console — **free**, no local setup needed.
- Uses your currently logged-in console credentials automatically (no separate key config needed).
- **Not available in all regions** — check region availability before relying on it.
- Files created in CloudShell **persist** across sessions.
- Default region in CloudShell = whatever region you're currently viewing in the console (unlike CLI, which uses your configured default unless overridden).
- Supports upload/download of files, multiple tabs/split panes, customizable font size/theme.

---

## 5. IAM Roles

- Roles are **like users, but intended for AWS services, not people** (e.g. EC2 instances, Lambda functions, CloudFormation).
- Used whenever an AWS service needs to perform actions on your behalf and needs its own permissions to do so.
- Example: an EC2 instance needing to read/write S3 → attach an IAM Role with the right policy, rather than embedding IAM user access keys on the instance (which is a security risk).
- Creating a role: choose the **trusted entity** (e.g. "EC2") → attach policy/policies → name the role.
- A role is a single entity combining an identity + attached permissions, "assumed" by the service rather than logged into.

---

## 6. IAM Security Tools

| Tool | Level | What it shows |
|---|---|---|
| **IAM Credentials Report** | Account-level | CSV report listing every user: password status/age, MFA enabled or not, access key status/rotation |
| **IAM Access Advisor** ("Last Accessed" tab) | User-level | Which services a specific user is permitted to use, and when each was last actually accessed |

- Credentials Report → good for account-wide **credential hygiene** audits (stale passwords, missing MFA, unrotated keys).
- Access Advisor / Last Accessed → good for finding **unused permissions** on a specific user, to tighten access toward least privilege.
- Both tools directly support the least privilege principle — they turn "we hope permissions are right" into something you can actually verify.

---

## 7. IAM Best Practices

- Don't use the root account except for initial setup.
- One AWS user = one physical person. Never share credentials — create a new user instead.
- Assign users to groups; manage permissions at the group level, not per-user.
- Enforce a strong password policy.
- Enable and enforce MFA wherever possible.
- Use **Roles**, not embedded credentials, for AWS services (e.g. EC2) needing permissions.
- If using CLI/SDK, generate access keys and keep them completely private.
- Regularly audit using the Credentials Report and Access Advisor / Last Accessed.
- Never, ever share IAM users or access keys with anyone.

---

## Section Summary (quick recap)
- **Users** → physical people, console password.
- **Groups** → contain only users, used to manage permissions collectively.
- **Policies** → JSON documents defining permissions, attached to users/groups/roles.
- **Roles** → identities for AWS services, not people.
- **MFA + password policy** → account security layer.
- **CLI / SDK** → programmatic access via access keys.
- **Access keys** → used for CLI/SDK, generated once, kept secret.
- **Credentials Report** (account-wide) + **Access Advisor / Last Accessed** (per-user) → auditing tools for least privilege.