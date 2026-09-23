# n8n Workflow Version Control

Git-based version control for workflows from the self-hosted n8n instance on `rudraghora`.

## Environment

- Server: `rudraghora`
- n8n version: `2.36.9`
- n8n container: `n8n-n8n-1`
- Git repository: `git@github.com:shunyaghora/n8n-workflows.git`
- Local Git directory: `/home/sanjay/n8n-git`
- Workflow directory: `/home/sanjay/n8n-git/workflows`

## Purpose

This repository provides manual Git version control for n8n workflows.

It allows you to:

- Save a snapshot of all n8n workflows.
- See the history of workflow changes.
- Compare workflow versions.
- Restore an earlier workflow version.
- Keep workflow history independently from the normal n8n backup process.

The complete n8n instance is backed up separately by the existing Docker backup process.

---

## Repository Structure

```text
/home/sanjay/n8n-git/
├── .git/
├── README.md
└── workflows/
    ├── workflow-id-1.json
    ├── workflow-id-2.json
    └── ...
