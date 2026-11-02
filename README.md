# Google Workspace Administration & Auditing — Python

Python utilities for **read-only Google Workspace reporting and access reviews** using the Google Admin SDK Directory API and Google Drive API. Developed to reduce repetitive administrative work and produce structured CSV reports for account inventories, Shared Drive permission reviews, and security investigations.

## Highlights

- **User inventory:** Report user identities, aliases, organizational units, account suspension status, and last-login timestamps.
- **Shared Drive access reviews:** Inventory drive-level permission entries, assigned roles, and potentially external access.
- **OAuth 2.0 authentication:** Browser-based sign-in, cached authorization, and token refresh.
- **Large-environment support:** Pagination across API results.
- **CSV reporting:** Timestamped, UTF-8-with-BOM CSV exports saved next to the scripts.
- **Read-only operations:** Neither script changes users, drives, or permissions.

## Scripts

| Script | Google API | Purpose |
|---|---|---|
| [`get_workspace_users.py`](get_workspace_users.py) | Admin SDK Directory API | Inventory Google Workspace user accounts and account status. |
| [`audit_shared_drive_access.py`](audit_shared_drive_access.py) | Drive API v3 | Review Shared Drive-level permissions, roles, and external permission entries. |

## Script documentation

<details>
<summary><strong>get_workspace_users.py</strong> — User Account Inventory</summary>

### Purpose

Retrieve user accounts visible to the authenticated administrator, including identity, organizational unit, and account activity information. Useful for account inventories, onboarding/offboarding reviews, and user lifecycle audits.

**Input:** None; retrieves accounts for `customer="my_customer"` using the administrator view.

**API and OAuth scope:**

- API: Google Admin SDK Directory API (`directory_v1`)
- Scope: `https://www.googleapis.com/auth/admin.directory.user.readonly`
- Retrieves up to **500 users per page**, following `nextPageToken` until finished.

**Reported CSV columns:**

| Column | Description |
|---|---|
| `FullName` | User's full display name |
| `PrimaryEmail` | Primary email address |
| `Aliases` | Standard and non-editable/domain-alias addresses, deduplicated and separated by semicolons |
| `OrganizationalUnit` | Organizational unit path |
| `Suspended` | Whether the account is suspended (`True` / `False`) |
| `LastLogin` | Last-login timestamp returned by Google, or `Not available` |

**Console output:** Prints the total number of users and detailed records for each user.

**CSV output:** `WorkspaceUsers_YYYYMMDD_HHMMSS.csv`, saved in the script's directory.

**Authentication files:** Reads `credentials.json` and creates/reuses `token.json`.

**Limitations:** Last-login data may be absent; `Not available` is not proof that an account never signed in. This is an account inventory, not a complete authentication event log.

</details>

<details>
<summary><strong>audit_shared_drive_access.py</strong> — Shared Drive Permission Audit</summary>

### Purpose

Enumerate Shared Drives visible through domain-admin access and review **drive-level permission entries** for members, permission roles, and possible external access. Useful for access reviews and identifying entries that require follow-up investigation.

**Input:** None. Before running, configure the `INTERNAL_DOMAINS` constant in the script for your organization.

**API and OAuth scope:**

- API: Google Drive API v3
- Scope: `https://www.googleapis.com/auth/drive.readonly`
- Uses `useDomainAdminAccess=True` to list Shared Drives and their permissions; the authenticated account must be authorized for these calls.
- Retrieves up to **100 drives/permissions per page**, following pagination tokens.

**Reported CSV columns:**

| Column | Description |
|---|---|
| `DriveName` | Shared Drive name |
| `DriveId` | Shared Drive identifier |
| `MemberName` | Permission entry display name, when provided by the API |
| `Email` | Member or group email address, when available |
| `MemberType` | Google permission type, such as `user`, `group`, `domain`, or `anyone` |
| `Domain` | Domain determined from the permission entry |
| `PermissionLevel` | Human-readable role (for example, Manager, Content Manager, Contributor, Commenter, Viewer) |
| `ApiRole` | Original permission role returned by Google |
| `ExternalAccess` | `Yes`, `No`, or `Unknown` classification |
| `Deleted` | Deleted flag returned by Google; defaults to `False` if absent |
| `Status` | `OK`, `Error`, or `No permissions returned` |
| `Notes` | Review notes, including group-membership caveats and API errors |

**External-access classification:**

- User or group email domains **not** listed in `INTERNAL_DOMAINS` are labeled `Yes`.
- Listed internal domains are labeled `No`.
- `anyone` permissions are labeled `Yes`.
- Domain permission entries are classified using their domain.
- Entries lacking enough information are labeled `Unknown`.
- Group entries are annotated because the script **does not expand group members**. An internally named group may still contain external people.

**Console output:** Displays per-drive audit progress and totals for Shared Drives, permission entries, external entries (excluding deleted), groups requiring review (excluding deleted), and drives whose permission checks failed.

**CSV output:** `SharedDriveAccess_YYYYMMDD_HHMMSS.csv`, saved in the script's directory.

**Authentication files:** Reads `credentials.json` and creates/reuses `token_drive.json`.

**Package handling:** Checks for required Google Python packages and attempts to install missing ones using `pip`.

**Scope and limitations:**

- Audits **Shared Drive-level** permissions only; does not scan individual files or folders.
- Does **not** expand Google Group memberships.
- `ExternalAccess` is a classification of an individual permission entry, **not** a verification of the drive's sharing-policy setting or an exhaustive determination of who can access every file.
- API failures are written as `Status=Error` rows; check the failed-drive total before treating the report as complete.

</details>

## Requirements and setup

1. Install Python 3 and `pip`.
2. Use a Google Workspace account authorized to access the relevant Admin SDK or Drive API data. Domain-admin access is required for the Shared Drive API calls used by this script.
3. In a Google Cloud project, enable the **Admin SDK API** and/or **Google Drive API**, as appropriate.
4. Configure an **OAuth client for a desktop application**, and download its client-secret JSON file to the repository directory as `credentials.json` (the precise downloaded filename may differ).
5. Install dependencies:

   ```bash
   python -m pip install google-api-python-client google-auth google-auth-httplib2 google-auth-oauthlib
   ```

   The Shared Drive script attempts to install missing packages itself. The Workspace Users script does **not**, so installing dependencies up front is recommended.

6. For the Shared Drive audit, edit the `INTERNAL_DOMAINS` set in `audit_shared_drive_access.py` to include **all** of your organization's internal email domains. For example:

   ```python
   INTERNAL_DOMAINS = {"example.com", "subsidiary.example.com"}
   ```

7. Run the desired script from the repository directory:

   ```bash
   python get_workspace_users.py
   python audit_shared_drive_access.py
   ```

On the first run, the script opens a browser for OAuth consent. Subsequent runs can reuse the locally saved token, and expired access tokens are refreshed when possible. You do **not** need to download `token.json` or `token_drive.json` from Google; the scripts create them after successful authorization.

### Authentication files

| File | Purpose | Created by |
|---|---|---|
| `credentials.json` | OAuth desktop application's client configuration | Downloaded from Google Cloud |
| `token.json` | Cached authorization for the Workspace Users script | `get_workspace_users.py` |
| `token_drive.json` | Cached authorization for the Shared Drive script | `audit_shared_drive_access.py` |

## Reporting and security

Reports are saved in the same folder as the Python script and named with the local date and time, for example:

```text
WorkspaceUsers_20261102_123000.csv
SharedDriveAccess_20261102_123500.csv
```

CSV files use **UTF-8 with BOM**, which can help Excel display text correctly. The Shared Drive report retains original API role values alongside readable labels for auditing.

**Protect authorization files and reports.** Never commit OAuth client secrets, cached tokens, or CSV exports containing organizational user or permission data. Add the following rules to `.gitignore`:

```gitignore
credentials.json
token.json
token_drive.json
*.csv
__pycache__/
.venv/
```

These rules protect against *new* additions; if secrets or reports were committed previously, remove them from Git tracking and rotate exposed credentials or revoke affected tokens as appropriate.

## Known limitations

- Access and visibility depend on Google Workspace permissions, API access, and authorization granted to the signed-in account.
- The user inventory does not retrieve a complete sign-in history.
- The Shared Drive audit does not include file/folder permissions or nested group-member expansion.
- External-access classifications depend on the configured list of internal domains and the permission entries returned by Google.
- Both scripts are **read-only** and generate reports; neither remediates permissions or account settings.

## About

This repository is part of my IT administration and automation portfolio. These Python scripts demonstrate practical use of Google Workspace APIs, OAuth 2.0, paginated data retrieval, account inventory, access auditing, error handling, and CSV-based administrative reporting.

**Technologies:** Python · Google Workspace · Admin SDK Directory API · Google Drive API · OAuth 2.0
