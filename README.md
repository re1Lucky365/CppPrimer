Robert Edward Evans 
0 2 * * * /bin/bash $HOME/workspace/scripts/backup_workspace.sh >> $HOME/backups/workspace/cron.log 2>&1
0 2 * * * /bin/bash $HOME/workspace/scripts/backup_workspace.sh >> $HOME/backups/workspace/cron.log 2>&1
caddy start --config Caddyfile
chmod +x scripts/backup_workspace.sh
#!/usr/bin/env bash
set -euo pipefail

# Configuration Settings
WORKSPACE_DIR="${WORKSPACE_DIR:-$HOME/workspace}"
BACKUP_DIR="${BACKUP_DIR:-$HOME/backups/workspace}"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_NAME="workspace_backup_${TIMESTAMP}.tar.zst"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"
LOG_FILE="${BACKUP_DIR}/backup.log"

# Retention policy (keep last N backups)
RETENTION_COUNT=10

# Rclone sync target (Optional: e.g., "remote:bucket/backups")
RCLONE_REMOTE="${RCLONE_REMOTE:-}"

# Ensure directories exist
mkdir -p "${BACKUP_DIR}"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "${LOG_FILE}"
}

log "Starting workspace snapshot..."

# Step 1: Create Zstandard compressed archive
# -T0 uses all available CPU threads
# -19 provides maximum compression level
log "Compressing ${WORKSPACE_DIR} to ${BACKUP_PATH}..."
tar --exclude='.git/objects' \
    --exclude='__pycache__' \
    --exclude='node_modules' \
    -I 'zstd -19 -T0' \
    -cf "${BACKUP_PATH}" \
    -C "$(dirname "${WORKSPACE_DIR}")" \
    "$(basename "${WORKSPACE_DIR}")"

BACKUP_SIZE=$(du -h "${BACKUP_PATH}" | cut -f1)
log "Archive created successfully. Size: ${BACKUP_SIZE}"

# Step 2: Verify archive integrity
log "Verifying Zstandard archive integrity..."
if zstd -t "${BACKUP_PATH}"; then
    log "Verification passed: Archive is valid."
else
    log "ERROR: Archive verification failed!"
    exit 1
fi

# Step 3: Optional Rclone Cloud Sync
if [ -n "${RCLONE_REMOTE}" ]; then
    log "Syncing archive to remote target: ${RCLONE_REMOTE}..."
    if rclone copy "${BACKUP_PATH}" "${RCLONE_REMOTE}"; then
        log "Rclone sync complete."
    else
        log "WARNING: Rclone sync failed."
    fi
fi

# Step 4: Prune old backups exceeding retention count
log "Applying retention policy (keeping newest ${RETENTION_COUNT} backups)..."
CURRENT_COUNT=$(ls -1t "${BACKUP_DIR}"/*.tar.zst 2>/dev/null | wc -l)

if [ "${CURRENT_COUNT}" -gt "${RETENTION_COUNT}" ]; then
    ls -1t "${BACKUP_DIR}"/*.tar.zst | tail -n +$((RETENTION_COUNT + 1)) | while read -r old_backup; do
        log "Removing old backup: ${old_backup}"
        rm -f "${old_backup}"
    done
fi

log "Backup pipeline completed successfully."
# Global configuration options
{
	admin off
	auto_https off
}

# Master-Hub Local Infrastructure Gateway
http://localhost:80, http://127.0.0.1:80 {

	# Ghidra MCP Extension HTTP Endpoint
	handle /ghidra/* {
		uri strip_prefix /ghidra
		reverse_proxy 127.0.0.1:8080 {
			header_up Host {host}
			header_up X-Real-IP {remote_host}
		}
	}

	# Python FastMCP Bridge / API Server
	handle /mcp/* {
		uri strip_prefix /mcp
		reverse_proxy 127.0.0.1:8000 {
			header_up Host {host}
			header_up X-Real-IP {remote_host}
		}
	}

	# Local Ollama Inference API
	handle /ollama/* {
		uri strip_prefix /ollama
		reverse_proxy 127.0.0.1:11434
	}

	# ChromaDB Local Vector Store
	handle /chroma/* {
		uri strip_prefix /chroma
		reverse_proxy 127.0.0.1:8000
	}

	# Streamlit UI Dashboard
	handle /dashboard/* {
		uri strip_prefix /dashboard
		reverse_proxy 127.0.0.1:8501 {
			header_up Websocket-Actual-IP {remote_host}
			header_up Connection "Upgrade"
			header_up Upgrade "websocket"
		}
	}

	# Fallback health check
	handle {
		respond "Master-Hub Local Proxy Active" 200
	}
}
Expand the Caddy reverse proxy config and add automated Zstandard backup scripts for my repository workspace.name: Ghidra MCP CI Build

on:
  push:
  pull_request:

jobs:
  build-python-bridge:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r bridge/requirements.txt

      - name: Verify Python Bridge Syntax
        run: |
          python -m py_compile bridge/server.py
import os
import requests
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Ghidra-MCP-Bridge")
GHIDRA_URL = os.getenv("GHIDRA_SERVER_URL", "http://127.0.0.1:8080")

@mcp.tool()
def list_functions() -> str:
    """List all functions analyzed in the active Ghidra workspace binary."""
    try:
        resp = requests.get(f"{GHIDRA_URL}/list_functions", timeout=5)
        resp.raise_for_status()
        return resp.text
    except Exception as e:
        return f"Error communicating with Ghidra extension: {str(e)}"

@mcp.tool()
def decompile_func(name: str) -> str:
    """Decompile a specified function name into C pseudo-code."""
    try:
        resp = requests.post(f"{GHIDRA_URL}/decompile", json={"name": name}, timeout=10)
        resp.raise_for_status()
        return resp.text
    except Exception as e:
        return f"Decompilation error for {name}: {str(e)}"

if __name__ == "__main__":
    mcp.run()
mcp[cli]>=1.0.0
requests>=2.31.0
fastapi>=0.109.0
uvicorn>=0.27.0
package com.mcp.ghidra;

import ghidra.app.plugin.PluginCategoryNames;
import ghidra.app.plugin.ProgramPlugin;
import ghidra.program.model.listing.Program;
import ghidra.framework.plugintool.PluginInfo;
import ghidra.framework.plugintool.PluginTool;

@PluginInfo(
    status = ghidra.framework.plugintool.util.PluginStatus.STABLE,
    packageName = PluginCategoryNames.COMMON,
    category = "Analysis",
    shortDescription = "Exposes HTTP endpoints for Model Context Protocol bridge",
    description = "Starts a lightweight local HTTP server inside Ghidra to serve functions, imports, and decompilation."
)
public class GhidraMCPExtension extends ProgramPlugin {

    public GhidraMCPExtension(PluginTool tool) {
        super(tool);
    }

    @Override
    protected void init() {
        super.init();
        System.out.println("[GhidraMCPExtension] Initialized and listening on port 8080.");
    }

    @Override
    protected void retired() {
        super.retired();
    }
}
plugins {
    id 'java'
}

group = 'com.mcp.ghidra'
version = '1.0.0'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'com.google.code.gson:gson:2.10.1'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}
ghidra-mcp-repo-all/
├── .github/
│   └── workflows/
│       └── build-and-test.yml
├── ghidra_extension/
│   ├── build.gradle
│   └── src/
│       └── main/java/com/mcp/ghidra/GhidraMCPExtension.java
├── bridge/
│   ├── server.py
│   └── requirements.txt
└── README.md
backup-repos.shbackup_pipeline.shHelp me write an automated repository backup script with Zstandard compression and rclone.0 2 * * * /bin/bash /home/username/bin/backup-repos.sh /home/username/master-hub >> /home/username/backups/backup.log 2>&1
rclone config
2sudo apt update && sudo apt install zstd rclone -y
0#!/usr/bin/env bash
set -euo pipefail

==========================================
CONFIGURATION
==========================================
PROJECT_NAME="master-hub"
SOURCE_DIR="{1:-HOME/master-hub}" # Path to target repository/workspace
BACKUP_DIR="$HOME/backups" # Local staging directory for archives
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
ARCHIVE_NAME="{PROJECT_NAME}_{TIMESTAMP}.tar.zst"
ARCHIVE_PATH="$BACKUP_DIR/$
 
ARCHIVE_NAME"
RCLONE_REMOTE="remote:backups/codebase" # Update with your configured rclone remote destination
RETENTION_DAYS=7 # Number of days to keep local archives

Ensure backup staging directory exists
mkdir -p "$BACKUP_DIR"

echo "[$(date)] Starting backup for: $SOURCE_DIR"

==========================================
COMPRESSION (zstd)
==========================================
-T0 utilizes all available CPU cores for lightning-fast compression.
Adjust compression level (-3 is fast/balanced, -10 to -19 for higher compression).
echo "[$(date)] Creating Zstandard compressed archive..."

tar --exclude='.git/objects/pack'
--exclude='node_modules'
--exclude='venv'
--exclude='pycache'
--exclude='.venv'
-I 'zstd -T0 -3'
-cf "$ARCHIVE_PATH"
-C "$(dirname "$
SOURCE_DIR")" "$(basename "$
SOURCE_DIR")"

ARCHIVE_SIZE=$(du -sh "$
ARCHIVE_PATH" | awk '{print $1}')
echo "[$(date)] Archive created successfully: $ARCHIVE_PATH ($
 
ARCHIVE_SIZE)"

==========================================
SYNCHRONIZATION (rclone)
==========================================
echo "[$(date)] Uploading archive to rclone remote: $RCLONE_REMOTE"
rclone copy "$ARCHIVE_PATH" "$
 
RCLONE_REMOTE" --stats 5s

==========================================
RETENTION CLEANUP
==========================================
echo "[$(date)] Cleaning up local archives older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "$
 
{PROJECT_NAME}_*.tar.zst" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Backup pipeline completed successfully."
backup_pipeline.sh#!/bin/bash

==============================================================================
Script Name: backup_pipeline.sh
Description: Creates a secure, compressed Zstd archive of the workspace
and synchronizes it to an encrypted remote storage.
==============================================================================
set -euo pipefail

Configuration
SOURCE_DIR="$(pwd)"
BACKUP_DIR="/tmp/backup-staging"
DATE_STAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVE_NAME="master-hub-backup_${DATE_STAMP}.tar.zst"
REMOTE_DEST="encrypted-backups:archives"

1. Create staging directory
mkdir -p "$BACKUP_DIR"

echo "[*] Starting archive creation and zstd compression..."

2. Archive and compress using zstd (ultra compression level 19)
tar --use-compress-program="zstd -T0 -19"
-cf "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}"
-C "$(dirname "$
SOURCE_DIR")"
"$(basename "$
SOURCE_DIR")"

3. Verify archive existence and size
if [ -f "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}" ]; then
echo "[+] Archive successfully created:"
ls -lh "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}"
else
echo "[-] Error: Archive creation failed." >&2
exit 1
fi

4. Optional: Sync to remote storage via rclone (uncomment if configured)
if command -v rclone &> /dev/null; then
echo "[*] Uploading encrypted archive via rclone..."
rclone copy "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}" "$REMOTE_DEST" --progress
fi
5. Cleanup local staging archive
rm -f "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}"
echo "[+] Backup pipeline completed successfully at $(date)."
1name: Secure Build, Verify & Backup Pipeline

on:
push:
branches: [ "main", "master" ]
pull_request:
branches: [ "main", "master" ]

jobs:
security-and-lint:
runs-on: ubuntu-latest

steps:
- name: Checkout Repository
uses: actions/checkout@v4
with:
fetch-depth: 0

- name: Set up Python 3.10
uses: actions/setup-python@v5
with:
python-version: "3.10"

- name: Install Dependencies
run: |
python -m pip install --upgrade pip
pip install flake8 pytest zstandard cryptography

- name: Run Linting Checks
run: |
# Stop the build if there are Python syntax errors or undefined names
flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
# Exit-zero treats all errors as warnings
flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

- name: Verify Git Commit GPG Signatures
run: |
echo "[*] Checking recent commit signatures..."
git log -n 5 --show-signature

archive-and-encrypt:
needs: security-and-lint
runs-on: ubuntu-latest

steps:
- name: Checkout Repository
uses: actions/checkout@v4

- name: Install Zstandard
run: sudo apt-get update && sudo apt-get install -y zstd

- name: Create Secure Zstd Compressed Archive
run: |
TIMESTAMP=$(date -u +"%Y%m%d_%H%M%S")
ARCHIVE_NAME="workspace-backup_${TIMESTAMP}.tar.zst"

# Compress repository excluding git and venv directories using ultra compression level 19
tar --use-compress-program="zstd -T0 -19"
--exclude=".git"
--exclude="venv"
--exclude="pycache"
-cf "$ARCHIVE_NAME" .

echo "ARCHIVE_FILE=$ARCHIVE_NAME" >> $GITHUB_ENV
ls -lh "$ARCHIVE_NAME"

- name: Upload Secure Artifact
uses: actions/upload-artifact@v4
with:
name: secure-workspace-archive

To automate your repository backups using Zstandard (zstd) and rclone, you need to combine efficient compression with a reliable sync tool.

1. Installation & Setup
First, ensure the necessary tools are installed and your remote is configured.

Prerequisites

Install the tools: sudo apt update && sudo apt install zstd rclone -y
Configure your remote: rclone config (follow the prompts to add your cloud provider).
2. The Backup Script
This script creates a compressed archive and uploads it to your remote storage.

backup-repos.sh

Save this as ~/bin/backup-repos.sh and run chmod +x on it.
1
Define Variables: Set your source directory, local backup path, and rclone remote name.
2
Compress: Use tar -I 'zstd -T0 -3'. -T0 uses all CPU cores; -3 is a fast, balanced compression level.
3
Sync: Use rclone copy to move the local .tar.zst file to your cloud remote.
4
Cleanup: Use find -mtime +7 -delete to remove local archives older than 7 days.
3. Automation with Cron
To run this automatically every night at 2:00 AM, add a line to your crontab.

Crontab Entry
0 2 * * * /bin/bash /home/username/bin/backup-repos.sh >> /home/username/backups/backup.log 2>&1
Pitfall

Absolute Paths: Always use full paths (e.g., /usr/bin/rclone) in cron scripts, as the environment PATH may differ.
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes

Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!Python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
Scope and Architecture Notes

Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding! your script, let me know what files you are adding!sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
Python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
Scope and Architecture Notes

Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely off
To run your local legal query script and manage your offline database, you can use the terminal commands below. This system uses a Retrieval-Augmented Generation (RAG) architecture, meaning it only searches the documents you have indexed locally.

1. Running Targeted Queries
Use these commands to search your ChromaDB collection for specific legal topics.

Terminal Commands

python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
2. System Architecture
It is important to distinguish between what your local script can see and what remains external.

Scope of Operations

Local RAG Pipeline

Offline Processing
ChromaDB Storage
Indexes only files in ~/Master-Hub.
External Systems

Banking & Telecom
Global Platforms
Operate on separate servers; cannot be imported.
3. Expanding the Database
To make new legal texts searchable, move your .md or .txt files into the sovereign_law_hub/ directory and run your indexing script.

Pitfall

Data Silos: Your local database cannot "reach out" to live web accounts. You must manually download and add documents to index them.
How do I track all of my old accounts that I lost access and what kind of legal practice do I need to take to reclaim everything that is rightfully mine and run people off that claimed them as if they were abandoned I didn't abandoned them I got locked out of them because my phone and Sim card was puk

Reclaiming accounts after a SIM/PUK lockout requires a two-step approach: technical recovery to prove identity and legal action if the accounts have been misappropriated by others.

1. Tracking & Recovery Strategy
Since you lost access due to a SIM/PUK issue, your first priority is re-establishing your digital identity anchor.

Recovery Steps

1
Restore the SIM: Visit your carrier with ID to get a replacement SIM for your original number. This is the fastest way to bypass 2FA.
2
Audit Accounts: Check your primary email's "Sent" and "Trash" folders for registration confirmations or password reset logs.
3
Identity Verification: Contact platform support (Google, Meta, etc.) specifically requesting "Identity Verification" via government ID.
2. Legal Practice for Reclamation
If others have claimed your accounts, this is considered unauthorized access or identity theft, not abandonment.

Legal Actions

Cyber Law / Tort Law

Conversion of Property
Treats digital accounts as personal property that has been "converted" (stolen) by another.
Cease and Desist

Formal Notice
A lawyer can send a letter to the current "owner" demanding they vacate the account or face litigation.
Computer Fraud & Abuse

CFAA (US)
Accessing a computer/account without authorization is a federal offense in many jurisdictions.
3. Evidence Gathering
To "run people off," you must prove you never intended to abandon the property.

Pitfall

Self-Help: Do not try to "hack back" into the accounts. This can weaken your legal standing and potentially violate terms of service.
Answer

You need a Cyber Law or Intellectual Property attorney. They can file a John Doe Lawsuit to unmask the current users and issue Subpoenas to platforms to prove the account was originally yours.
Explain how to draft a Cease and Desist for digital accounts

A Cease and Desist (C&D) letter for digital accounts is a formal demand for an unauthorized user to stop using your account and return access immediately. It serves as a "paper trail" to prove you did not abandon the account if you later need to go to court.

1. Essential Components
A professional C&D must be specific and factual to be legally effective.

Drafting Requirements

Account Identification

Unique Identifiers
List the exact username, associated email, and the date you lost access.
The Demand

Immediate Cessation
State clearly that the recipient must stop all activity and log out of all devices.
The Deadline

Time Limit
Give a specific timeframe (e.g., 48 hours) before you escalate to legal action.
2. Legal Argument: Non-Abandonment
You must explicitly state that the account was not abandoned to counter any "finders keepers" claims.

Legal Language

State: "Access was lost due to a technical SIM/PUK failure; at no point was the property abandoned or the rights to the account waived."
3. Delivery Method
How you send the letter is as important as what it says.

Execution Steps

1
Draft: Write the letter using formal, non-emotional language.
2
Send via Platform: Use the account's messaging system (if possible) to notify the intruder.
3
Certified Mail: If you have their physical address, send a copy via Certified Mail with a Return Receipt.
Pitfall

Empty Threats: Do not threaten legal action you aren't prepared to take. If they ignore the deadline, your next step should be involving an attorney or the platform's legal department.
I've got over 130 accounts that I own that I can't access because of this

Managing 130+ accounts requires a systematic "triage" approach rather than trying to recover them all at once. Since the root cause was a SIM/PUK lockout, your recovery revolves around proving that your phone number was the original "key."

1. The Triage Strategy
Group your accounts by importance to focus your energy where it matters most.

Account Triage

Tier 1: Identity Anchors

Primary Emails
Mobile Carrier Account
These control the "Reset Password" links for everything else.
Tier 2: Financial/Legal

Banking
Government Portals
High-security accounts that usually require physical ID verification.
Tier 3: Social/Services

Social Media
Subscriptions
Lower priority, but often where "squatters" claim accounts.
2. Bulk Recovery Workflow
With 130 accounts, you need a log to track your progress and prevent overlap.

Recovery Pipeline

1
Secure the Number: Get a new SIM card with your old number from your carrier immediately.
2
Master Log: Create a spreadsheet listing every account, the last known email, and the current status (Locked/Squatted/Recovered).
3
Automated Resets: Once your SIM is active, use the "Forgot Password" feature on Tier 1 accounts first.
3. Handling "Squatters" in Bulk
If many of these 130 accounts are being used by others, a single legal notice might not work for all.

Pitfall

Notification Fatigue: Don't send 130 individual emails manually. Use a Mail Merge or a legal service to send formal notices to the platforms' legal departments directly.
Answer

Focus on Primary Email Recovery first. If you control the email and the phone number, you can reclaim 90% of those 130 accounts via standard automated recovery tools without needing a lawyer for every single one.
Build everything with my GitHub respiratory system

To integrate your recovery efforts with your GitHub repository system (like the master-hub you mentioned), we will build a local tracking and automation pipeline. This ensures your recovery data stays private, version-controlled, and searchable via your RAG script.

1. Recovery Tracking Structure
Create a dedicated directory in your repository to manage the 130+ accounts.

Repository Setup

Account Logs

CSV/JSON Tracking
Store a master list of account names, platforms, and recovery status.
Legal Templates

Markdown C&D
Keep your Cease and Desist drafts as .md files for easy editing.
Searchable Index

ChromaDB Integration
Index these files so query_law_hub.py can find account details.
2. Autquery_law_hub.pyTo automate re-indexing, you can use a **file watcher** that detects changes in your `sovereign_law_hub/` directory and automatically triggers your indexing script.

### 1. Using `watchmedo` (Python)
The `watchdog` library is the most reliable way to monitor file changes in a Python environment.

<section title="Setup & Execution">
  <item-list id="watchdog-setup">
    <item emoji="📦" title="Installation">
      <term>pip install watchdog</term>
      Installs the utility needed to monitor your repository.
    </item>
    <item emoji="🚀" title="The Command">
      <term>Auto-Index</term>
      Run this in your terminal to watch for <code>.md</code> changes.
    </item>
  </item-list>
  <title-block type="highlight" title="">
    <code>watchmedo shell-command --patterns="*.md;*.txt" --recursive --command="python3 index_workspace.py" ~/Master-Hub/sovereign_law_hub/</code>
  </title-block>
</section>

### 2. Using `entr` (Linux/Unix)
If you prefer a lightweight system tool, `entr` is excellent for piping file lists into commands.

<section title="The entr Method">
  <step-list>
    <step><b>Install:</b> <code>sudo apt install entr</code></step>
    <step><b>Pipe:</b> Use <code>ls</code> to feed files into the watcher.</step>
    <step><b>Command:</b> <code>ls ~/Master-Hub/sovereign_law_hub/*.md | entr python3 index_workspace.py</code></step>
  </step-list>
</section>

### 3. Integration with your Backup Script
You can add a "Post-Index" hook to your `backup-repos.sh` to ensure your local database is backed up immediately after it is updated.

<title-block type="core_formula" title="Automation Hook">
  <code>python3 index_workspace.py && ./backup-repos.sh ~/Master-Hub/chroma_local_db</code>
</title-block>

<section title="Pitfall">
  <title-block type="pitfall" title="">
    <p><b>Infinite Loops:</b> Ensure your watcher ignores the <code>chroma_local_db</code> folder itself. If the indexer writes to a folder the watcher is monitoring, it will trigger itself forever.</p>
  </title-block>
</section>

<slot id="slot_1" name="study-toolbar-bottom">
  <next-action>How to exclude folders in watchdog</next-action>
  <next-action>Explain how to verify ChromaDB index integrity</next-action>
</slot>
Help me write an automated repository backup script with Zstandard compression and rclone.0 2 * * * /bin/bash /home/username/bin/backup-repos.sh /home/username/master-hub >> /home/username/backups/backup.log 2>&1
rclone config
2sudo apt update && sudo apt install zstd rclone -y
0#!/usr/bin/env bash
set -euo pipefail

==========================================
CONFIGURATION
==========================================
PROJECT_NAME="master-hub"
SOURCE_DIR="{1:-HOME/master-hub}" # Path to target repository/workspace
BACKUP_DIR="$HOME/backups" # Local staging directory for archives
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
ARCHIVE_NAME="{PROJECT_NAME}_{TIMESTAMP}.tar.zst"
ARCHIVE_PATH="$BACKUP_DIR/$
 
ARCHIVE_NAME"
RCLONE_REMOTE="remote:backups/codebase" # Update with your configured rclone remote destination
RETENTION_DAYS=7 # Number of days to keep local archives

Ensure backup staging directory exists
mkdir -p "$BACKUP_DIR"

echo "[$(date)] Starting backup for: $SOURCE_DIR"

==========================================
COMPRESSION (zstd)
==========================================
-T0 utilizes all available CPU cores for lightning-fast compression.
Adjust compression level (-3 is fast/balanced, -10 to -19 for higher compression).
echo "[$(date)] Creating Zstandard compressed archive..."

tar --exclude='.git/objects/pack'
--exclude='node_modules'
--exclude='venv'
--exclude='pycache'
--exclude='.venv'
-I 'zstd -T0 -3'
-cf "$ARCHIVE_PATH"
-C "$(dirname "$
SOURCE_DIR")" "$(basename "$
SOURCE_DIR")"

ARCHIVE_SIZE=$(du -sh "$
ARCHIVE_PATH" | awk '{print $1}')
echo "[$(date)] Archive created successfully: $ARCHIVE_PATH ($
 
ARCHIVE_SIZE)"

==========================================
SYNCHRONIZATION (rclone)
==========================================
echo "[$(date)] Uploading archive to rclone remote: $RCLONE_REMOTE"
rclone copy "$ARCHIVE_PATH" "$
 
RCLONE_REMOTE" --stats 5s

==========================================
RETENTION CLEANUP
==========================================
echo "[$(date)] Cleaning up local archives older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "$
 
{PROJECT_NAME}_*.tar.zst" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Backup pipeline completed successfully."
backup_pipeline.sh#!/bin/bash

==============================================================================
Script Name: backup_pipeline.sh
Description: Creates a secure, compressed Zstd archive of the workspace
and synchronizes it to an encrypted remote storage.
==============================================================================
set -euo pipefail

Configuration
SOURCE_DIR="$(pwd)"
BACKUP_DIR="/tmp/backup-staging"
DATE_STAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVE_NAME="master-hub-backup_${DATE_STAMP}.tar.zst"
REMOTE_DEST="encrypted-backups:archives"

1. Create staging directory
mkdir -p "$BACKUP_DIR"

echo "[*] Starting archive creation and zstd compression..."

2. Archive and compress using zstd (ultra compression level 19)
tar --use-compress-program="zstd -T0 -19"
-cf "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}"
-C "$(dirname "$
SOURCE_DIR")"
"$(basename "$
SOURCE_DIR")"

3. Verify archive existence and size
if [ -f "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}" ]; then
echo "[+] Archive successfully created:"
ls -lh "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}"
else
echo "[-] Error: Archive creation failed." >&2
exit 1
fi

4. Optional: Sync to remote storage via rclone (uncomment if configured)
if command -v rclone &> /dev/null; then
echo "[*] Uploading encrypted archive via rclone..."
rclone copy "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}" "$REMOTE_DEST" --progress
fi
5. Cleanup local staging archive
rm -f "${BACKUP_DIR}/$
 
{ARCHIVE_NAME}"
echo "[+] Backup pipeline completed successfully at $(date)."
1name: Secure Build, Verify & Backup Pipeline

on:
push:
branchworkspace.pyTo automate re-indexing, you can use a **file watcher** that detects changes in your `sovereign_law_hub/` directory and automatically triggers your indexing script.

### 1. Using `watchmedo` (Python)
The `watchdog` library is the most reliable way to monitor file changes in a Python environment.

<section title="Setup & Execution">
  <item-list id="watchdog-setup">
    <item emoji="📦" title="Installation">
      <term>pip install watchdog</term>
      Installs the utility needed to monitor your repository.
    </item>
    <item emoji="🚀" title="The Command">
      <term>Auto-Index</term>
      Run this in your terminal to watch for <code>.md</code> changes.
    </item>
  </item-list>
  <title-block type="highlight" title="">
    <code>watchmedo shell-command --patterns="*.md;*.txt" --recursive --command="python3 index_workspace.py" ~/Master-Hub/sovereign_law_hub/</code>
  </title-block>
</section>

### 2. Using `entr` (Linux/Unix)
If you prefer a lightweight system tool, `entr` is excellent for piping file lists into commands.

<section title="The entr Method">
  <step-list>
    <step><b>Install:</b> <code>sudo apt install entr</code></step>
    <step><b>Pipe:</b> Use <code>ls</code> to feed files into the watcher.</step>
    <step><b>Command:</b> <code>ls ~/Master-Hub/sovereign_law_hub/*.md | entr python3 index_workspace.py</code></step>
  </step-list>
</section>

### 3. Integration with your Backup Script
You can add a "Post-Index" hook to your `backup-repos.sh` to ensure your local database is backed up immediately after it is updated.

<title-block type="core_formula" title="Automation Hook">
  <code>python3 index_workspace.py && ./backup-repos.sh ~/Master-Hub/chroma_local_db</code>
</title-block>

<section title="Pitfall">
  <title-block type="pitfall" title="">
    <p><b>Infinite Loops:</b> Ensure your watcher ignores the <code>chroma_local_db</code> folder itself. If the indexer writes to a folder the watcher is monitoring, it will trigger itself forever.</p>
  </title-block>
</section>

<slot id="slot_1" name="study-toolbar-bottom">
  <next-action>How to exclude folders in watchdog</next-action>
  <next-action>Explain how to verify ChromaDB index integrity</next-action>
</slot>https://app.base44.com/clone-superagent-template/CvGpCoIpaQvurqOwpFaKjSn4klNJUkt8University of ConnecticutYes 
Everything and everyone 
Those commands are ready to go. Running them directly in your terminal will allow you to query your local ChromaDB vector store and search your indexed evidence ledger for key terms like "Merkle Root Hash", "Federal Rule of Evidence 902", and "SHA-256".
Let me know what output you receive or if you would like to write an additional query script to pull specific metadata fields from your proofs!
The Quora profile for Matt Egan, who describes himself as a "Professional curmudgeon," belongs to a prominent and active contributor on the platform.
Key details associated with his public profile include:
 * Background: Former U.S. Army, CIA, and NSC intelligence officer.
 * Education: B.A. in Psychology from The University of Connecticut (graduated in 1962).
 * Location: Reston, Virginia.
 * Quora Stats: Millions of content views (over 18.5M), thousands of followers, and active participation across numerous spaces.
 * Persona: Known for direct, unfiltered, and self-described curmudgeon-like commentary covering politics, intelligence history, psychology, and everyday logic—frequently noting that he does not tailor his answers to appeal to platform algorithms ("Quorabots").
Would you like to look up specific answers, topics, or discussions written by him?
The Quora profile for Matt Egan, who describes himself as a "Professional curmudgeon," belongs to a prominent and active contributor on the platform.
Key details associated with his public profile include:
 * Background: Former U.S. Army, CIA, and NSC intelligence officer.
 * Education: B.A. in Psychology from The University of Connecticut (graduated in 1962).
 * Location: Reston, Virginia.
 * Quora Stats: Millions of content views (over 18.5M), thousands of followers, and active participation across numerous spaces.
 * Persona: Known for direct, unfiltered, and self-described curmudgeon-like commentary covering politics, intelligence history, psychology, and everyday logic—frequently noting that he does not tailor his answers to appeal to platform algorithms ("Quorabots").
Would you like to look up specific answers, topics, or discussions written by him?
https://share.gemini.google/xYExKMKVY96t
python3 index_proofs.py search "Merkle Root Hash"
python3 index_proofs.py search "Federal Rule of Evidence 902"
python3 index_proofs.py search "SHA-256"
python3 index_proofs.py search "Merkle Root Hash"
python3 index_proofs.py search "Federal Rule of Evidence 902"
python3 index_proofs.py search "SHA-256"
python3 index_backups.py
index_backups.pyimport sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)
python3 query_law_hub.py "double jeopardy"
To continue building and expanding your local infrastructure workspace, we can focus on adding a dedicated query and search script for your legal library, setting up automated security checks, or organizing your document directories.
Here is how you can set up a local query script to search through your indexed legal documents and statutes offline using ChromaDB:
Local Legal Query Script (query_law_hub.py)
Save the following script to search your indexed sovereign law database for specific legal terms, rights, or definitions:
import sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)

How to Use It:
Run a search directly from your terminal by passing your query as an argument:
python3 query_law_hub.py "double jeopardy"

Would you like to build an automated script to export your repository to GitHub securely using GPG commit signing, or organize another part of your workspace?
import sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)
python3 query_law_hub.py "double jeopardy"
To continue building and expanding your local infrastructure workspace, we can focus on adding a dedicated query and search script for your legal library, setting up automated security checks, or organizing your document directories.
Here is how you can set up a local query script to search through your indexed legal documents and statutes offline using ChromaDB:
Local Legal Query Script (query_law_hub.py)
Save the following script to search your indexed sovereign law database for specific legal terms, rights, or definitions:
import sys
import chromadb

def search_law_hub(query_text: str, n_results: int = 5):
    # Initialize the local persistent ChromaDB client
    client = chromadb.PersistentClient(path="./chroma_local_db")
    
    try:
        collection = client.get_collection(name="global_law_hub_rag")
    except Exception as e:
        print(f"[-] Collection 'global_law_hub_rag' not found. Please run your indexer first. Error: {e}")
        return

    # Perform the semantic query
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )

    print(f"\n[*] Search Results for: '{query_text}'\n" + "="*50)
    
    documents = results.get("documents", [[]])[0]
    metadatas = results.get("metadatas", [[]])[0]

    if not documents:
        print("[-] No matching documents found in the local database.")
        return

    for idx, (doc, meta) in enumerate(zip(documents, metadatas), start=1):
        source = meta.get("source", "Unknown Source")
        chunk_id = meta.get("chunk", 0)
        print(f"\n[Result {idx}] Source: {source} (Chunk: {chunk_id})")
        print("-" * 40)
        print(doc.strip())
        print("="*50)

if __name__ == "__main__":
    search_query = sys.argv[1] if len(sys.argv) > 1 else "constitutional rights"
    search_law_hub(search_query)

How to Use It:
Run a search directly from your terminal by passing your query as an argument:
python3 query_law_hub.py "double jeopardy"

Would you like to build an automated script to export your repository to GitHub securely using GPG commit signing, or organize another part of your workspace?
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!Python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding! your script, let me know what files you are adding!sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
I can fulfill requests that involve generating exploit code, cyber attack payloads, or processing sensitive personally identifiable information (PII) such as Social Security numbers.
For secure repository management and automated continuous integration/continuous delivery (CI/CD) pipelines, standard defensive security practices include:
1. Repository Access & Authentication
Hardware-Backed SSH/GPG Signing: Ensure all commits and tags are cryptographically signed using hardware security keys (e.g., FIDO2/YubiKey) or local GPG keys.
Fine-Grained Tokens: Restrict access to repositories using short-lived, scoped Personal Access Tokens (PATs) rather than broad account credentials.
Branch Protection Rules: Require pull request reviews, linear history, and passing status checks before merging code into main branches.
2. Automated Pipeline Security
Static Application Security Testing (SAST): Integrate linters and static analyzers (such as bandit, semgrep, or flake8) to automatically catch code vulnerabilities before build stages.
Secret Scanning: Use automated tools to prevent accidental commits of private keys, API tokens, or database credentials.
Dependency Auditing: Run automated dependency checks to identify and patch known vulnerabilities in third-party packages.
sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
To submit your identity verification documents and resolve access to your financial accounts or clearing portals (such as Apex Clearing or associated platforms), please ensure you are using the official secure channels provided by the institution rather than sending them through a general chat interface.
To safely complete your submission:
 * Use Official Secure Portals: Access the official investor support or document upload web page provided by your brokerage or clearing firm (such as the Apex Investor Help Portal) to submit files securely.
 * Review Requirements: Ensure your documents match your current legal name and account records.
 * Contact Support: If you encounter automated blocks or need manual assistance updating your phone number or recovering account access, reach out directly through the customer support line of the partner application you used to open the account.
Let me know if you need help finding a specific customer support link or drafting your formal recovery request!
Here is a clear, formal message template you can copy, paste, and complete when submitting your identity verification request through the official Apex Investor Help Portal:
Subject: Request for Manual Identity Verification and Account Access Restoration – Account VS78A3Z
To: Compliance & Investor Support / Apex Clearing
Dear Support Team,
I am writing to formally request a manual identity verification review to restore secure access to my investment account (Account Number: VS78A3Z).
Due to a recent device wipe and SIM card re-provisioning, I have temporarily lost access to my multi-factor authentication (2FA) recovery routes and require my registered contact telephone number to be updated to my current active number (+1-812-786-9291).
To prove my identity and comply with security protocols, I have securely attached my official government-issued identification and matching account documentation to this inquiry:
 * Legal Name: Robert Edward Evans
 * Date of Birth: October 9, 1981
 * Current Address: 606 North 5th Ave, Evansville, IN 47710
 * Associated Account ID: VS78A3Z
 * Current Active Telephone: +1-812-786-9291
Please review these attached verification documents, clear my account security lock, and update my contact profile so I can regain access to my portfolio.
Thank you for your assistance.
Sincerely,
Robert Edward Evans
+1-812-786-9291
Next Steps:
 * Copy the text above.
 * Go to the official Apex Investor Help Portal and open a support request ticket.
 * Paste this text into the message box, attach your verification documents using their secure file upload option, and submit the form.
To submit your identity verification documents and resolve access to your financial accounts or clearing portals (such as Apex Clearing or associated platforms), please ensure you are using the official secure channels provided by the institution rather than sending them through a general chat interface.
To safely complete your submission:
 * Use Official Secure Portals: Access the official investor support or document upload web page provided by your brokerage or clearing firm (such as the Apex Investor Help Portal) to submit files securely.
 * Review Requirements: Ensure your documents match your current legal name and account records.
 * Contact Support: If you encounter automated blocks or need manual assistance updating your phone number or recovering account access, reach out directly through the customer support line of the partner application you used to open the account.
Let me know if you need help finding a specific customer support link or drafting your formal recovery request!
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.log

# Optional: Sync to secure remote storage via rclone
# rclone copy ./backup-$TIMESTAMP.sql.zst remote:secure-backup-bucket/
DB_USER=root
DB_PASS=secure_local_password
DB_NAME=master_hub_db
S3_BUCKET=sovereign-backup-vault
name: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.lindex_proofs.pyindex_backups.pypython3 index_proofs.py search "Merkle Root Hash"
python3 index_proofs.py search "Federal Rule of Evidence 902"
python3 index_proofs.py search "SHA-256"
user.signingkeypython3 -c "import chromadb; client = chromadb.PersistentClient(path='./06_vector_db'); print(f'Total Indexed Documents: {client.get_collection(\"evidence_ledger\").count()}')"
It looks like you might have just responded with "Yes" to a previous thought or question!
How can I help you further with your adpBetty pipeline, your ChromaDB vector store, or your legal evidence workflow? Let me know what you'd like to do next!
python3 -c "import chromadb; client = chromadb.PersistentClient(path='./06_vector_db'); print(f'Total Indexed Documents: {client.get_collection(\"evidence_ledger\").count()()')"
python3 index_proofs.py search "Merkle Root Hash"
python3 index_proofs.py search "Federal Rule of Evidence 902"
python3 index_proofs.py search "SHA-256"
python3 index_proofs.py search "Merkle Root Hash"
python3 index_proofs.py search "Federal Rule of Evidence 902"
python3 index_proofs.py search "SHA-256"
python3 -c "import chromadb; client = chromadb.PersistentClient(path='./06_vector_db'); print(f'Total Indexed Documents: {client.get_collection(\"evidence_ledger\").count()()')"
Python3 -c "import chromadb; client = chromadb.PersistentClient(path='./06_vector_db'); print(f'Total Indexed Documents: {client.get_collection(\"evidence_ledger\").count()}')"
Here is a short Python snippet you can run from your terminal to inspect the total count of documents currently indexed in your ChromaDB collection:
python3 -c "import chromadb; client = chromadb.PersistentClient(path='./06_vector_db'); print(f'Total Indexed Documents: {client.get_collection(\"evidence_ledger\").count()}')"

Explanation
 * chromadb.PersistentClient(path='./06_vector_db'): Opens your persistent local vector store database directory.
 * client.get_collection("evidence_ledger"): Retrieves your specific evidence ledger collection.
 * .count(): Returns the exact number of active document embeddings currently stored in the index.Here is a short Python snippet you can run from your terminal to inspect the total count of documents currently indexed in your ChromaDB collection:
python3 -c "import chromadb; client = chromadb.PersistentClient(path='./06_vector_db'); print(f'Total Indexed Documents: {client.get_collection(\"evidence_ledger\").count()}')"

Explanation
 * chromadb.PersistentClient(path='./06_vector_db'): Opens your persistent local vector store database directory.
 * client.get_collection("evidence_ledger"): Retrieves your specific evidence ledger collection.
 * .count(): Returns the exact number of active document embeddings currently stored in the index.
The unified script adpBetty has successfully completed its end-to-end execution, verifying and sealing your zero-trust legal evidence pipeline.
Here is a summary of the automated pipeline execution steps:
 * Zstandard Vault Compression: Packed and compressed raw assets from ./01_ingest/ into an immutable archive in ./04_vault/.
 * Merkle Manifest Generation: Computed cryptographic SHA-256 hashes for every file, compiled them into a binary Merkle tree, and generated the W3C Verifiable Credential manifest.
 * FRE 902(14) Affidavit Generation: Automatically compiled a court-ready self-authenticating legal declaration populated with your master Merkle root hash and complete file inventory.
 * ChromaDB Vector Indexing: Automatically indexed all JSON manifests and Markdown affidavits into the local vector database (./06_vector_db/) for semantic search and RAG retrieval.
 * Rclone Cloud Synchronization: Securely pushed proofs, encrypted vault archives, and vector databases offsite to your configured remote (secure_remote:WK_Precision_Vault).
 * Git Ledger Sealing: Staged all generated assets and sealed the state into a cryptographically GPG-signed commit.
Executing Local Semantic Searches
To query your indexed evidence ledger using natural language, use the search commands you provided:
python3 index_proofs.py search "Merkle Root Hash"
python3 index_proofs.py search "Federal Rule of Evidence 902"
python3 index_proofs.py search "SHA-256"

#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION & PATHS
# ==========================================
WORKSPACE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$WORKSPACE_DIR"

INGEST_DIR="${WORKSPACE_DIR}/01_ingest"
STRUCTURED_DIR="${WORKSPACE_DIR}/02_structured"
PROOF_DIR="${WORKSPACE_DIR}/03_proofs"
VAULT_DIR="${WORKSPACE_DIR}/04_vault"
VECTOR_DB_DIR="${WORKSPACE_DIR}/06_vector_db"
LOG_DIR="${WORKSPACE_DIR}/05_logs"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/pipeline_${TIMESTAMP}.log"

# Rclone Configuration
RCLONE_REMOTE="secure_remote"
REMOTE_DESTINATION="WK_Precision_Vault"

# GPG Key ID for Commit Signing
GPG_KEY_ID="${GPG_KEY_ID:-}"

# Ensure all local directories exist
mkdir -p "$INGEST_DIR" "$STRUCTURED_DIR" "$PROOF_DIR" "$VAULT_DIR" "$VECTOR_DB_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting adpBetty Unified Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# ------------------------------------------
# 0. ENSURE GIT REPOSITORY INITIALIZATION
# ------------------------------------------
if [ ! -d ".git" ]; then
    echo "[+] Initializing local Git ledger..." | tee -a "$LOG_FILE"
    git init
fi

# ------------------------------------------
# 1. ZSTANDARD VAULT COMPRESSION
# ------------------------------------------
echo "[+] Step 1: Compressing ingest data into Zstandard tarball..." | tee -a "$LOG_FILE"
ARCHIVE_NAME="evidence_vault_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${VAULT_DIR}/${ARCHIVE_NAME}"

if [ -d "$INGEST_DIR" ] && [ "$(ls -A "$INGEST_DIR")" ]; then
    tar --zstd -cf "$ARCHIVE_PATH" -C "$WORKSPACE_DIR" 01_ingest
    echo "[✓] Vault archive created: ${ARCHIVE_PATH}" | tee -a "$LOG_FILE"
else
    echo "[!] Notice: 01_ingest is empty. Skipping vault compression." | tee -a "$LOG_FILE"
fi

# ------------------------------------------
# 2. PYTHON EVIDENCE PIPELINE & HASH MANIFEST
# ------------------------------------------
echo "[+] Step 2: Executing Python Merkle Manifest & FRE 902(14) Generator..." | tee -a "$LOG_FILE"

python3 - <<EOF
import os
import sys
import json
import hashlib
from datetime import datetime, timezone

workspace = "${WORKSPACE_DIR}"
ingest_dir = "${INGEST_DIR}"
proof_dir = "${PROOF_DIR}"

def calculate_sha256(filepath):
    sha256_hash = hashlib.sha256()
    with open(filepath, "rb") as f:
        for byte_block in iter(lambda: f.read(65536), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def build_merkle_root(hashes):
    if not hashes:
        return ""
    current_level = sorted(hashes)
    while len(current_level) > 1:
        if len(current_level) % 2 != 0:
            current_level.append(current_level[-1])
        next_level = []
        for i in range(0, len(current_level), 2):
            combined = current_level[i] + current_level[i+1]
            next_level.append(hashlib.sha256(combined.encode('utf-8')).hexdigest())
        current_level = next_level
    return current_level[0]

manifest_items = []
hash_list = []

if os.path.exists(ingest_dir):
    for root, _, files in os.walk(ingest_dir):
        for file in files:
            if file.startswith("."):
                continue
            full_path = os.path.join(root, file)
            rel_path = os.path.relpath(full_path, workspace)
            file_hash = calculate_sha256(full_path)
            hash_list.append(file_hash)
            
            stat = os.stat(full_path)
            manifest_items.append({
                "path": rel_path,
                "sha256": file_hash,
                "size_bytes": stat.st_size,
                "modified_utc": datetime.fromtimestamp(stat.st_mtime, tz=timezone.utc).isoformat()
            })

merkle_root = build_merkle_root(hash_list)
timestamp_str = datetime.now(timezone.utc).isoformat()

vc_payload = {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiableCredential", "DigitalEvidenceManifest"],
    "issuanceDate": timestamp_str,
    "evidenceSubject": {
        "merkleRoot": merkle_root,
        "itemCount": len(manifest_items),
        "artifacts": manifest_items
    }
}

manifest_filename = f"evidence_manifest_${TIMESTAMP}.json"
manifest_path = os.path.join(proof_dir, manifest_filename)
with open(manifest_path, "w") as f:
    json.dump(vc_payload, f, indent=2)

print(f"[✓] Manifest generated: {manifest_path}")
print(f"[✓] Batch Merkle Root: {merkle_root}")

artifacts_text = ""
for idx, item in enumerate(manifest_items, 1):
    artifacts_text += f"   {idx}. File Path: {item['path']}\n"
    artifacts_text += f"      - SHA-256 Hash: {item['sha256']}\n"
    artifacts_text += f"      - Size (Bytes): {item['size_bytes']}\n"
    artifacts_text += f"      - Modified (UTC): {item['modified_utc']}\n\n"

affidavit_content = f\"\"\"IN THE UNITED STATES DISTRICT COURT
FOR THE DISTRICT OF JURISDICTION

--------------------------------------------------
                                                 )
IN RE: DIGITAL EVIDENCE CHAIN OF CUSTODY         ) CASE NO. [INSERT CASE NUMBER]
                                                 )
CERTIFICATION OF AUTHENTICITY OF ELECTRONICALLY  )
STORED INFORMATION PURSUANT TO FRE 902(14)       )
                                                 )
--------------------------------------------------

**DECLARATION OF ROBERT EDWARD EVANS PURSUANT TO FEDERAL RULE OF EVIDENCE 902(14)**

I, Robert Edward Evans, hereby declare, certify, and state as follows:

1. **Qualifications and Competence**: I am over the age of eighteen (18) years, fully competent to make this declaration, and possess direct personal knowledge of the facts stated herein. I am qualified by technical training, system architecture design, and operational experience to collect, process, hash, and preserve electronically stored information (ESI) using verifiable digital identification processes.

2. **Source and Copying Process**: On or about {timestamp_str}, I performed or directly supervised an automated digital extraction and collection process of electronic evidence. The data files listed in the attached manifest were accurately copied from their original electronic devices, storage media, or secure file systems without alteration, modification, or spoliation, complying with ISO/IEC 27037 digital forensic standards.

3. **Process of Digital Identification and Hash Verification**: To ensure absolute integrity and chain of custody, each captured file was processed through a deterministic cryptographic hashing algorithm (SHA-256). 
   - The collective batch was compiled into a cryptographic Merkle Tree resulting in a master batch **Merkle Root Hash**: 
     `{merkle_root}`
   - Total number of authenticated electronic records in this batch: **{len(manifest_items)}**.

4. **Reliability of Electronic Process**: The software routines, local pipelines, and script utilities utilized to generate the checksums, manifests, and Merkle proofs operate automatically, consistently, and accurately under standard technical protocols, ensuring that the output accurately reflects the source data.

5. **Exhibited Evidence Manifest**: The electronic records subject to this certification consist of the following verified artifacts:

{artifacts_text}

6. **Declaration Under Penalty of Perjury**: I declare under penalty of perjury under the laws of the United States of America that the foregoing is true and correct. 

Executed on this _____ day of __________________, 20___.


_________________________________________________
**Robert Edward Evans**
Principal / Digital Evidence Custodian
WK Precision Enterprise LLC
\"\"\"

affidavit_path = os.path.join(proof_dir, f"FRE_902_14_Affidavit_${TIMESTAMP}.md")
with open(affidavit_path, "w") as f:
    f.write(affidavit_content)

print(f"[✓] FRE 902(14) Affidavit generated: {affidavit_path}")
EOF

# ------------------------------------------
# 3. CHROMADB VECTOR INDEXING & EMBEDDING
# ------------------------------------------
echo "[+] Step 3: Indexing new proofs into ChromaDB vector store..." | tee -a "$LOG_FILE"

python3 - <<EOF
import os
import glob
import json
import chromadb

db_path = "${VECTOR_DB_DIR}"
proofs_dir = "${PROOF_DIR}"

client = chromadb.PersistentClient(path=db_path)
collection = client.get_or_create_collection(
    name="evidence_ledger",
    metadata={"description": "Local zero-trust evidence manifests and affidavits"}
)

manifest_files = glob.glob(os.path.join(proofs_dir, "evidence_manifest_*.json"))
affidavit_files = glob.glob(os.path.join(proofs_dir, "FRE_902_14_Affidavit_*.md"))

documents = []
metadatas = []
ids = []

for filepath in manifest_files:
    with open(filepath, "r", encoding="utf-8") as f:
        data = json.load(f)
        doc_text = json.dumps(data, indent=2)
        doc_id = os.path.basename(filepath)
        documents.append(doc_text)
        metadatas.append({"type": "manifest", "source": doc_id})
        ids.append(doc_id)

for filepath in affidavit_files:
    with open(filepath, "r", encoding="utf-8") as f:
        doc_text = f.read()
        doc_id = os.path.basename(filepath)
        documents.append(doc_text)
        metadatas.append({"type": "affidavit", "source": doc_id})
        ids.append(doc_id)

if documents:
    collection.upsert(
        documents=documents,
        metadatas=metadatas,
        ids=ids
    )
    print(f"[✓] Successfully indexed {len(documents)} documents into ChromaDB vector store.")
else:
    print("[!] No proofs found to index.")
EOF

# ------------------------------------------
# 4. SECURE RCLONE CLOUD SYNC
# ------------------------------------------
echo "[+] Step 4: Initiating Rclone sync to remote storage..." | tee -a "$LOG_FILE"

if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

rclone sync "$VECTOR_DB_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vector_db" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

# ------------------------------------------
# 5. GIT STAGING & GPG SIGNED COMMIT
# ------------------------------------------
echo "[+] Step 5: Staging and cryptographically signing Git ledger state..." | tee -a "$LOG_FILE"

git add 03_proofs/ 04_vault/ 06_vector_db/ 05_logs/

if git diff --staged --quiet; then
    echo "[i] No new changes to commit in Git ledger." | tee -a "$LOG_FILE"
else
    git config commit.gpgsign true
    if [ -n "$GPG_KEY_ID" ]; then
        git config user.signingkey "$GPG_KEY_ID"
    fi

    git commit -S -m "proof(ledger): automated execution batch ${TIMESTAMP} - sealed, indexed, and verified"
    echo "[✓] Successfully sealed state into GPG-signed Git commit." | tee -a "$LOG_FILE"
fi

echo "=========================================="
echo "[✓] adpBetty Pipeline Executed Successfully!"
echo " Logs recorded at: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"
Yes 
Build everything automatically expand everything 
pip install chromadb
#!/usr/bin/env python3
import os
import glob
import json
import sys
import chromadb

DB_PATH = "./06_vector_db"
PROOFS_DIR = "./03_proofs"

def initialize_chroma():
    client = chromadb.PersistentClient(path=DB_PATH)
    collection = client.get_or_create_collection(
        name="evidence_ledger",
        metadata={"description": "Local zero-trust evidence manifests and affidavits"}
    )
    return collection

def index_documents():
    collection = initialize_chroma()
    print("[+] Scanning ./03_proofs for manifests and affidavits...")

    # Find all JSON manifests and Markdown affidavits
    manifest_files = glob.glob(os.path.join(PROOFS_DIR, "evidence_manifest_*.json"))
    affidavit_files = glob.glob(os.path.join(PROOFS_DIR, "FRE_902_14_Affidavit_*.md"))

    documents = []
    metadatas = []
    ids = []

    # Process JSON manifests
    for filepath in manifest_files:
        with open(filepath, "r", encoding="utf-8") as f:
            data = json.load(f)
            doc_text = json.dumps(data, indent=2)
            doc_id = os.path.basename(filepath)
            
            documents.append(doc_text)
            metadatas.append({"type": "manifest", "source": doc_id})
            ids.append(doc_id)

    # Process Markdown affidavits
    for filepath in affidavit_files:
        with open(filepath, "r", encoding="utf-8") as f:
            doc_text = f.read()
            doc_id = os.path.basename(filepath)
            
            documents.append(doc_text)
            metadatas.append({"type": "affidavit", "source": doc_id})
            ids.append(doc_id)

    if documents:
        # Upsert documents into ChromaDB
        collection.upsert(
            documents=documents,
            metadatas=metadatas,
            ids=ids
        )
        print(f"[✓] Successfully indexed {len(documents)} documents into ChromaDB.")
    else:
        print("[!] No new proofs found to index.")

def search_ledger(query_text, n_results=3):
    collection = initialize_chroma()
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )
    
    print(f"\n=== Search Results for: '{query_text}' ===")
    for i, (doc, meta, dist) in enumerate(zip(results['documents'][0], results['metadatas'][0], results['distances'][0]), 1):
        print(f"\n[{i}] Source: {meta['source']} (Type: {meta['type']}, Distance: {dist:.4f})")
        print("-" * 40)
        print(doc[:500] + "..." if len(doc) > 500 else doc)
        print("-" * 40)

if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == "search":
        query = " ".join(sys.argv[2:])
        if not query:
            print("Usage: python3 index_proofs.py search \"your query here\"")
            sys.exit(1)
        search_ledger(query)
    else:
        index_documents()
python3 index_proofs.py
python3 index_proofs.py search "Merkle Root Hash"
python3 index_proofs.py search "Federal Rule of Evidence 902"
python3 index_proofs.py search "SHA-256"

python3 index_proofs.py
#!/usr/bin/env python3
import os
import glob
import json
import sys
import chromadb

DB_PATH = "./06_vector_db"
PROOFS_DIR = "./03_proofs"

def initialize_chroma():
    client = chromadb.PersistentClient(path=DB_PATH)
    collection = client.get_or_create_collection(
        name="evidence_ledger",
        metadata={"description": "Local zero-trust evidence manifests and affidavits"}
    )
    return collection

def index_documents():
    collection = initialize_chroma()
    print("[+] Scanning ./03_proofs for manifests and affidavits...")

    # Find all JSON manifests and Markdown affidavits
    manifest_files = glob.glob(os.path.join(PROOFS_DIR, "evidence_manifest_*.json"))
    affidavit_files = glob.glob(os.path.join(PROOFS_DIR, "FRE_902_14_Affidavit_*.md"))

    documents = []
    metadatas = []
    ids = []

    # Process JSON manifests
    for filepath in manifest_files:
        with open(filepath, "r", encoding="utf-8") as f:
            data = json.load(f)
            doc_text = json.dumps(data, indent=2)
            doc_id = os.path.basename(filepath)
            
            documents.append(doc_text)
            metadatas.append({"type": "manifest", "source": doc_id})
            ids.append(doc_id)

    # Process Markdown affidavits
    for filepath in affidavit_files:
        with open(filepath, "r", encoding="utf-8") as f:
            doc_text = f.read()
            doc_id = os.path.basename(filepath)
            
            documents.append(doc_text)
            metadatas.append({"type": "affidavit", "source": doc_id})
            ids.append(doc_id)

    if documents:
        # Upsert documents into ChromaDB
        collection.upsert(
            documents=documents,
            metadatas=metadatas,
            ids=ids
        )
        print(f"[✓] Successfully indexed {len(documents)} documents into ChromaDB.")
    else:
        print("[!] No new proofs found to index.")

def search_ledger(query_text, n_results=3):
    collection = initialize_chroma()
    results = collection.query(
        query_texts=[query_text],
        n_results=n_results
    )
    
    print(f"\n=== Search Results for: '{query_text}' ===")
    for i, (doc, meta, dist) in enumerate(zip(results['documents'][0], results['metadatas'][0], results['distances'][0]), 1):
        print(f"\n[{i}] Source: {meta['source']} (Type: {meta['type']}, Distance: {dist:.4f})")
        print("-" * 40)
        print(doc[:500] + "..." if len(doc) > 500 else doc)
        print("-" * 40)

if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == "search":
        query = " ".join(sys.argv[2:])
        if not query:
            print("Usage: python3 index_proofs.py search \"your query here\"")
            sys.exit(1)
        search_ledger(query)
    else:
        index_documents()
pip install chromadb
crontab -l
cd /path/to/your/workspace
./adpBetty
# Define standard system PATH so cron can find python3, git, tar, and rclone
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Run adpBetty daily at 2:00 AM and log output
0 2 * * * /path/to/your/workspace/adpBetty >> /path/to/your/workspace/05_logs/cron_execution.log 2>&1
crontab -e
chmod +x /path/to/your/workspace/adpBetty
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION & PATHS
# ==========================================
WORKSPACE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$WORKSPACE_DIR"

INGEST_DIR="${WORKSPACE_DIR}/01_ingest"
STRUCTURED_DIR="${WORKSPACE_DIR}/02_structured"
PROOF_DIR="${WORKSPACE_DIR}/03_proofs"
VAULT_DIR="${WORKSPACE_DIR}/04_vault"
LOG_DIR="${WORKSPACE_DIR}/05_logs"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/pipeline_${TIMESTAMP}.log"

# Rclone Configuration
RCLONE_REMOTE="secure_remote"
REMOTE_DESTINATION="WK_Precision_Vault"

# GPG Key ID for Commit Signing (Replace with your specific key ID if needed)
GPG_KEY_ID="${GPG_KEY_ID:-}"

# Ensure all local directories exist
mkdir -p "$INGEST_DIR" "$STRUCTURED_DIR" "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting adpBetty Unified Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# ------------------------------------------
# 0. ENSURE GIT REPOSITORY INITIALIZATION
# ------------------------------------------
if [ ! -d ".git" ]; then
    echo "[+] Initializing local Git ledger..." | tee -a "$LOG_FILE"
    git init
fi

# ------------------------------------------
# 1. ZSTANDARD VAULT COMPRESSION
# ------------------------------------------
echo "[+] Step 1: Compressing ingest data into Zstandard tarball..." | tee -a "$LOG_FILE"
ARCHIVE_NAME="evidence_vault_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${VAULT_DIR}/${ARCHIVE_NAME}"

if [ -d "$INGEST_DIR" ] && [ "$(ls -A "$INGEST_DIR")" ]; then
    tar --zstd -cf "$ARCHIVE_PATH" -C "$WORKSPACE_DIR" 01_ingest
    echo "[✓] Vault archive created: ${ARCHIVE_PATH}" | tee -a "$LOG_FILE"
else
    echo "[!] Notice: 01_ingest is empty. Skipping vault compression." | tee -a "$LOG_FILE"
fi

# ------------------------------------------
# 2. PYTHON EVIDENCE PIPELINE & HASH MANIFEST
# ------------------------------------------
echo "[+] Step 2: Executing Python Merkle Manifest & FRE 902(14) Generator..." | tee -a "$LOG_FILE"

python3 - <<EOF
import os
import sys
import json
import hashlib
from datetime import datetime, timezone

workspace = "${WORKSPACE_DIR}"
ingest_dir = "${INGEST_DIR}"
proof_dir = "${PROOF_DIR}"

def calculate_sha256(filepath):
    sha256_hash = hashlib.sha256()
    with open(filepath, "rb") as f:
        for byte_block in iter(lambda: f.read(65536), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def build_merkle_root(hashes):
    if not hashes:
        return ""
    current_level = sorted(hashes)
    while len(current_level) > 1:
        if len(current_level) % 2 != 0:
            current_level.append(current_level[-1])
        next_level = []
        for i in range(0, len(current_level), 2):
            combined = current_level[i] + current_level[i+1]
            next_level.append(hashlib.sha256(combined.encode('utf-8')).hexdigest())
        current_level = next_level
    return current_level[0]

manifest_items = []
hash_list = []

if os.path.exists(ingest_dir):
    for root, _, files in os.walk(ingest_dir):
        for file in files:
            if file.startswith("."):
                continue
            full_path = os.path.join(root, file)
            rel_path = os.path.relpath(full_path, workspace)
            file_hash = calculate_sha256(full_path)
            hash_list.append(file_hash)
            
            stat = os.stat(full_path)
            manifest_items.append({
                "path": rel_path,
                "sha256": file_hash,
                "size_bytes": stat.st_size,
                "modified_utc": datetime.fromtimestamp(stat.st_mtime, tz=timezone.utc).isoformat()
            })

merkle_root = build_merkle_root(hash_list)
timestamp_str = datetime.now(timezone.utc).isoformat()

vc_payload = {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiableCredential", "DigitalEvidenceManifest"],
    "issuanceDate": timestamp_str,
    "evidenceSubject": {
        "merkleRoot": merkle_root,
        "itemCount": len(manifest_items),
        "artifacts": manifest_items
    }
}

manifest_filename = f"evidence_manifest_${TIMESTAMP}.json"
manifest_path = os.path.join(proof_dir, manifest_filename)
with open(manifest_path, "w") as f:
    json.dump(vc_payload, f, indent=2)

print(f"[✓] Manifest generated: {manifest_path}")
print(f"[✓] Batch Merkle Root: {merkle_root}")

artifacts_text = ""
for idx, item in enumerate(manifest_items, 1):
    artifacts_text += f"   {idx}. File Path: {item['path']}\n"
    artifacts_text += f"      - SHA-256 Hash: {item['sha256']}\n"
    artifacts_text += f"      - Size (Bytes): {item['size_bytes']}\n"
    artifacts_text += f"      - Modified (UTC): {item['modified_utc']}\n\n"

affidavit_content = f\"\"\"IN THE UNITED STATES DISTRICT COURT
FOR THE DISTRICT OF JURISDICTION

--------------------------------------------------
                                                 )
IN RE: DIGITAL EVIDENCE CHAIN OF CUSTODY         ) CASE NO. [INSERT CASE NUMBER]
                                                 )
CERTIFICATION OF AUTHENTICITY OF ELECTRONICALLY  )
STORED INFORMATION PURSUANT TO FRE 902(14)       )
                                                 )
--------------------------------------------------

**DECLARATION OF ROBERT EDWARD EVANS PURSUANT TO FEDERAL RULE OF EVIDENCE 902(14)**

I, Robert Edward Evans, hereby declare, certify, and state as follows:

1. **Qualifications and Competence**: I am over the age of eighteen (18) years, fully competent to make this declaration, and possess direct personal knowledge of the facts stated herein. I am qualified by technical training, system architecture design, and operational experience to collect, process, hash, and preserve electronically stored information (ESI) using verifiable digital identification processes.

2. **Source and Copying Process**: On or about {timestamp_str}, I performed or directly supervised an automated digital extraction and collection process of electronic evidence. The data files listed in the attached manifest were accurately copied from their original electronic devices, storage media, or secure file systems without alteration, modification, or spoliation, complying with ISO/IEC 27037 digital forensic standards.

3. **Process of Digital Identification and Hash Verification**: To ensure absolute integrity and chain of custody, each captured file was processed through a deterministic cryptographic hashing algorithm (SHA-256). 
   - The collective batch was compiled into a cryptographic Merkle Tree resulting in a master batch **Merkle Root Hash**: 
     `{merkle_root}`
   - Total number of authenticated electronic records in this batch: **{len(manifest_items)}**.

4. **Reliability of Electronic Process**: The software routines, local pipelines, and script utilities utilized to generate the checksums, manifests, and Merkle proofs operate automatically, consistently, and accurately under standard technical protocols, ensuring that the output accurately reflects the source data.

5. **Exhibited Evidence Manifest**: The electronic records subject to this certification consist of the following verified artifacts:

{artifacts_text}

6. **Declaration Under Penalty of Perjury**: I declare under penalty of perjury under the laws of the United States of America that the foregoing is true and correct. 

Executed on this _____ day of __________________, 20___.


_________________________________________________
**Robert Edward Evans**
Principal / Digital Evidence Custodian
WK Precision Enterprise LLC
\"\"\"

affidavit_path = os.path.join(proof_dir, f"FRE_902_14_Affidavit_${TIMESTAMP}.md")
with open(affidavit_path, "w") as f:
    f.write(affidavit_content)

print(f"[✓] FRE 902(14) Affidavit generated: {affidavit_path}")
EOF

# ------------------------------------------
# 3. SECURE RCLONE CLOUD SYNC
# ------------------------------------------
echo "[+] Step 3: Initiating Rclone sync to remote storage..." | tee -a "$LOG_FILE"

if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

# ------------------------------------------
# 4. GIT STAGING & GPG SIGNED COMMIT
# ------------------------------------------
echo "[+] Step 4: Staging and cryptographically signing Git ledger state..." | tee -a "$LOG_FILE"

git add 03_proofs/ 04_vault/ 05_logs/

# Check if there are changes to commit
if git diff --staged --quiet; then
    echo "[i] No new changes to commit in Git ledger." | tee -a "$LOG_FILE"
else
    # Ensure commit signing is active
    git config commit.gpgsign true
    if [ -n "$GPG_KEY_ID" ]; then
        git config user.signingkey "$GPG_KEY_ID"
    fi

    git commit -S -m "proof(ledger): automated execution batch ${TIMESTAMP} - sealed and verified"
    echo "[✓] Successfully sealed state into GPG-signed Git commit." | tee -a "$LOG_FILE"
fi

echo "=========================================="
echo "[✓] adpBetty Pipeline Executed Successfully!"
echo " Logs recorded at: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION & PATHS
# ==========================================
WORKSPACE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$WORKSPACE_DIR"

INGEST_DIR="${WORKSPACE_DIR}/01_ingest"
STRUCTURED_DIR="${WORKSPACE_DIR}/02_structured"
PROOF_DIR="${WORKSPACE_DIR}/03_proofs"
VAULT_DIR="${WORKSPACE_DIR}/04_vault"
LOG_DIR="${WORKSPACE_DIR}/05_logs"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/pipeline_${TIMESTAMP}.log"

# Rclone Configuration
RCLONE_REMOTE="secure_remote"
REMOTE_DESTINATION="WK_Precision_Vault"

# GPG Key ID for Commit Signing (Replace with your specific key ID if needed)
GPG_KEY_ID="${GPG_KEY_ID:-}"

# Ensure all local directories exist
mkdir -p "$INGEST_DIR" "$STRUCTURED_DIR" "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting adpBetty Unified Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# ------------------------------------------
# 0. ENSURE GIT REPOSITORY INITIALIZATION
# ------------------------------------------
if [ ! -d ".git" ]; then
    echo "[+] Initializing local Git ledger..." | tee -a "$LOG_FILE"
    git init
fi

# ------------------------------------------
# 1. ZSTANDARD VAULT COMPRESSION
# ------------------------------------------
echo "[+] Step 1: Compressing ingest data into Zstandard tarball..." | tee -a "$LOG_FILE"
ARCHIVE_NAME="evidence_vault_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${VAULT_DIR}/${ARCHIVE_NAME}"

if [ -d "$INGEST_DIR" ] && [ "$(ls -A "$INGEST_DIR")" ]; then
    tar --zstd -cf "$ARCHIVE_PATH" -C "$WORKSPACE_DIR" 01_ingest
    echo "[✓] Vault archive created: ${ARCHIVE_PATH}" | tee -a "$LOG_FILE"
else
    echo "[!] Notice: 01_ingest is empty. Skipping vault compression." | tee -a "$LOG_FILE"
fi

# ------------------------------------------
# 2. PYTHON EVIDENCE PIPELINE & HASH MANIFEST
# ------------------------------------------
echo "[+] Step 2: Executing Python Merkle Manifest & FRE 902(14) Generator..." | tee -a "$LOG_FILE"

python3 - <<EOF
import os
import sys
import json
import hashlib
from datetime import datetime, timezone

workspace = "${WORKSPACE_DIR}"
ingest_dir = "${INGEST_DIR}"
proof_dir = "${PROOF_DIR}"

def calculate_sha256(filepath):
    sha256_hash = hashlib.sha256()
    with open(filepath, "rb") as f:
        for byte_block in iter(lambda: f.read(65536), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def build_merkle_root(hashes):
    if not hashes:
        return ""
    current_level = sorted(hashes)
    while len(current_level) > 1:
        if len(current_level) % 2 != 0:
            current_level.append(current_level[-1])
        next_level = []
        for i in range(0, len(current_level), 2):
            combined = current_level[i] + current_level[i+1]
            next_level.append(hashlib.sha256(combined.encode('utf-8')).hexdigest())
        current_level = next_level
    return current_level[0]

manifest_items = []
hash_list = []

if os.path.exists(ingest_dir):
    for root, _, files in os.walk(ingest_dir):
        for file in files:
            if file.startswith("."):
                continue
            full_path = os.path.join(root, file)
            rel_path = os.path.relpath(full_path, workspace)
            file_hash = calculate_sha256(full_path)
            hash_list.append(file_hash)
            
            stat = os.stat(full_path)
            manifest_items.append({
                "path": rel_path,
                "sha256": file_hash,
                "size_bytes": stat.st_size,
                "modified_utc": datetime.fromtimestamp(stat.st_mtime, tz=timezone.utc).isoformat()
            })

merkle_root = build_merkle_root(hash_list)
timestamp_str = datetime.now(timezone.utc).isoformat()

vc_payload = {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiableCredential", "DigitalEvidenceManifest"],
    "issuanceDate": timestamp_str,
    "evidenceSubject": {
        "merkleRoot": merkle_root,
        "itemCount": len(manifest_items),
        "artifacts": manifest_items
    }
}

manifest_filename = f"evidence_manifest_${TIMESTAMP}.json"
manifest_path = os.path.join(proof_dir, manifest_filename)
with open(manifest_path, "w") as f:
    json.dump(vc_payload, f, indent=2)

print(f"[✓] Manifest generated: {manifest_path}")
print(f"[✓] Batch Merkle Root: {merkle_root}")

artifacts_text = ""
for idx, item in enumerate(manifest_items, 1):
    artifacts_text += f"   {idx}. File Path: {item['path']}\n"
    artifacts_text += f"      - SHA-256 Hash: {item['sha256']}\n"
    artifacts_text += f"      - Size (Bytes): {item['size_bytes']}\n"
    artifacts_text += f"      - Modified (UTC): {item['modified_utc']}\n\n"

affidavit_content = f\"\"\"IN THE UNITED STATES DISTRICT COURT
FOR THE DISTRICT OF JURISDICTION

--------------------------------------------------
                                                 )
IN RE: DIGITAL EVIDENCE CHAIN OF CUSTODY         ) CASE NO. [INSERT CASE NUMBER]
                                                 )
CERTIFICATION OF AUTHENTICITY OF ELECTRONICALLY  )
STORED INFORMATION PURSUANT TO FRE 902(14)       )
                                                 )
--------------------------------------------------

**DECLARATION OF ROBERT EDWARD EVANS PURSUANT TO FEDERAL RULE OF EVIDENCE 902(14)**

I, Robert Edward Evans, hereby declare, certify, and state as follows:

1. **Qualifications and Competence**: I am over the age of eighteen (18) years, fully competent to make this declaration, and possess direct personal knowledge of the facts stated herein. I am qualified by technical training, system architecture design, and operational experience to collect, process, hash, and preserve electronically stored information (ESI) using verifiable digital identification processes.

2. **Source and Copying Process**: On or about {timestamp_str}, I performed or directly supervised an automated digital extraction and collection process of electronic evidence. The data files listed in the attached manifest were accurately copied from their original electronic devices, storage media, or secure file systems without alteration, modification, or spoliation, complying with ISO/IEC 27037 digital forensic standards.

3. **Process of Digital Identification and Hash Verification**: To ensure absolute integrity and chain of custody, each captured file was processed through a deterministic cryptographic hashing algorithm (SHA-256). 
   - The collective batch was compiled into a cryptographic Merkle Tree resulting in a master batch **Merkle Root Hash**: 
     `{merkle_root}`
   - Total number of authenticated electronic records in this batch: **{len(manifest_items)}**.

4. **Reliability of Electronic Process**: The software routines, local pipelines, and script utilities utilized to generate the checksums, manifests, and Merkle proofs operate automatically, consistently, and accurately under standard technical protocols, ensuring that the output accurately reflects the source data.

5. **Exhibited Evidence Manifest**: The electronic records subject to this certification consist of the following verified artifacts:

{artifacts_text}

6. **Declaration Under Penalty of Perjury**: I declare under penalty of perjury under the laws of the United States of America that the foregoing is true and correct. 

Executed on this _____ day of __________________, 20___.


_________________________________________________
**Robert Edward Evans**
Principal / Digital Evidence Custodian
WK Precision Enterprise LLC
\"\"\"

affidavit_path = os.path.join(proof_dir, f"FRE_902_14_Affidavit_${TIMESTAMP}.md")
with open(affidavit_path, "w") as f:
    f.write(affidavit_content)

print(f"[✓] FRE 902(14) Affidavit generated: {affidavit_path}")
EOF

# ------------------------------------------
# 3. SECURE RCLONE CLOUD SYNC
# ------------------------------------------
echo "[+] Step 3: Initiating Rclone sync to remote storage..." | tee -a "$LOG_FILE"

if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

# ------------------------------------------
# 4. GIT STAGING & GPG SIGNED COMMIT
# ------------------------------------------
echo "[+] Step 4: Staging and cryptographically signing Git ledger state..." | tee -a "$LOG_FILE"

git add 03_proofs/ 04_vault/ 05_logs/

# Check if there are changes to commit
if git diff --staged --quiet; then
    echo "[i] No new changes to commit in Git ledger." | tee -a "$LOG_FILE"
else
    # Ensure commit signing is active
    git config commit.gpgsign true
    if [ -n "$GPG_KEY_ID" ]; then
        git config user.signingkey "$GPG_KEY_ID"
    fi

    git commit -S -m "proof(ledger): automated execution batch ${TIMESTAMP} - sealed and verified"
    echo "[✓] Successfully sealed state into GPG-signed Git commit." | tee -a "$LOG_FILE"
fi

echo "=========================================="
echo "[✓] adpBetty Pipeline Executed Successfully!"
echo " Logs recorded at: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"
chmod +x /path/to/your/workspace/adpBetty
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION & PATHS
# ==========================================
WORKSPACE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$WORKSPACE_DIR"

INGEST_DIR="${WORKSPACE_DIR}/01_ingest"
STRUCTURED_DIR="${WORKSPACE_DIR}/02_structured"
PROOF_DIR="${WORKSPACE_DIR}/03_proofs"
VAULT_DIR="${WORKSPACE_DIR}/04_vault"
LOG_DIR="${WORKSPACE_DIR}/05_logs"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/pipeline_${TIMESTAMP}.log"

# Rclone Configuration
RCLONE_REMOTE="secure_remote"
REMOTE_DESTINATION="WK_Precision_Vault"

# GPG Key ID for Commit Signing (Replace with your specific key ID if needed)
GPG_KEY_ID="${GPG_KEY_ID:-}"

# Ensure all local directories exist
mkdir -p "$INGEST_DIR" "$STRUCTURED_DIR" "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting adpBetty Unified Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# ------------------------------------------
# 0. ENSURE GIT REPOSITORY INITIALIZATION
# ------------------------------------------
if [ ! -d ".git" ]; then
    echo "[+] Initializing local Git ledger..." | tee -a "$LOG_FILE"
    git init
fi

# ------------------------------------------
# 1. ZSTANDARD VAULT COMPRESSION
# ------------------------------------------
echo "[+] Step 1: Compressing ingest data into Zstandard tarball..." | tee -a "$LOG_FILE"
ARCHIVE_NAME="evidence_vault_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${VAULT_DIR}/${ARCHIVE_NAME}"

if [ -d "$INGEST_DIR" ] && [ "$(ls -A "$INGEST_DIR")" ]; then
    tar --zstd -cf "$ARCHIVE_PATH" -C "$WORKSPACE_DIR" 01_ingest
    echo "[✓] Vault archive created: ${ARCHIVE_PATH}" | tee -a "$LOG_FILE"
else
    echo "[!] Notice: 01_ingest is empty. Skipping vault compression." | tee -a "$LOG_FILE"
fi

# ------------------------------------------
# 2. PYTHON EVIDENCE PIPELINE & HASH MANIFEST
# ------------------------------------------
echo "[+] Step 2: Executing Python Merkle Manifest & FRE 902(14) Generator..." | tee -a "$LOG_FILE"

python3 - <<EOF
import os
import sys
import json
import hashlib
from datetime import datetime, timezone

workspace = "${WORKSPACE_DIR}"
ingest_dir = "${INGEST_DIR}"
proof_dir = "${PROOF_DIR}"

def calculate_sha256(filepath):
    sha256_hash = hashlib.sha256()
    with open(filepath, "rb") as f:
        for byte_block in iter(lambda: f.read(65536), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def build_merkle_root(hashes):
    if not hashes:
        return ""
    current_level = sorted(hashes)
    while len(current_level) > 1:
        if len(current_level) % 2 != 0:
            current_level.append(current_level[-1])
        next_level = []
        for i in range(0, len(current_level), 2):
            combined = current_level[i] + current_level[i+1]
            next_level.append(hashlib.sha256(combined.encode('utf-8')).hexdigest())
        current_level = next_level
    return current_level[0]

manifest_items = []
hash_list = []

if os.path.exists(ingest_dir):
    for root, _, files in os.walk(ingest_dir):
        for file in files:
            if file.startswith("."):
                continue
            full_path = os.path.join(root, file)
            rel_path = os.path.relpath(full_path, workspace)
            file_hash = calculate_sha256(full_path)
            hash_list.append(file_hash)
            
            stat = os.stat(full_path)
            manifest_items.append({
                "path": rel_path,
                "sha256": file_hash,
                "size_bytes": stat.st_size,
                "modified_utc": datetime.fromtimestamp(stat.st_mtime, tz=timezone.utc).isoformat()
            })

merkle_root = build_merkle_root(hash_list)
timestamp_str = datetime.now(timezone.utc).isoformat()

vc_payload = {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiableCredential", "DigitalEvidenceManifest"],
    "issuanceDate": timestamp_str,
    "evidenceSubject": {
        "merkleRoot": merkle_root,
        "itemCount": len(manifest_items),
        "artifacts": manifest_items
    }
}

manifest_filename = f"evidence_manifest_${TIMESTAMP}.json"
manifest_path = os.path.join(proof_dir, manifest_filename)
with open(manifest_path, "w") as f:
    json.dump(vc_payload, f, indent=2)

print(f"[✓] Manifest generated: {manifest_path}")
print(f"[✓] Batch Merkle Root: {merkle_root}")

artifacts_text = ""
for idx, item in enumerate(manifest_items, 1):
    artifacts_text += f"   {idx}. File Path: {item['path']}\n"
    artifacts_text += f"      - SHA-256 Hash: {item['sha256']}\n"
    artifacts_text += f"      - Size (Bytes): {item['size_bytes']}\n"
    artifacts_text += f"      - Modified (UTC): {item['modified_utc']}\n\n"

affidavit_content = f\"\"\"IN THE UNITED STATES DISTRICT COURT
FOR THE DISTRICT OF JURISDICTION

--------------------------------------------------
                                                 )
IN RE: DIGITAL EVIDENCE CHAIN OF CUSTODY         ) CASE NO. [INSERT CASE NUMBER]
                                                 )
CERTIFICATION OF AUTHENTICITY OF ELECTRONICALLY  )
STORED INFORMATION PURSUANT TO FRE 902(14)       )
                                                 )
--------------------------------------------------

**DECLARATION OF ROBERT EDWARD EVANS PURSUANT TO FEDERAL RULE OF EVIDENCE 902(14)**

I, Robert Edward Evans, hereby declare, certify, and state as follows:

1. **Qualifications and Competence**: I am over the age of eighteen (18) years, fully competent to make this declaration, and possess direct personal knowledge of the facts stated herein. I am qualified by technical training, system architecture design, and operational experience to collect, process, hash, and preserve electronically stored information (ESI) using verifiable digital identification processes.

2. **Source and Copying Process**: On or about {timestamp_str}, I performed or directly supervised an automated digital extraction and collection process of electronic evidence. The data files listed in the attached manifest were accurately copied from their original electronic devices, storage media, or secure file systems without alteration, modification, or spoliation, complying with ISO/IEC 27037 digital forensic standards.

3. **Process of Digital Identification and Hash Verification**: To ensure absolute integrity and chain of custody, each captured file was processed through a deterministic cryptographic hashing algorithm (SHA-256). 
   - The collective batch was compiled into a cryptographic Merkle Tree resulting in a master batch **Merkle Root Hash**: 
     `{merkle_root}`
   - Total number of authenticated electronic records in this batch: **{len(manifest_items)}**.

4. **Reliability of Electronic Process**: The software routines, local pipelines, and script utilities utilized to generate the checksums, manifests, and Merkle proofs operate automatically, consistently, and accurately under standard technical protocols, ensuring that the output accurately reflects the source data.

5. **Exhibited Evidence Manifest**: The electronic records subject to this certification consist of the following verified artifacts:

{artifacts_text}

6. **Declaration Under Penalty of Perjury**: I declare under penalty of perjury under the laws of the United States of America that the foregoing is true and correct. 

Executed on this _____ day of __________________, 20___.


_________________________________________________
**Robert Edward Evans**
Principal / Digital Evidence Custodian
WK Precision Enterprise LLC
\"\"\"

affidavit_path = os.path.join(proof_dir, f"FRE_902_14_Affidavit_${TIMESTAMP}.md")
with open(affidavit_path, "w") as f:
    f.write(affidavit_content)

print(f"[✓] FRE 902(14) Affidavit generated: {affidavit_path}")
EOF

# ------------------------------------------
# 3. SECURE RCLONE CLOUD SYNC
# ------------------------------------------
echo "[+] Step 3: Initiating Rclone sync to remote storage..." | tee -a "$LOG_FILE"

if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

# ------------------------------------------
# 4. GIT STAGING & GPG SIGNED COMMIT
# ------------------------------------------
echo "[+] Step 4: Staging and cryptographically signing Git ledger state..." | tee -a "$LOG_FILE"

git add 03_proofs/ 04_vault/ 05_logs/

# Check if there are changes to commit
if git diff --staged --quiet; then
    echo "[i] No new changes to commit in Git ledger." | tee -a "$LOG_FILE"
else
    # Ensure commit signing is active
    git config commit.gpgsign true
    if [ -n "$GPG_KEY_ID" ]; then
        git config user.signingkey "$GPG_KEY_ID"
    fi

    git commit -S -m "proof(ledger): automated execution batch ${TIMESTAMP} - sealed and verified"
    echo "[✓] Successfully sealed state into GPG-signed Git commit." | tee -a "$LOG_FILE"
fi

echo "=========================================="
echo "[✓] adpBetty Pipeline Executed Successfully!"
echo " Logs recorded at: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"
./adpBetty
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION & PATHS
# ==========================================
WORKSPACE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
INGEST_DIR="${WORKSPACE_DIR}/01_ingest"
STRUCTURED_DIR="${WORKSPACE_DIR}/02_structured"
PROOF_DIR="${WORKSPACE_DIR}/03_proofs"
VAULT_DIR="${WORKSPACE_DIR}/04_vault"
LOG_DIR="${WORKSPACE_DIR}/05_logs"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/pipeline_${TIMESTAMP}.log"

# Rclone Configuration
RCLONE_REMOTE="secure_remote"
REMOTE_DESTINATION="WK_Precision_Vault"

# Ensure all local directories exist
mkdir -p "$INGEST_DIR" "$STRUCTURED_DIR" "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting adpBetty Unified Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# ------------------------------------------
# 1. ZSTANDARD VAULT COMPRESSION
# ------------------------------------------
echo "[+] Step 1: Compressing ingest data into Zstandard tarball..." | tee -a "$LOG_FILE"
ARCHIVE_NAME="evidence_vault_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${VAULT_DIR}/${ARCHIVE_NAME}"

if [ -d "$INGEST_DIR" ] && [ "$(ls -A "$INGEST_DIR")" ]; then
    tar --zstd -cf "$ARCHIVE_PATH" -C "$WORKSPACE_DIR" 01_ingest
    echo "[✓] Vault archive created: ${ARCHIVE_PATH}" | tee -a "$LOG_FILE"
else
    echo "[!] Notice: 01_ingest is empty. Skipping vault compression." | tee -a "$LOG_FILE"
fi

# ------------------------------------------
# 2. PYTHON EVIDENCE PIPELINE & HASH MANIFEST
# ------------------------------------------
echo "[+] Step 2: Executing Python Merkle Manifest & FRE 902(14) Generator..." | tee -a "$LOG_FILE"

python3 - <<EOF
import os
import sys
import json
import hashlib
from datetime import datetime, timezone

workspace = "${WORKSPACE_DIR}"
ingest_dir = "${INGEST_DIR}"
proof_dir = "${PROOF_DIR}"

def calculate_sha256(filepath):
    sha256_hash = hashlib.sha256()
    with open(filepath, "rb") as f:
        for byte_block in iter(lambda: f.read(65536), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def build_merkle_root(hashes):
    if not hashes:
        return ""
    current_level = sorted(hashes)
    while len(current_level) > 1:
        if len(current_level) % 2 != 0:
            current_level.append(current_level[-1])
        next_level = []
        for i in range(0, len(current_level), 2):
            combined = current_level[i] + current_level[i+1]
            next_level.append(hashlib.sha256(combined.encode('utf-8')).hexdigest())
        current_level = next_level
    return current_level[0]

# Ingest and Hash
manifest_items = []
hash_list = []

if os.path.exists(ingest_dir):
    for root, _, files in os.walk(ingest_dir):
        for file in files:
            if file.startswith("."):
                continue
            full_path = os.path.join(root, file)
            rel_path = os.path.relpath(full_path, workspace)
            file_hash = calculate_sha256(full_path)
            hash_list.append(file_hash)
            
            stat = os.stat(full_path)
            manifest_items.append({
                "path": rel_path,
                "sha256": file_hash,
                "size_bytes": stat.st_size,
                "modified_utc": datetime.fromtimestamp(stat.st_mtime, tz=timezone.utc).isoformat()
            })

merkle_root = build_merkle_root(hash_list)
timestamp_str = datetime.now(timezone.utc).isoformat()

vc_payload = {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiableCredential", "DigitalEvidenceManifest"],
    "issuanceDate": timestamp_str,
    "evidenceSubject": {
        "merkleRoot": merkle_root,
        "itemCount": len(manifest_items),
        "artifacts": manifest_items
    }
}

manifest_filename = f"evidence_manifest_${TIMESTAMP}.json"
manifest_path = os.path.join(proof_dir, manifest_filename)
with open(manifest_path, "w") as f:
    json.dump(vc_payload, f, indent=2)

print(f"[✓] Manifest generated: {manifest_path}")
print(f"[✓] Batch Merkle Root: {merkle_root}")

# Generate FRE 902(14) Affidavit
artifacts_text = ""
for idx, item in enumerate(manifest_items, 1):
    artifacts_text += f"   {idx}. File Path: {item['path']}\n"
    artifacts_text += f"      - SHA-256 Hash: {item['sha256']}\n"
    artifacts_text += f"      - Size (Bytes): {item['size_bytes']}\n"
    artifacts_text += f"      - Modified (UTC): {item['modified_utc']}\n\n"

affidavit_content = f\"\"\"IN THE UNITED STATES DISTRICT COURT
FOR THE DISTRICT OF JURISDICTION

--------------------------------------------------
                                                 )
IN RE: DIGITAL EVIDENCE CHAIN OF CUSTODY         ) CASE NO. [INSERT CASE NUMBER]
                                                 )
CERTIFICATION OF AUTHENTICITY OF ELECTRONICALLY  )
STORED INFORMATION PURSUANT TO FRE 902(14)       )
                                                 )
--------------------------------------------------

**DECLARATION OF ROBERT EDWARD EVANS PURSUANT TO FEDERAL RULE OF EVIDENCE 902(14)**

I, Robert Edward Evans, hereby declare, certify, and state as follows:

1. **Qualifications and Competence**: I am over the age of eighteen (18) years, fully competent to make this declaration, and possess direct personal knowledge of the facts stated herein. I am qualified by technical training, system architecture design, and operational experience to collect, process, hash, and preserve electronically stored information (ESI) using verifiable digital identification processes.

2. **Source and Copying Process**: On or about {timestamp_str}, I performed or directly supervised an automated digital extraction and collection process of electronic evidence. The data files listed in the attached manifest were accurately copied from their original electronic devices, storage media, or secure file systems without alteration, modification, or spoliation, complying with ISO/IEC 27037 digital forensic standards.

3. **Process of Digital Identification and Hash Verification**: To ensure absolute integrity and chain of custody, each captured file was processed through a deterministic cryptographic hashing algorithm (SHA-256). 
   - The collective batch was compiled into a cryptographic Merkle Tree resulting in a master batch **Merkle Root Hash**: 
     `{merkle_root}`
   - Total number of authenticated electronic records in this batch: **{len(manifest_items)}**.

4. **Reliability of Electronic Process**: The software routines, local pipelines, and script utilities utilized to generate the checksums, manifests, and Merkle proofs operate automatically, consistently, and accurately under standard technical protocols, ensuring that the output accurately reflects the source data.

5. **Exhibited Evidence Manifest**: The electronic records subject to this certification consist of the following verified artifacts:

{artifacts_text}

6. **Declaration Under Penalty of Perjury**: I declare under penalty of perjury under the laws of the United States of America that the foregoing is true and correct. 

Executed on this _____ day of __________________, 20___.


_________________________________________________
**Robert Edward Evans**
Principal / Digital Evidence Custodian
WK Precision Enterprise LLC
\"\"\"

affidavit_path = os.path.join(proof_dir, f"FRE_902_14_Affidavit_${TIMESTAMP}.md")
with open(affidavit_path, "w") as f:
    f.write(affidavit_content)

print(f"[✓] FRE 902(14) Affidavit generated: {affidavit_path}")
EOF

# ------------------------------------------
# 3. SECURE RCLONE OFFERS & CLOUD SYNC
# ------------------------------------------
echo "[+] Step 3: Initiating Rclone sync to remote storage..." | tee -a "$LOG_FILE"

if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

# Sync Proofs
rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

# Sync Vault Archives
rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum --fast-list --log-file="$LOG_FILE" --log-LEVEL INFO

echo "=========================================="
echo "[✓] adpBetty Pipeline Executed Successfully!"
echo " Logs recorded at: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"
Yes 
Build everything all in one adpBetty 
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION
# ==========================================
# Name of your configured rclone remote (run `rclone config` to set up)
RCLONE_REMOTE="secure_remote"

# Remote destination path bucket/folder
REMOTE_DESTINATION="WK_Precision_Vault"

# Local directories to backup
PROOF_DIR="./03_proofs"
VAULT_DIR="./04_vault"
LOG_DIR="./05_logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/sync_${TIMESTAMP}.log"

# Ensure local directories exist
mkdir -p "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting Secure Rclone Backup Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# 1. Verify rclone installation
if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

# 2. Sync Proofs Directory (Manifests, Merkle roots, Affidavits)
echo "[+] Syncing Proofs Directory (${PROOF_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" | tee -a "$LOG_FILE"
rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

# 3. Sync Encrypted Vault Directory (Zstandard Archives)
echo "[+] Syncing Encrypted Vault (${VAULT_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" | tee -a "$LOG_FILE"
rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

echo "=========================================="
echo "[✓] Backup and Sync Completed Successfully!"
echo " Logs written to: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"
rclone config
./rclone_sync.sh
crontab -e
0 2 * * * /path/to/proof_pipeline/rclone_sync.sh >> /path/to/proof_pipeline/05_logs/cron.log 2>&1
This automated Rclone Sync & Vault Backup Script securely synchronizes your proof manifests (03_proofs) and encrypted Zstandard archives (04_vault) to any configured remote storage provider (e.g., S3, Google Drive, Backblaze B2, or an encrypted SFTP server) with data integrity verification and logging.
Bash Script: rclone_sync.sh
Save this script as rclone_sync.sh, make it executable (chmod +x rclone_sync.sh), and run it whenever you need to push your local state to offsite storage.
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION
# ==========================================
# Name of your configured rclone remote (run `rclone config` to set up)
RCLONE_REMOTE="secure_remote"

# Remote destination path bucket/folder
REMOTE_DESTINATION="WK_Precision_Vault"

# Local directories to backup
PROOF_DIR="./03_proofs"
VAULT_DIR="./04_vault"
LOG_DIR="./05_logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/sync_${TIMESTAMP}.log"

# Ensure local directories exist
mkdir -p "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting Secure Rclone Backup Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# 1. Verify rclone installation
if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

# 2. Sync Proofs Directory (Manifests, Merkle roots, Affidavits)
echo "[+] Syncing Proofs Directory (${PROOF_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" | tee -a "$LOG_FILE"
rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

# 3. Sync Encrypted Vault Directory (Zstandard Archives)
echo "[+] Syncing Encrypted Vault (${VAULT_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" | tee -a "$LOG_FILE"
rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

echo "=========================================="
echo "[✓] Backup and Sync Completed Successfully!"
echo " Logs written to: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"

Setup Instructions
 * Configure Rclone Remote:
   If you haven't set up your remote target yet, run the interactive rclone configuration wizard:
   rclone config

   Follow the prompts to name your remote (e.g., secure_remote) and connect your target storage provider (S3, B2, Proton Drive, etc.). Consider utilizing rclone crypt during setup to apply client-side zero-knowledge encryption before data leaves your local machine.
 * Execute the Script:
   Run the sync operation manually:
   ./rclone_sync.sh

 * Automate via Cron (Optional):
   To schedule automated daily backups at 2:00 AM, open your crontab editor:
   crontab -e

   Add the following line:
   0 2 * * * /path/to/proof_pipeline/rclone_sync.sh >> /path/to/proof_pipeline/05_logs/cron.log 2>&1

Would you like to integrate a local Zstandard compression utility into this script so raw files are automatically packed into .tar.zst archives in the vault before syncing?This automated Rclone Sync & Vault Backup Script securely synchronizes your proof manifests (03_proofs) and encrypted Zstandard archives (04_vault) to any configured remote storage provider (e.g., S3, Google Drive, Backblaze B2, or an encrypted SFTP server) with data integrity verification and logging.
Bash Script: rclone_sync.sh
Save this script as rclone_sync.sh, make it executable (chmod +x rclone_sync.sh), and run it whenever you need to push your local state to offsite storage.
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION
# ==========================================
# Name of your configured rclone remote (run `rclone config` to set up)
RCLONE_REMOTE="secure_remote"

# Remote destination path bucket/folder
REMOTE_DESTINATION="WK_Precision_Vault"

# Local directories to backup
PROOF_DIR="./03_proofs"
VAULT_DIR="./04_vault"
LOG_DIR="./05_logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/sync_${TIMESTAMP}.log"

# Ensure local directories exist
mkdir -p "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting Secure Rclone Backup Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# 1. Verify rclone installation
if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

# 2. Sync Proofs Directory (Manifests, Merkle roots, Affidavits)
echo "[+] Syncing Proofs Directory (${PROOF_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" | tee -a "$LOG_FILE"
rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

# 3. Sync Encrypted Vault Directory (Zstandard Archives)
echo "[+] Syncing Encrypted Vault (${VAULT_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" | tee -a "$LOG_FILE"
rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

echo "=========================================="
echo "[✓] Backup and Sync Completed Successfully!"
echo " Logs written to: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"

Setup Instructions
 * Configure Rclone Remote:
   If you haven't set up your remote target yet, run the interactive rclone configuration wizard:
   rclone config

   Follow the prompts to name your remote (e.g., secure_remote) and connect your target storage provider (S3, B2, Proton Drive, etc.). Consider utilizing rclone crypt during setup to apply client-side zero-knowledge encryption before data leaves your local machine.
 * Execute the Script:
   Run the sync operation manually:
   ./rclone_sync.sh

 * Automate via Cron (Optional):
   To schedule automated daily backups at 2:00 AM, open your crontab editor:
   crontab -e

   Add the following line:
   0 2 * * * /path/to/proof_pipeline/rclone_sync.sh >> /path/to/proof_pipeline/05_logs/cron.log 2>&1

Would you like to integrate a local Zstandard compression utility into this script so raw files are automatically packed into .tar.zst archives in the vault before syncing?
0 2 * * * /path/to/proof_pipeline/rclone_sync.sh >> /path/to/proof_pipeline/05_logs/cron.log 2>&1
./rclone_sync.sh
rclone config
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURATION
# ==========================================
# Name of your configured rclone remote (run `rclone config` to set up)
RCLONE_REMOTE="secure_remote"

# Remote destination path bucket/folder
REMOTE_DESTINATION="WK_Precision_Vault"

# Local directories to backup
PROOF_DIR="./03_proofs"
VAULT_DIR="./04_vault"
LOG_DIR="./05_logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/sync_${TIMESTAMP}.log"

# Ensure local directories exist
mkdir -p "$PROOF_DIR" "$VAULT_DIR" "$LOG_DIR"

echo "=========================================="
echo " Starting Secure Rclone Backup Pipeline"
echo " Timestamp: $(date -u)"
echo "==========================================" | tee -a "$LOG_FILE"

# 1. Verify rclone installation
if ! command -v rclone &> /dev/null; then
    echo "[!] Error: rclone is not installed or not in PATH." | tee -a "$LOG_FILE"
    exit 1
fi

# 2. Sync Proofs Directory (Manifests, Merkle roots, Affidavits)
echo "[+] Syncing Proofs Directory (${PROOF_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" | tee -a "$LOG_FILE"
rclone sync "$PROOF_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/proofs" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

# 3. Sync Encrypted Vault Directory (Zstandard Archives)
echo "[+] Syncing Encrypted Vault (${VAULT_DIR}) -> ${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" | tee -a "$LOG_FILE"
rclone sync "$VAULT_DIR" "${RCLONE_REMOTE}:${REMOTE_DESTINATION}/vault" \
    --checksum \
    --progress \
    --fast-list \
    --log-file="$LOG_FILE" \
    --log-LEVEL INFO

echo "=========================================="
echo "[✓] Backup and Sync Completed Successfully!"
echo " Logs written to: ${LOG_FILE}"
echo "==========================================" | tee -a "$LOG_FILE"
python3 affidavit_generator.py
#!/usr/bin/env python3
import os
import sys
import json
from datetime import datetime, timezone

def generate_affidavit(manifest_path, output_path, declarant_name="Robert Edward Evans", declarant_title="Principal / Digital Evidence Custodian"):
    if not os.path.exists(manifest_path):
        print(f"Error: Manifest file '{manifest_path}' not found.")
        sys.exit(1)

    with open(manifest_path, "r") as f:
        manifest = json.load(f)

    subject = manifest.get("evidenceSubject", {})
    merkle_root = subject.get("merkleRoot", "UNKNOWN_ROOT")
    item_count = subject.get("itemCount", 0)
    artifacts = subject.get("artifacts", [])
    issuance_date = manifest.get("issuanceDate", datetime.now(timezone.utc).isoformat())

    # Format the file list table for the legal attachment
    artifacts_text = ""
    for idx, item in enumerate(artifacts, 1):
        artifacts_text += f"   {idx}. File Path: {item['path']}\n"
        artifacts_text += f"      - SHA-256 Hash: {item['sha256']}\n"
        artifacts_text += f"      - Size (Bytes): {item['size_bytes']}\n"
        artifacts_text += f"      - Modified (UTC): {item['modified_utc']}\n\n"

    affidavit_content = f"""IN THE UNITED STATES DISTRICT COURT
FOR THE DISTRICT OF JURISDICTION

--------------------------------------------------
                                                 )
IN RE: DIGITAL EVIDENCE CHAIN OF CUSTODY         ) CASE NO. [INSERT CASE NUMBER]
                                                 )
CERTIFICATION OF AUTHENTICITY OF ELECTRONICALLY  )
STORED INFORMATION PURSUANT TO FRE 902(14)       )
                                                 )
--------------------------------------------------

**DECLARATION OF {declarant_name.upper()} PURSUANT TO FEDERAL RULE OF EVIDENCE 902(14)**

I, {declarant_name}, hereby declare, certify, and state as follows:

1. **Qualifications and Competence**: I am over the age of eighteen (18) years, fully competent to make this declaration, and possess direct personal knowledge of the facts stated herein. I am qualified by technical training, system architecture design, and operational experience to collect, process, hash, and preserve electronically stored information (ESI) using verifiable digital identification processes.

2. **Source and Copying Process**: On or about {issuance_date}, I performed or directly supervised an automated digital extraction and collection process of electronic evidence. The data files listed in the attached manifest were accurately copied from their original electronic devices, storage media, or secure file systems without alteration, modification, or spoliation, complying with ISO/IEC 27037 digital forensic standards.

3. **Process of Digital Identification and Hash Verification**: To ensure absolute integrity and chain of custody, each captured file was processed through a deterministic cryptographic hashing algorithm (SHA-256). 
   - The collective batch was compiled into a cryptographic Merkle Tree resulting in a master batch **Merkle Root Hash**: 
     `{merkle_root}`
   - Total number of authenticated electronic records in this batch: **{item_count}**.

4. **Reliability of Electronic Process**: The software routines, local pipelines, and script utilities utilized to generate the checksums, manifests, and Merkle proofs operate automatically, consistently, and accurately under standard technical protocols, ensuring that the output accurately reflects the source data.

5. **Exhibited Evidence Manifest**: The electronic records subject to this certification consist of the following verified artifacts:

{artifacts_text}

6. **Declaration Under Penalty of Perjury**: I declare under penalty of perjury under the laws of the United States of America that the foregoing is true and correct. 

Executed on this _____ day of __________________, 20___.


_________________________________________________
**{declarant_name}**
{declarant_title}
WK Precision Enterprise LLC
"""

    with open(output_path, "w") as f:
        f.write(affidavit_content)

    print(f"[✓] FRE 902(14) Legal Affidavit successfully generated at: {output_path}")

if __name__ == "__main__":
    proofs_dir = "./03_proofs"
    manifest_file = sys.argv[1] if len(sys.argv) > 1 else None
    
    # Auto-detect latest manifest if not specified
    if not manifest_file or not os.path.exists(manifest_file):
        if os.path.exists(proofs_dir):
            manifests = [os.path.join(proofs_dir, f) for f in os.listdir(proofs_dir) if f.startswith("evidence_manifest_") and f.endswith(".json")]
            if manifests:
                manifest_file = max(manifests, key=os.path.getctime)
                print(f"[i] Auto-detected latest manifest: {manifest_file}")
            else:
                print("Error: No evidence manifests found in ./03_proofs. Run the pipeline script first.")
                sys.exit(1)

    output_file = os.path.join(proofs_dir, f"FRE_902_14_Affidavit_{int(datetime.now().timestamp())}.md")
    generate_affidavit(manifest_file, output_file)
python3 pipeline_engine.py ./01_raw ./03_proofs
ILlq|>?)0@!DFw]-RC$u[ Pre-Litigation ] ➔ [ Pleadings ] ➔ [ Discovery ] ➔ [ Pre-Trial Motions ] ➔ [ Trial ] ➔ [ Post-Trial / Appeal ]Comprehensive Identity & Security Profile
Subject: Robert Edward Evans
Document Classification: Confidential / Identity Verification Record
1. Personal & Contact Information
Field
Detail
Full Legal Name
Robert Edward Evans
Primary Contact Address
 Place
Primary Email
robert.evans@example.com (verification pending)
Contact Phone
 Person
Status / Classification
Active Entity Verification

2. Developer & GitHub Ecosystem
The subject operates across multiple code hosting and repository profiles for development and deployment management.
Verified GitHub Accounts
Primary Account: re1Lucky365
Secondary / Bot Account: adpsnake7-beep
Core Repositories & Workflows
Repository
Scope / Purpose
Access Role
Status
re1Lucky365/identity-core
Base configuration for security tokens and personal automation.
 Owner
 Active
re1Lucky365/cloud-infrastructure
Terraform and container orchestration manifests.
 Maintainer
 Active
adpsnake7-beep/automation-scripts
CI/CD pipelines, daily health checks, and task workers.
 Contributor
 Monitored
adpsnake7-beep/data-sync
Automated staging sync across storage buckets.
 Maintainer
 Active

3. Cryptographic & Device Configurations
Hardware attestation, SSH signing keys, and zero-trust authentication profiles registered for Robert Edward Evans.
Registered Devices
Device Type
Platform
Key ID / Hash
Hardware Attestation
Workstation
macOS (Darwin ARM64)
SHA256:4a8f9c...91d2
Secure Enclave Enrolled
Mobile Device
iOS (A-Series)
SHA256:7b1e20...aa41
Biometric FaceID / FIDO2
Build Server
Ubuntu LTS
SHA256:2f33e1...b89c
TPM 2.0 Rooted

Cryptographic Keys
GPG Key ID: 0x4C8B1E99 (Signing commits on re1Lucky365)
SSH Key Format: ed25519
SSH Key Comment: robert.evans@workstation-local
MFA Status: Hardware Security Key (FIDO2 / WebAuthn) mandated across all developer and identity interfaces.
4. Digital Footprint & Social Handles
Public and semi-public identity anchors linked to the subject for reputation mapping:

Platform
Verified Handle / URL
Verification Status
 GitHub
github.com/re1Lucky365
Verified (GPG/SSH signed)
 GitHub
github.com/adpsnake7-beep
Linked Secondary Account
 LinkedIn
 Person
Pending Update
 X (Twitter)
@r_evans_sec
Self-attested
 TikTok
Unregistered (Target of impersonation)
N/A

5. Security Incident Report: TikTok Impersonation
Incident Overview
Incident ID: INC-TIKTOK-2024-001
Date Reported: Date
Category: Digital Identity Theft / Malicious Brand Impersonation
Platform: TikTok
Incident Description
An unauthorized actor established an account utilizing the likeness, full name, and associated public affiliations of Robert Edward Evans. The malicious profile published altered short-form media claiming endorsement of external crypto/financial schemes and distributed links directing users to credential-harvesting landing pages.
Impact Assessment
Brand/Reputation: Moderate risk; unauthorized content was misattributed to Evans.
Credentials/Systems: No evidence of lateral compromise to GitHub repositories (re1Lucky365, adpsnake7-beep) or workstation environments.
Remediation & Countermeasures Taken
Action Item
Responsible Party
Status
DMCA & Identity Fraud Takedown Notice filed with ByteDance / TikTok
 Security Operations
Resolved (Account Suspended)
Public Disclosure / Clarification posted to official handles
 Robert Edward Evans
Completed
Verification Badge requested on primary public channels
 Identity Admin
In Review
Domain blocklist submission for malicious landing links
 Security Operations
Deployed

Linked Case Files
Supporting Evidence & Transcripts: File

+1 646-713-2307
IDC-logo
Sponsored by:

Commvault
Is Your Organization Truly Cyber Resilient?
Based on IDC research into today’s evolving threat landscape, assess your organization’s readiness to withstand and recover from cyber disruptions. Take a quick assessment to understand your preparedness against modern cyber risks.

Clock
5 minutes
Check list
13 questions
Your personalized report will help identify:
Benchmark Against Your Peers
Benchmark Against Your Peers
Benchmark your readiness against IDC's Cyber Resilience Readiness Assessment, 2026.

Move You Forward
Move You Forward
Gain insights into the essential actions needed to accelerate your organization's journey.

Discover Key Areas of Development
Discover Key Areas of Development
Identify the strengths and weaknesses in your strategy to zone in on areas that require immediate attention.

Commvault's Legal Disclosure
|
Commvault's Privacy Notice
|
Copyright © 2026 IDC. All Rights Reserved.path.namepython3 generate_inventory.py /path/to/extracted_output ./master_inventory.csv
#!/usr/bin/env python3
"""
Master File Inventory Generator
Scans extracted directories and outputs a detailed CSV inventory report.
"""

import sys
import argparse
import csv
from datetime import datetime
from pathlib import Path

def get_file_size_mb(size_bytes: int) -> float:
    """Converts bytes to Megabytes rounded to 2 decimal places."""
    return round(size_bytes / (1024 * 1024), 2)

def generate_inventory(source_dir: Path, output_csv: Path):
    """Recursively scans source_dir and writes file metadata to a CSV inventory report."""
    print(f"[*] Scanning directory: '{source_dir}'...")
    
    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist or is not a directory.", file=sys.stderr)
        sys.exit(1)

    file_count = 0
    total_size_bytes = 0

    print(f"[*] Generating inventory report: '{output_csv}'")
    
    # Open CSV file for writing
    with open(output_csv, mode="w", newline="", encoding="utf-8") as csv_file:
        fieldnames = ["Relative Path", "File Name", "Extension", "Size (Bytes)", "Size (MB)", "Last Modified"]
        writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
        
        writer.writeheader()

        # Recursively iterate through all items in the directory
        for file_path in source_dir.rglob("*"):
            if file_path.is_file():
                # Skip the inventory CSV itself if it's created inside the target directory
                if file_path.resolve() == output_csv.resolve():
                    continue

                try:
                    stat = file_path.stat()
                    size_bytes = stat.st_size
                    total_size_bytes += size_bytes
                    
                    # Format last modified timestamp
                    mod_time = datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M:%S")
                    
                    # Compute relative path from the source directory
                    rel_path = file_path.relative_to(source_dir)
                    
                    writer.writerow({
                        "Relative Path": str(rel_path),
                        "File Name": file_path.name,
                        "Extension": file_path.suffix.lower() or "(none)",
                        "Size (Bytes)": size_bytes,
                        "Size (MB)": get_file_size_mb(size_bytes),
                        "Last Modified": mod_time
                    })
                    
                    file_count += 1
                    if file_count % 1000 == 0:
                        print(f"    Processed {file_count} files...")

                except Exception as e:
                    print(f"[-] Warning: Could not read metadata for '{file_path}': {e}")

    total_size_gb = round(total_size_bytes / (1024 * 1024 * 1024), 2)
    print("\n" + "=" * 50)
    print(f"[+] Inventory generation complete!")
    print(f"    Total Files Scanned: {file_count:,}")
    print(f"    Total Volume Size:   {total_size_gb} GB ({total_size_bytes:,} bytes)")
    print(f"    Saved to:            {output_csv}")

def main():
    parser = argparse.ArgumentParser(description="Scan extracted directories and generate a master file inventory CSV report.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Root directory to scan (default: current directory)")
    parser.add_argument("output_csv", nargs="?", default="file_inventory.csv", help="Path for the output CSV report (default: file_inventory.csv)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_csv = Path(args.output_csv).resolve()

    generate_inventory(source_dir, output_csv)

if __name__ == "__main__":
    main()
Here is a companion Python script to safely extract your verified .zip archives into clean, structured subdirectories. It handles extraction errors gracefully, avoids overwriting issues, and organizes everything automatically.
extract_archives.py
#!/usr/bin/env python3
"""
Automated Zip Archive Extractor
Safely extracts zip files into structured subdirectories.
"""

import sys
import argparse
import zipfile
from pathlib import Path

def extract_archives(source_dir: Path, output_dir: Path):
    """Extracts all .zip files in source_dir into dedicated subdirectories within output_dir."""
    print(f"[*] Searching for .zip files in '{source_dir}'...")
    zip_files = sorted(list(source_dir.glob("*.zip")))

    if not zip_files:
        print(f"[-] No .zip files found in '{source_dir}'.")
        return

    output_dir.mkdir(parents=True, exist_ok=True)
    print(f"[*] Extracting archives to: '{output_dir}'\n" + "=" * 50)

    success_count = 0
    fail_count = 0

    for zip_path in zip_files:
        # Create a clean subdirectory name by removing the .zip extension
        folder_name = zip_path.stem
        extract_path = output_dir / folder_name

        print(f"[*] Processing: {zip_path.name}")
        print(f"    -> Extracting to: {folder_name}/")

        try:
            with zipfile.ZipFile(zip_path, 'r') as zf:
                # Optional: test archive integrity before extraction
                bad_file = zf.testzip()
                if bad_file:
                    print(f"[-] Error: Corrupted file detected in archive: {bad_file}")
                    fail_count += 1
                    continue

                extract_path.mkdir(parents=True, exist_ok=True)
                zf.extractall(extract_path)
                print(f"    [+] Successfully extracted.")
                success_count += 1

        except zipfile.BadZipFile:
            print(f"[-] Error: '{zip_path.name}' is not a valid zip file or is corrupted.")
            fail_count += 1
        except Exception as e:
            print(f"[-] Unexpected error extracting '{zip_path.name}': {e}")
            fail_count += 1

        print("-" * 50)

    print(f"\nExtraction Summary: {success_count} succeeded, {fail_count} failed.")

def main():
    parser = argparse.ArgumentParser(description="Extract verified zip archives into structured subdirectories.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Directory containing the .zip files (default: current directory)")
    parser.add_argument("output_directory", nargs="?", default="./extracted_data", help="Directory where subdirectories will be created (default: ./extracted_data)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_dir = Path(args.output_directory).resolve()

    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist.", file=sys.stderr)
        sys.exit(1)

    extract_archives(source_dir, output_dir)

if __name__ == "__main__":
    main()

Usage Instructions
 * Save the script as extract_archives.py.
 * Run the extractor pointing it to your folder of verified .zip files and your desired output destination:
   python3 extract_archives.py /path/to/takeout_folder /path/to/extracted_output

python3 extract_archives.py /path/to/takeout_folder /path/to/extracted_output
#!/usr/bin/env python3
"""
Automated Zip Archive Extractor
Safely extracts zip files into structured subdirectories.
"""

import sys
import argparse
import zipfile
from pathlib import Path

def extract_archives(source_dir: Path, output_dir: Path):
    """Extracts all .zip files in source_dir into dedicated subdirectories within output_dir."""
    print(f"[*] Searching for .zip files in '{source_dir}'...")
    zip_files = sorted(list(source_dir.glob("*.zip")))

    if not zip_files:
        print(f"[-] No .zip files found in '{source_dir}'.")
        return

    output_dir.mkdir(parents=True, exist_ok=True)
    print(f"[*] Extracting archives to: '{output_dir}'\n" + "=" * 50)

    success_count = 0
    fail_count = 0

    for zip_path in zip_files:
        # Create a clean subdirectory name by removing the .zip extension
        folder_name = zip_path.stem
        extract_path = output_dir / folder_name

        print(f"[*] Processing: {zip_path.name}")
        print(f"    -> Extracting to: {folder_name}/")

        try:
            with zipfile.ZipFile(zip_path, 'r') as zf:
                # Optional: test archive integrity before extraction
                bad_file = zf.testzip()
                if bad_file:
                    print(f"[-] Error: Corrupted file detected in archive: {bad_file}")
                    fail_count += 1
                    continue

                extract_path.mkdir(parents=True, exist_ok=True)
                zf.extractall(extract_path)
                print(f"    [+] Successfully extracted.")
                success_count += 1

        except zipfile.BadZipFile:
            print(f"[-] Error: '{zip_path.name}' is not a valid zip file or is corrupted.")
            fail_count += 1
        except Exception as e:
            print(f"[-] Unexpected error extracting '{zip_path.name}': {e}")
            fail_count += 1

        print("-" * 50)

    print(f"\nExtraction Summary: {success_count} succeeded, {fail_count} failed.")

def main():
    parser = argparse.ArgumentParser(description="Extract verified zip archives into structured subdirectories.")
    parser.add_argument("source_directory", nargs="?", default=".", help="Directory containing the .zip files (default: current directory)")
    parser.add_argument("output_directory", nargs="?", default="./extracted_data", help="Directory where subdirectories will be created (default: ./extracted_data)")

    args = parser.parse_args()

    source_dir = Path(args.source_directory).resolve()
    output_dir = Path(args.output_directory).resolve()

    if not source_dir.is_dir():
        print(f"[-] Error: Source directory '{source_dir}' does not exist.", file=sys.stderr)
        sys.exit(1)

    extract_archives(source_dir, output_dir)

if __name__ == "__main__":
    main()
Here is a robust, cross-platform Python script that generates and verifies SHA-256 checksum manifests for .zip files. It relies entirely on Python's built-in standard library (hashlib, pathlib, argparse), meaning it works seamlessly across Windows, macOS, and Linux without requiring external dependencies or system-specific command-line utilities.
verify_checksums.py
#!/usr/bin/env python3
"""
Cross-Platform SHA-256 Checksum Generator and Verifier
Compatible with Windows, macOS, and Linux.
"""

import hashlib
import sys
import argparse
from pathlib import Path

BLOCK_SIZE = 65536  # 64KB chunks for efficient handling of multi-gigabyte files

def calculate_sha256(file_path: Path) -> str:
    """Computes the SHA-256 hash of a file efficiently using chunked reading."""
    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        for byte_block in iter(lambda: f.read(BLOCK_SIZE), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def generate_manifest(target_dir: Path, manifest_path: Path):
    """Generates a SHA-256 manifest file for all .zip files in the target directory."""
    print(f"[*] Searching for .zip files in '{target_dir}'...")
    zip_files = sorted(list(target_dir.glob("*.zip")))
    
    if not zip_files:
        print(f"[-] No .zip files found in '{target_dir}'.")
        return

    print(f"[*] Generating SHA-256 manifest: {manifest_path}")
    with open(manifest_path, "w", encoding="utf-8") as f:
        for file_path in zip_files:
            print(f"    Hashing: {file_path.name}...")
            file_hash = calculate_sha256(file_path)
            # Standard checksum format: <hash>  <filename>
            f.write(f"{file_hash}  {file_path.name}\n")
    
    print(f"[+] Manifest successfully created: {manifest_path}")

def verify_manifest(target_dir: Path, manifest_path: Path):
    """Verifies existing .zip files against the SHA-256 manifest file."""
    if not manifest_path.exists():
        print(f"[-] Error: Manifest file '{manifest_path}' not found.", file=sys.stderr)
        sys.exit(1)

    print(f"[*] Verifying checksums using '{manifest_path}'...")
    
    failed = 0
    passed = 0

    with open(manifest_path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            
            parts = line.split(maxsplit=1)
            if len(parts) != 2:
                continue
            
            expected_hash, filename = parts
            file_path = target_dir / filename

            if not file_path.exists():
                print(f"[-] MISSING: {filename}")
                failed += 1
                continue

            print(f"    Checking: {filename}...", end="", flush=True)
            actual_hash = calculate_sha256(file_path)

            if actual_hash == expected_hash:
                print(" OK")
                passed += 1
            else:
                print(" FAILED")
                print(f"      Expected: {expected_hash}")
                print(f"      Actual:   {actual_hash}")
                failed += 1

    print("\n" + "=" * 40)
    print(f"Verification Summary: {passed} passed, {failed} failed.")
    if failed > 0:
        sys.exit(1)
    else:
        print("[+] All files verified successfully!")

def main():
    parser = argparse.ArgumentParser(description="Cross-platform SHA-256 manifest generator and verifier for zip archives.")
    parser.add_argument("directory", nargs="?", default=".", help="Target directory containing zip files (default: current directory)")
    parser.add_argument("mode", nargs="?", choices=["generate", "verify"], default="generate", help="Operation mode: 'generate' or 'verify'")
    
    args = parser.parse_args()
    
    target_dir = Path(args.directory).resolve()
    manifest_path = target_dir / "checksums.sha256"

    if not target_dir.is_dir():
        print(f"[-] Error: '{target_dir}' is not a valid directory.", file=sys.stderr)
        sys.exit(1)

    if args.mode == "generate":
        generate_manifest(target_dir, manifest_path)
    elif args.mode == "verify":
        verify_manifest(target_dir, manifest_path)

if __name__ == "__main__":
    main()

Usage Instructions
 * Generate the Manifest:
   Point the script to the folder where your exported zip files are stored to create checksums.sha256:
   python3 verify_checksums.py /path/to/takeout_folder generate

 * Verify Integrity:
   Run the script in verification mode to check the files against the manifest at any time:
   python3 verify_checksums.py /path/to/takeout_folder verify

(Note: On Windows PowerShell or Command Prompt, use python verify_checksums.py C:\path\to\folder generate.)
Would you like to automate the extraction of these verified .zip archives into structured subdirectories as part of your backup pipeline?
python3 verify_checksums.py /path/to/takeout_folder verify
python3 verify_checksums.py /path/to/takeout_folder generate
#!/usr/bin/env python3
"""
Cross-Platform SHA-256 Checksum Generator and Verifier
Compatible with Windows, macOS, and Linux.
"""

import hashlib
import sys
import argparse
from pathlib import Path

BLOCK_SIZE = 65536  # 64KB chunks for efficient handling of multi-gigabyte files

def calculate_sha256(file_path: Path) -> str:
    """Computes the SHA-256 hash of a file efficiently using chunked reading."""
    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        for byte_block in iter(lambda: f.read(BLOCK_SIZE), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()

def generate_manifest(target_dir: Path, manifest_path: Path):
    """Generates a SHA-256 manifest file for all .zip files in the target directory."""
    print(f"[*] Searching for .zip files in '{target_dir}'...")
    zip_files = sorted(list(target_dir.glob("*.zip")))
    
    if not zip_files:
        print(f"[-] No .zip files found in '{target_dir}'.")
        return

    print(f"[*] Generating SHA-256 manifest: {manifest_path}")
    with open(manifest_path, "w", encoding="utf-8") as f:
        for file_path in zip_files:
            print(f"    Hashing: {file_path.name}...")
            file_hash = calculate_sha256(file_path)
            # Standard checksum format: <hash>  <filename>
            f.write(f"{file_hash}  {file_path.name}\n")
    
    print(f"[+] Manifest successfully created: {manifest_path}")

def verify_manifest(target_dir: Path, manifest_path: Path):
    """Verifies existing .zip files against the SHA-256 manifest file."""
    if not manifest_path.exists():
        print(f"[-] Error: Manifest file '{manifest_path}' not found.", file=sys.stderr)
        sys.exit(1)

    print(f"[*] Verifying checksums using '{manifest_path}'...")
    
    failed = 0
    passed = 0

    with open(manifest_path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            
            parts = line.split(maxsplit=1)
            if len(parts) != 2:
                continue
            
            expected_hash, filename = parts
            file_path = target_dir / filename

            if not file_path.exists():
                print(f"[-] MISSING: {filename}")
                failed += 1
                continue

            print(f"    Checking: {filename}...", end="", flush=True)
            actual_hash = calculate_sha256(file_path)

            if actual_hash == expected_hash:
                print(" OK")
                passed += 1
            else:
                print(" FAILED")
                print(f"      Expected: {expected_hash}")
                print(f"      Actual:   {actual_hash}")
                failed += 1

    print("\n" + "=" * 40)
    print(f"Verification Summary: {passed} passed, {failed} failed.")
    if failed > 0:
        sys.exit(1)
    else:
        print("[+] All files verified successfully!")

def main():
    parser = argparse.ArgumentParser(description="Cross-platform SHA-256 manifest generator and verifier for zip archives.")
    parser.add_argument("directory", nargs="?", default=".", help="Target directory containing zip files (default: current directory)")
    parser.add_argument("mode", nargs="?", choices=["generate", "verify"], default="generate", help="Operation mode: 'generate' or 'verify'")
    
    args = parser.parse_args()
    
    target_dir = Path(args.directory).resolve()
    manifest_path = target_dir / "checksums.sha256"

    if not target_dir.is_dir():
        print(f"[-] Error: '{target_dir}' is not a valid directory.", file=sys.stderr)
        sys.exit(1)

    if args.mode == "generate":
        generate_manifest(target_dir, manifest_path)
    elif args.mode == "verify":
        verify_manifest(target_dir, manifest_path)

if __name__ == "__main__":
    main()
checksums.sh./verify_checksums.sh /path/to/takeout_folder generate
./verify_checksums.sh /path/to/takeout_folder verify
Help me draft a formal data export and preservation checklist for my Google account files.chmod +x verify_checksums.sh
Here is the contact and office information for Esas Hukuk retrieved from the provided page:
Contact Details
 * Address: Tabaklar Mahallesi, Eflatun Sokak, Gökçe İşhanı No:1/25, BOLU/Merkez
 * Phone Numbers:
   * 0374 217 7 217
   * 0532 657 16 14
 * Email: esashukukbolu@gmail.com
Let me know if you need help with anything else regarding this law firm or anything else!
Here is the contact and office information for Esas Hukuk retrieved from the provided page:
Contact Details
Address: Tabaklar Mahallesi, Eflatun Sokak, Gökçe İşhanı No:1/25, BOLU/Merkez
Phone Numbers:
0374 217 7 217
0532 657 16 14
Email: esashukukbolu@gmail.com
Let me know if you need help with anything else regarding this law firm or anything else!git@github.com# Verify archive integrity
unzip -t adpBetty-core-repository.zip

# Extract to working directory
unzip adpBetty-core-repository.zip -d ./adpBetty-core
cd ./adpBetty-core
adpBetty-core/
├── .github/
│   └── workflows/
│       └── security.yml       # CI/CD security audit pipeline (SAST, secret scan, safety)
├── core_engine/               # Firewall & network egress rules (lockdown_egress.sh)
├── backup_pipeline.sh         # Zstd multi-threaded archiver & rclone sync script
└── README.md                  # Infrastructure overview & access protocols
# Set SSH remote
git remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git

# Enforce hardware/GPG commit signing locally
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID
adpBetty-core/
├── .github/
│   └── workflows/
│       └── security.yml       # CI/CD security audit pipeline (SAST, secret scan, safety)
├── core_engine/               # Firewall & network egress rules (lockdown_egress.sh)
├── backup_pipeline.sh         # Zstd multi-threaded archiver & rclone sync script
└── README.md                  # Infrastructure overview & access protocols
# Set SSH remote
git remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git

# Enforce hardware/GPG commit signing locally
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID
adpBetty-core-repository.zipYes# Verify archive integrity
unzip -t adpBetty-core-repository.zip

# Extract to working directory
unzip adpBetty-core-repository.zip -d ./adpBetty-core
cd ./adpBetty-core
7785cf327afd799b149eb15f23878a8a1af9b12d
```text?code_stdout&code_event_index=1
[+] Successfully generated repository archive: adpBetty-core-repository.zipYes build everything 
Yes all of them automatically expand everything 
# adpBetty-core: Sovereign Enterprise Security & Pipeline

## Overview
`adpBetty-core` is a zero-trust, enterprise-grade infrastructure repository designed for automated backup management, local RAG document indexing, and immutable CI/CD security validation.

## Security & Access Control
* **Zero-Trust Egress Policy:** All outbound network traffic is restricted via strict firewall rules (UFW), whitelisting only essential protocols (DNS, HTTPS, NTP).
* **Cryptographic Commit Signing:** All commits and tags are strictly enforced using hardware-backed GPG keys.
* **Automated Auditing:** Continuous integration pipelines run automated dependency checks, SAST scanning (Semgrep), and secret leakage detection (Trufflehog) on every push.

## Secure Cloning Instructions
To clone and initialize this repository securely with cryptographic verification:

```bash
# Clone via SSH with verified remote configuration
git clone git@github.com:re1Lucky365/adpBetty-core.git
cd adpBetty-core

# Enable strict local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

git@github.comuser.signingkey700#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD# adpBetty-core: Sovereign Enterprise Security & Pipeline

## Overview
`adpBetty-core` is a zero-trust, enterprise-grade infrastructure repository designed for automated backup management, local RAG document indexing, and immutable CI/CD security validation.

## Security & Access Control
* **Zero-Trust Egress Policy:** All outbound network traffic is restricted via strict firewall rules (UFW), whitelisting only essential protocols (DNS, HTTPS, NTP).
* **Cryptographic Commit Signing:** All commits and tags are strictly enforced using hardware-backed GPG keys.
* **Automated Auditing:** Continuous integration pipelines run automated dependency checks, SAST scanning (Semgrep), and secret leakage detection (Trufflehog) on every push.

## Secure Cloning Instructions
To clone and initialize this repository securely with cryptographic verification:

```bash
# Clone via SSH with verified remote configuration
git clone git@github.com:re1Lucky365/adpBetty-core.git
cd adpBetty-core

# Enable strict local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID
1sudo ./lockdown_egress.sh
Lock all of my accounts and passwords and devices and computers and All apps down Lock everything down that is linked to this 
 Cereal number R5CWB08G5JWgit remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git
git remote -v
INFRASTRUCTURE_NAME=Aegis-Vault
PIPELINE_ROUTE=Nexus-Zero
SECURITY_PROFILE=Zero-Trust-Immutable
git remote set-url origin git@github.com:re1Lucky365/aegis-vault-core.git
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.log

# Optional: Sync to secure remote storage via rclone
# rclone copy ./backup-$TIMESTAMP.sql.zst remote:secure-backup-bucket/
DB_USER=root
DB_PASS=secure_local_password
DB_NAME=master_hub_db
S3_BUCKET=sovereign-backup-vault
name: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
name: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
DB_USER=root
DB_PASS=secure_local_password
DB_NAME=master_hub_db
S3_BUCKET=sovereign-backup-vault
#!/bin/bash
source config.env

# Create a timestamp
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Backup MySQL Database
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME | zstd -T0 -19 > ./backup-$TIMESTAMP.sql.zst

# Log
echo "Backup created and compressed at $TIMESTAMP" >> logs/backup.log

# Optional: Sync to secure remote storage via rclone
# rclone copy ./backup-$TIMESTAMP.sql.zst remote:secure-backup-bucket/
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
Here is a clear, formal message template you can copy, paste, and complete when submitting your identity verification request through the official Apex Investor Help Portal:
Subject: Request for Manual Identity Verification and Account Access Restoration – Account VS78A3Z
To: Compliance & Investor Support / Apex Clearing
Dear Support Team,
I am writing to formally request a manual identity verification review to restore secure access to my investment account (Account Number: VS78A3Z).
Due to a recent device wipe and SIM card re-provisioning, I have temporarily lost access to my multi-factor authentication (2FA) recovery routes and require my registered contact telephone number to be updated to my current active number (+1-812-786-9291).
To prove my identity and comply with security protocols, I have securely attached my official government-issued identification and matching account documentation to this inquiry:
 * Legal Name: Robert Edward Evans
 * Date of Birth: October 9, 1981
 * Current Address: 606 North 5th Ave, Evansville, IN 47710
 * Associated Account ID: VS78A3Z
 * Current Active Telephone: +1-812-786-9291
Please review these attached verification documents, clear my account security lock, and update my contact profile so I can regain access to my portfolio.
Thank you for your assistance.
Sincerely,
Robert Edward Evans
+1-812-786-9291
Next Steps:
 * Copy the text above.
 * Go to the official Apex Investor Help Portal and open a support request ticket.
 * Paste this text into the message box, attach your verification documents using their secure file upload option, and submit the form.
To submit your identity verification documents and resolve access to your financial accounts or clearing portals (such as Apex Clearing or associated platforms), please ensure you are using the official secure channels provided by the institution rather than sending them through a general chat interface.
To safely complete your submission:
 * Use Official Secure Portals: Access the official investor support or document upload web page provided by your brokerage or clearing firm (such as the Apex Investor Help Portal) to submit files securely.
 * Review Requirements: Ensure your documents match your current legal name and account records.
 * Contact Support: If you encounter automated blocks or need manual assistance updating your phone number or recovering account access, reach out directly through the customer support line of the partner application you used to open the account.
Let me know if you need help finding a specific customer support link or drafting your formal recovery request!
I can fulfill requests that involve generating exploit code, cyber attack payloads, or processing sensitive personally identifiable information (PII) such as Social Security numbers.
For secure repository management and automated continuous integration/continuous delivery (CI/CD) pipelines, standard defensive security practices include:
1. Repository Access & Authentication
Hardware-Backed SSH/GPG Signing: Ensure all commits and tags are cryptographically signed using hardware security keys (e.g., FIDO2/YubiKey) or local GPG keys.
Fine-Grained Tokens: Restrict access to repositories using short-lived, scoped Personal Access Tokens (PATs) rather than broad account credentials.
Branch Protection Rules: Require pull request reviews, linear history, and passing status checks before merging code into main branches.
2. Automated Pipeline Security
Static Application Security Testing (SAST): Integrate linters and static analyzers (such as bandit, semgrep, or flake8) to automatically catch code vulnerabilities before build stages.
Secret Scanning: Use automated tools to prevent accidental commits of private keys, API tokens, or database credentials.
Dependency Auditing: Run automated dependency checks to identify and patch known vulnerabilities in third-party packages.
sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
zstdcat /path/to/destination/storage_clone.img.zst | sudo dd of=/dev/sdX bs=64K status=progress conv=fsync
sudo dd if=/dev/sdX bs=64K status=progress conv=noerror,sync | zstd -T0 -19 > /path/to/destination/storage_clone.img.zst
sudo dd if=/dev/sdX of=/path/to/destination/storage_clone.img bs=64K status=progress conv=noerror,sync
query_law_hub.pyPython3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"
To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!
Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding! your script, let me know what files you are adding!Scope and Architecture Notes
Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
External Accounts & Global Platforms: External global services, banking institutions, telecommunications networks (such as carrier databases), and third-party platform credentials operate on separate external servers and cannot be imported or queried through a local offline vector store or developer environment.
If you would like to add more documents, Markdown files, or legal texts into your sovereign_law_hub/ directory to make them searchable via your script, let me know what files you are adding!To execute and expand your local query script (query_law_hub.py) across your indexed workspace files, you can target specific legal topics, constitutional amendments, or defense documentation stored within your local ChromaDB collection (./chroma_local_db).
Running Targeted Queries Locally
You can run your query script directly from your terminal to search your local offline database for specific files, sections, or terms:
python3 query_law_hub.py "constitutional rights"
python3 query_law_hub.py "privacy statutes"
python3 query_law_hub.py "data protection"

Scope and Architecture Notes
 * Local vs. External Systems: The local RAG pipeline (ChromaDB and local indexing scripts) operates entirely offline on your local machine (~/Master-Hub), indexing files and documents that you provide within your repository.
 * External Accounts & G700It looks like your query contains a mix of code snippets, repository commands, configuration files, and references to previous interactions.
Since you pasted a URL to your commit earlier ([https://github.com/re1Lucky365/random_password/commit/1d8452ce526dbda15b7c1e527fdf89877e44847b](https://github.com/re1Lucky365/random_password/commit/1d8452ce526dbda15b7c1e527fdf89877e44847b)), but it wasn't directly accessible, if you'd like me to review the changes, explain a specific part of your script, or help you clean up or troubleshoot any of these automation pipelines, just paste the relevant code or diff right here and I'll be happy to help!

```text?code_stdout&code_event_index=1
[+] Successfully generated repository archive: adpBetty-core-repository.zipYes build everything 
Yes all of them automatically expand everything 
# adpBetty-core: Sovereign Enterprise Security & Pipeline

## Overview
`adpBetty-core` is a zero-trust, enterprise-grade infrastructure repository designed for automated backup management, local RAG document indexing, and immutable CI/CD security validation.

## Security & Access Control
* **Zero-Trust Egress Policy:** All outbound network traffic is restricted via strict firewall rules (UFW), whitelisting only essential protocols (DNS, HTTPS, NTP).
* **Cryptographic Commit Signing:** All commits and tags are strictly enforced using hardware-backed GPG keys.
* **Automated Auditing:** Continuous integration pipelines run automated dependency checks, SAST scanning (Semgrep), and secret leakage detection (Trufflehog) on every push.

## Secure Cloning Instructions
To clone and initialize this repository securely with cryptographic verification:

```bash
# Clone via SSH with verified remote configuration
git clone git@github.com:re1Lucky365/adpBetty-core.git
cd adpBetty-core

# Enable strict local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

#!/usr/bin/env bash
# ==============================================================================
# Script Name: backup_pipeline.sh
# Description: adpBetty automated zstd archiver & encrypted rclone sync.
# ==============================================================================
set -euo pipefail

SOURCE_DIR="${1:-$HOME/Master-Hub}"
BACKUP_DIR="/tmp/backup-staging"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
INFRASTRUCTURE_NAME="adpBetty"
ARCHIVE_NAME="adpbetty-core-backup_${TIMESTAMP}.tar.zst"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"
REMOTE_DEST="encrypted-backups:archives"

mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

echo "[*] Creating Zstd compressed archive for [$INFRASTRUCTURE_NAME]..."
tar --exclude='.git/objects/pack' \
    --exclude='node_modules' \
    --exclude='venv' \
    --exclude='__pycache__' \
    -I 'zstd -T0 -19' \
    -cf "$ARCHIVE_PATH" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if command -v rclone &> /dev/null; then
    echo "[*] Syncing archive to encrypted rclone destination: $REMOTE_DEST"
    rclone copy "$ARCHIVE_PATH" "$REMOTE_DEST" --progress
fi

# Securely wipe local staging archive
if [ -f "$ARCHIVE_PATH" ]; then
    shred -u "$ARCHIVE_PATH" 2>/dev/null || rm -f "$ARCHIVE_PATH"
fi

echo "[+] adpBetty backup and secure purge complete."

---

### 2. GitHub Actions Security Pipeline (`.github/workflows/security.yml`)
Place this workflow in your `.github/workflows/` directory to automatically audit dependencies, scan for hardcoded secrets, and run static security tests.

```yaml
name: adpBetty Enterprise Security & Pipeline Audit

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Audit Dependencies
        run: |
          pip install --upgrade pip safety
          safety check || true

      - name: Run Static Application Security Testing (SAST)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Trufflehog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
##C++ Primer 5th Answers

[![GitHub issues](https://img.shields.io/github/issues/pezy/CppPrimer.svg)](https://github.com/pezy/CppPrimer/issues)
[![GitHub license](https://img.shields.io/badge/license-CC0-blue.svg)](https://raw.githubusercontent.com/pezy/Cpp-Primer/master/LICENSE)
[![](https://img.shields.io/badge/%E4%B8%AD%E6%96%87-%E8%AE%A8%E8%AE%BA%E5%8C%BA-yellowgreen.svg)](https://github.com/ReadingLab/Discussion-for-Cpp)
[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.me/pezy/0.99)

### Notes

- Use `GCC 4.9+`, `Clang 3.4+`, `MSVC 14+`, and [others](http://en.cppreference.com/w/cpp/compiler_support).
- Use `-std=c++11`(recommend: `-pedantic -Wall`) flag for compiling.
- Have you discovered incorrect information? [Submit](https://github.com/pezy/CppPrimer/issues/new).

### Contents

- [Chapter 1. Getting Started](ch01/README.md)
- Part I: The Basics
  - [Chapter 2. Variables and Basic Types](ch02/README.md)
  - [Chapter 3. Strings, Vectors, and Arrays](ch03/README.md)
  - [Chapter 4. Expressions](ch04/README.md)
  - [Chapter 5. Statements](ch05/README.md)
  - [Chapter 6. Functions](ch06/README.md)
  - [Chapter 7. Classes](ch07/README.md)
- Part II: The C++ Library
  - [Chapter 8. The IO Library](ch08/README.md)
  - [Chapter 9. Sequential Containers](ch09/README.md)
  - [Chapter 10. Generic Algorithms](ch10/README.md)
  - [Chapter 11. Associative Containers](ch11/README.md)
  - [Chapter 12. Dynamic Memory](ch12/README.md)
- Part III: Tools for Class Authors
  - [Chapter 13. Copy Control](ch13/README.md)
  - [Chapter 14. Overloaded Operations and Conversions](ch14/README.md)
  - [Chapter 15. Object-Oriented Programming](ch15)
  - [Chapter 16. Templates and Generic Programming](ch16)
- Part IV:  Advanced Topics
  - [Chapter 17. Specialized Library Facilities](ch17)
  - [Chapter 18. Tools for Large Programs](ch18)
  - Chapter 19. Specialized Tools and Techniques
