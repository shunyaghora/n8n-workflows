# n8n Workflow Version Control

This repository provides Git-based version control and backup for n8n workflows running on the `rudraghora` server.

The purpose of this setup is to maintain a version-controlled backup of n8n workflows while preventing API keys, tokens, passwords, and other embedded credentials from being committed to GitHub.

## Architecture

```text
                    n8n
                     │
                     │ export workflows
                     ▼
             Temporary export
                     │
                     │ sanitize
                     ▼
          Credential / Secret Redaction
                     │
                     │ secret scan
                     ▼
              Git working tree
                     │
                     │ git commit
                     ▼
                 Git history
                     │
                     │ git push
                     ▼
                  GitHub
```

The live n8n workflows are never modified by the backup process. The n8n export is copied to a temporary directory first. Credentials are redacted there. Only the sanitized copy is placed into the Git repository.

## Repository

GitHub repository:

```text
https://github.com/shunyaghora/n8n-workflows
```

Local repository:

```text
/home/sanjay/n8n-git
```

Workflow directory:

```text
/home/sanjay/n8n-git/workflows
```

Each n8n workflow is stored as a separate JSON file.

## n8n Installation

n8n is running in Docker.

Container:

```text
n8n-n8n-1
```

n8n data volume:

```text
n8n_n8n-data
```

Docker Compose configuration:

```text
/home/sanjay/n8n/compose.yml
```

## Backup Script

The backup script is:

```text
/usr/local/bin/n8n-git-backup
```

Run it with:

```bash
sudo /usr/local/bin/n8n-git-backup
```

The script:

1. Creates a temporary export directory.
2. Exports all n8n workflows.
3. Copies the export out of the n8n container.
4. Sanitizes credentials and secrets.
5. Performs a structural secret scan.
6. Only after the scan succeeds, updates the Git workflow directory.
7. Creates a Git commit if changes exist.
8. Pushes the sanitized workflows to GitHub.

## Workflow Export

The script uses:

```bash
n8n export:workflow --all --separate
```

Temporary locations:

```text
/tmp/n8n-workflows
/tmp/n8n-git-export
```

The temporary export is removed after the sanitized workflows are copied into Git.

## Credential and Secret Redaction

Credential values are never intentionally copied into GitHub.

For example:

```json
{
  "name": "Authorization",
  "value": "ACTUAL_SECRET"
}
```

becomes:

```json
{
  "name": "Authorization",
  "value": "[REDACTED-AUTHORIZATION]"
}
```

The live n8n workflow is not modified.

### Authorization headers

Any n8n HTTP header object whose name is `Authorization` has its value replaced with:

```text
[REDACTED-AUTHORIZATION]
```

### Google / Gemini API keys

Google API keys matching the common `AIza...` format are redacted as:

```text
[REDACTED-GOOGLE-API-KEY]
```

### GitHub tokens

Recognized GitHub token formats are redacted as:

```text
[REDACTED-GITHUB-TOKEN]
```

### AWS access keys

Recognized AWS access keys are redacted as:

```text
[REDACTED-AWS-ACCESS-KEY]
```

### JWTs

Recognized JWT-style values are redacted as:

```text
[REDACTED-JWT]
```

## Secret-Like Fields

The sanitizer also examines common credential field names, including:

```text
apiKey
api_key
api-key
accessToken
access_token
auth_token
authToken
bearer_token
clientSecret
client_secret
secretKey
secret_key
privateKey
private_key
password
passwd
```

Credential-like values are replaced with:

```text
[REDACTED-SECRET]
```

## Structural Secret Scan

After sanitization, the script performs another scan before modifying the Git working tree.

The scan checks for remaining Authorization values and recognizable API-key/token formats.

If a recognizable secret remains, the backup stops and does not:

- copy the export into the Git workflow directory
- create a Git commit
- push to GitHub

This is intentional.

## Security Flow

```text
n8n
 │
 ▼
Temporary export
 │
 ▼
Sanitization
 │
 ▼
Secret scan
 │
 ├── Secret found → ABORT
 │
 └── Clean
       │
       ▼
    Git working tree
       │
       ▼
     Commit
       │
       ▼
      Push
```

The Git working tree is not updated until after sanitization and the secret scan succeed.

## Live Credentials vs Git Credentials

The live n8n workflow can continue using its real credentials.

The GitHub copy contains sanitized values such as:

```text
Authorization: [REDACTED-AUTHORIZATION]
```

Therefore, the GitHub repository is a workflow/version-control backup, not a complete credential backup.

Actual credentials remain managed by n8n.

## Git Configuration

Local repository:

```text
/home/sanjay/n8n-git
```

Remote:

```text
git@github.com:shunyaghora/n8n-workflows.git
```

Branch:

```text
main
```

Git identity:

```text
Sanjay Asrani
shunyaghora@gmail.com
```

SSH key:

```text
/home/sanjay/.ssh/id_ed25519
```

Test GitHub SSH authentication:

```bash
sudo -u sanjay ssh -T git@github.com
```

## Running a Backup

```bash
sudo /usr/local/bin/n8n-git-backup
```

A successful run should contain messages similar to:

```text
=== Exporting n8n workflows ===
Successfully exported 23 workflows.

=== Sanitizing exported workflows ===
Redacted <N> credential/secret value(s).

=== Final structural secret scan ===
Secret scan passed.

=== Updating Git working tree ===
=== Committing ===
=== Pushing ===
=== Backup complete ===
```

The number of workflows and redactions may change over time.

## No Changes

If workflows have not changed:

```text
No workflow changes detected.
```

No unnecessary commit is created.

## Failed Secret Scan

If a secret is detected that cannot be safely sanitized:

```text
BACKUP ABORTED
A recognizable secret remains.
Nothing was copied to Git.
Nothing was committed or pushed.
```

Do not bypass the protection simply to make a backup succeed. Investigate the new credential format and update the sanitizer if appropriate.

## Checking Repository Status

```bash
cd /home/sanjay/n8n-git
git status
```

A clean repository should report:

```text
nothing to commit, working tree clean
```

View recent history:

```bash
git log --oneline --decorate -5
```

Check the remote:

```bash
git fetch origin
git status
```

## Checking Workflow Files

List workflows:

```bash
ls -lh /home/sanjay/n8n-git/workflows
```

Count workflows:

```bash
find /home/sanjay/n8n-git/workflows -name '*.json' | wc -l
```

## GitHub Is Not a Credential Store

Do not intentionally store live credentials in GitHub.

The purpose of this repository is to version-control workflow definitions, not secrets.

Credentials should remain in n8n credential storage, environment variables, a secrets manager, or another dedicated credential-management system.

## Credential Rotation

If a credential has accidentally been committed, assume it may have been exposed.

Recommended procedure:

1. Revoke or rotate the credential with the provider.
2. Create the replacement credential.
3. Update the live n8n credential.
4. Test the affected workflow.
5. Remove the credential from Git history if necessary.
6. Update the sanitizer so the same credential format cannot be committed again.

Deleting a secret from only the latest commit is not sufficient if it existed in earlier Git history.

## Previous Credential Incident

This repository previously contained exposed credentials.

The affected provider credentials were rotated or replaced.

The old Git history was subsequently replaced with a clean root history.

The previous secret-containing commit:

```text
9578ebe
```

is no longer reachable from `main`.

The clean repository root is:

```text
bdcd75f
```

Subsequent workflow backup commits are based on that clean history.

## Git History Cleanup

If sensitive information is ever committed accidentally, do not simply delete the file and create another commit. The secret may remain in Git history.

The appropriate process is:

```text
1. Revoke/rotate the credential
2. Remove the secret from repository history
3. Force-update the remote branch
4. Verify the old commit is unreachable
5. Improve the backup sanitizer
```

The previous cleanup used an orphan root commit to create a clean repository history.

The remote can be checked with:

```bash
git merge-base --is-ancestor 9578ebe origin/main
```

The expected result is that the old commit is not reachable.

## GitHub Push Protection

GitHub Push Protection may detect secrets even if the local scanner does not.

If GitHub rejects a push because of a detected credential:

1. Identify the detected secret.
2. Determine where it exists in the workflow.
3. Update the sanitizer.
4. Regenerate the sanitized Git copy.
5. Verify the secret is gone.
6. Push again.

Do not automatically bypass GitHub Push Protection.

## Restoring a Workflow

A workflow can be restored from its JSON file:

```text
workflows/<workflow-id>.json
```

The Git version intentionally contains redacted credentials.

After importing a sanitized workflow into n8n, credential references may need to be reconnected or configured.

The repository should not be expected to restore production credential values.

## Local Git Workflow

```text
Edit workflow in n8n
        │
        ▼
Run backup script
        │
        ▼
Export workflows
        │
        ▼
Redact credentials
        │
        ▼
Scan
        │
        ▼
Commit
        │
        ▼
Push
```

There is no need to manually export individual workflows.

## Manual Git Commands

Check changes:

```bash
cd /home/sanjay/n8n-git
git status
```

View changes:

```bash
git diff
```

View staged changes:

```bash
git diff --cached
```

View history:

```bash
git log --oneline --decorate --graph
```

View a specific commit:

```bash
git show <commit>
```

Push manually if necessary:

```bash
git push origin main
```

## Important: Don't Run Git as Root

The backup script is designed to be invoked with:

```bash
sudo /usr/local/bin/n8n-git-backup
```

Docker operations require elevated privileges, while Git operations are explicitly executed as `sanjay`.

This is important because the GitHub SSH key belongs to `sanjay`.

Running Git directly as `root` can result in:

```text
Permission denied (publickey)
```

The script avoids this by executing Git operations as `sanjay`.

## File Ownership

The Git repository should be owned by:

```text
sanjay:sanjay
```

Check:

```bash
ls -ld /home/sanjay/n8n-git
```

If ownership becomes incorrect:

```bash
sudo chown -R sanjay:sanjay /home/sanjay/n8n-git
```

## Backup Philosophy

This setup intentionally separates two concerns.

### Workflow version control

GitHub contains:

- workflow structure
- node configuration
- expressions
- URLs
- workflow metadata
- credential references
- sanitized configuration

### Credential management

n8n contains:

- API keys
- tokens
- passwords
- OAuth secrets
- other sensitive values

This separation is intentional.

## Disaster Recovery

In a disaster-recovery scenario:

1. Restore n8n.
2. Restore/import workflow definitions from GitHub.
3. Recreate/configure n8n credentials.
4. Associate the credentials with the workflows.
5. Test the workflows.

The Git repository provides workflow configuration but intentionally does not provide production secret values.

## Testing the Sanitizer

Run:

```bash
sudo /usr/local/bin/n8n-git-backup
```

Verify:

```text
Secret scan passed.
```

Then:

```bash
cd /home/sanjay/n8n-git
git status
```

The repository should be clean.

## Current Repository State

Main branch:

```text
main
```

Clean-history root:

```text
bdcd75f Initial clean n8n workflow repository
```

A successful sanitized workflow backup has been pushed as:

```text
fc10a8f
```

## Security Recommendations

1. Never intentionally commit live API keys.
2. Never bypass GitHub Push Protection simply to make a backup succeed.
3. Rotate credentials immediately if they are exposed.
4. Keep the Git repository private.
5. Treat Git history as sensitive.
6. Keep n8n credentials separate from workflow version control.
7. Update the sanitizer when a new credential format is discovered.
8. Review unexpected secret-scan failures rather than disabling the scan.
9. Keep SSH keys outside the Git repository.
10. Periodically verify that the remote history does not contain old secret-bearing commits.

## Quick Reference

### Run backup

```bash
sudo /usr/local/bin/n8n-git-backup
```

### Check status

```bash
cd /home/sanjay/n8n-git
git status
```

### View history

```bash
git log --oneline --decorate --graph -10
```

### Verify remote

```bash
git fetch origin
git status
```

### Test GitHub SSH

```bash
sudo -u sanjay ssh -T git@github.com
```

### Repository

```text
/home/sanjay/n8n-git
```

### Workflows

```text
/home/sanjay/n8n-git/workflows
```

### Backup script

```text
/usr/local/bin/n8n-git-backup
```

### n8n container

```text
n8n-n8n-1
```

### GitHub repository

```text
shunyaghora/n8n-workflows
```

## Summary

The n8n Git backup system is designed around one core principle:

> **Version-control the workflows, not the secrets.**

The live n8n environment retains the real credentials.

The GitHub repository retains sanitized workflow definitions.

The backup script automatically exports, sanitizes, scans, commits, and pushes the workflows while stopping if recognizable secrets remain.

This provides Git-based workflow history without intentionally turning GitHub into a credential store.
