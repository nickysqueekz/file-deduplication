ZFS Backup Deduplication and Consolidation
This repository provides a robust strategy and accompanying scripts for deduplicating and consolidating mixed file types, primarily photos, from multiple SMB backup shares onto a single ZFS mirrored archive dataset. The solution leverages czkawka_cli for high-performance duplicate detection and a PostgreSQL database for comprehensive inventory and incremental processing.

Table of Contents
Overview

Strategy

Prerequisites

Setup Guide

4.1. Prepare Environment and PostgreSQL Schema

4.2. Run Initial Czkawka Duplicate Scan (Stage 1)

4.3. Parse and Ingest into PostgreSQL

4.4. Consolidated Copying to ZFS Archive (Stage 2)

Automation

Scripts

ingest_duplicates.py

deduplicate_and_consolidate.py (Placeholder)

Troubleshooting

1. Overview
The primary goal is to merge approximately 7TB of mixed files (mostly photos) from various SMB-mounted backup shares into one consolidated ZFS mirror dataset, eliminating all duplicate files. This process is designed to be repeatable and incremental, ensuring efficient handling of new or changed files over time.

2. Strategy
The deduplication and consolidation is a two-stage process:

Stage 1: Internal Deduplication Across Backup Sources: Identify and remove identical files existing within the backup sources (/mnt/backup-*) before considering the master archive. This ensures that only unique content is processed for consolidation.

Tool: czkawka_cli is used for high-speed content-based duplicate detection.

Inventory: A PostgreSQL database stores metadata (path, hash, size, mtime) for all scanned files, enabling efficient querying and incremental processing.

Stage 2: Cross-Deduplication Against the Master Archive: Detect files in the backup sources that are already present in the ZFS archive (/mnt/archive). This prevents re-importing or duplicating data that has already been consolidated.

Consolidation: Unique files are copied to the ZFS archive. Duplicates are handled by creating hard links to the single canonical copy within the ZFS filesystem, saving disk space.

Logical Organization: Files in the archive are organized logically (e.g., by year/month for photos) rather than strictly preserving original share paths.

Why czkawka_cli and PostgreSQL?

czkawka_cli is chosen for its excellent performance on large datasets, multithreaded scanning, and ability to output results in JSON format, which is easily parsable for scripting.

PostgreSQL provides a robust, scalable, and queryable index for all scanned files, allowing for efficient change tracking, audit trails, and quick identification of duplicate groups. It eliminates the need to re-hash the entire 7TB on each run.

ZFS Configuration Note:
ZFS block-level deduplication is not recommended due to its extremely high RAM requirements (approximately 20 GB of RAM per TB of storage). Instead, file-level deduplication using hard links within the ZFS dataset achieves the same space-saving goal with significantly lower resource overhead. LZ4 compression is recommended for additional space savings with minimal CPU cost.

3. Prerequisites
Proxmox VE Host: Running your Proxmox server.

Linux VM (e.g., Linux Mint): A dedicated Linux VM for running the deduplication workflow. Installing on a VM is preferred over the PVE host for security, isolation, and system integrity. Linux Mint is fully compatible.

VM Resources: 2-4 vCPU, 4-8 GB RAM (adjust as needed for scanning speed).

Disk: No need for large internal storage; data remains on mounted shares.

SMB Backup Shares: Multiple SMB shares with your backup files (e.g., /mnt/backup-disk, /mnt/backup-disk1, etc.).

ZFS Archive Share: A ZFS mirrored dataset on your Proxmox host, exported via NFS or SMB, to serve as the consolidated archive (e.g., /mnt/archive).

PostgreSQL Server: An existing PostgreSQL database server available on the Proxmox host or accessible from the VM.

Required Packages on VM:

build-essential

cargo (for Czkawka)

cifs-utils (for mounting SMB shares)

postgresql-client (for psql and client libraries)

python3

python3-psycopg2 (Python library for PostgreSQL)

rustup (for managing Rust versions for Czkawka)

jq (for pretty-printing JSON in terminal)

4. Setup Guide
4.1. Prepare Environment and PostgreSQL Schema
4.1.1. Install Czkawka CLI and Rustup on Mint VM
First, install rustup to manage your Rust toolchain, which is required by czkawka_cli.

# Install rustup
curl https://sh.rustup.rs -sSf | sh
# When prompted: Type 'y' and press Enter to continue if there's an existing Rust installation warning.
# Choose default options.

# Restart your shell or source the env file to apply changes
source $HOME/.cargo/env

# Confirm Rust version (should be >= 1.85.0 for Czkawka 9.0.0)
rustc --version

# Install czkawka_cli using cargo
cargo install czkawka_cli

# Add cargo's bin directory to your PATH for easy execution
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

4.1.2. Create PostgreSQL Inventory Schema
Connect to your PostgreSQL server and create the files table and its indexes.

CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    full_path TEXT NOT NULL UNIQUE,
    hash TEXT,
    size BIGINT,
    mtime TIMESTAMP,
    scan_time TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_hash ON files(hash);
CREATE INDEX idx_mtime ON files(mtime);

4.1.3. Define Scan Targets (Mount All Shares)
Ensure all SMB backup shares and the ZFS archive are mounted on your Mint VM. It's recommended to use /etc/fstab for persistent mounts.

# Example for SMB shares (adjust for your specific share paths and credentials)
sudo mkdir -p /mnt/backup-disk /mnt/backup-disk1 /mnt/backup-disk2 /mnt/backup-root /mnt/backup-root2
sudo mount -t cifs //your_smb_server/share1 /mnt/backup-disk -o username=your_smb_user,password=your_smb_password,uid=$(id -u),gid=$(id -g),noperm,vers=3.0 # or vers=2.1, 2.0 depending on server
sudo mount -t cifs //your_smb_server/share2 /mnt/backup-disk1 -o username=your_smb_user,password=your_smb_password,uid=$(id -u),gid=$(id -g),noperm,vers=3.0
# Repeat for all your backup shares

# Example for ZFS archive (via NFS if shared from Proxmox host)
sudo mkdir -p /mnt/archive
sudo mount -t nfs your_proxmox_host_ip:/your_zfs_pool/archive /mnt/archive

Confirm mounts are successful:

ls /mnt/backup-disk
ls /mnt/archive

4.1.4. Set Up PostgreSQL Access
For passwordless CLI and script access, consider setting up a .pgpass file in your home directory:

echo "your_postgresql_host:your_postgresql_port:your_database_name:your_username:your_password" > ~/.pgpass
chmod 600 ~/.pgpass

4.2. Run Initial Czkawka Duplicate Scan (Stage 1)
This stage focuses on identifying and collapsing identical files that exist within the backup sources (/mnt/backup-*) before involving the master archive set.

4.2.1. Confirm Mount Points
Verify all shares are mounted correctly:

ls /mnt/backup-disk
ls /mnt/backup-disk1
ls /mnt/backup-disk2
ls /mnt/backup-root
ls /mnt/backup-root2
ls /mnt/archive

4.2.2. Run Czkawka Duplicate Scan (JSON Output)
This command scans the five backup locations for duplicates and excludes the master dataset in /mnt/archive from scanning.

# Create a dedicated directory for Czkawka output if it doesn't exist
mkdir -p /home/media/czkawka_output

czkawka_cli dup \
--directories /mnt/backup-disk \
--directories /mnt/backup-disk1 \
--directories /mnt/backup-disk2 \
--directories /mnt/backup-root \
--directories /mnt/backup-root2 \
--excluded-items /mnt/archive \
--file-to-save /home/media/czkawka_output/duplicates.json

Command Breakdown:

czkawka_cli dup: Executes the "duplicate finder" module of Czkawka CLI.

--directories: Specifies the directories to search for duplicates. Each directory must be listed with a separate --directories flag.

--excluded-items: Specifies directories or files to exclude; wildcards are not required for this version. In your case, /mnt/archive is excluded to prevent cross-contamination during the initial scan.

--file-to-save: Specifies the output file path where the results will be saved in JSON format.

This will output a JSON file (e.g., /home/media/czkawka_output/duplicates.json) with blocks like:

{
  "name": "Duplicates",
  "entries": [
    {
      "size": 123456,
      "hash": "abcdef...",
      "paths": ["/mnt/shares/share1/file.jpg", "/mnt/shares/share2/other/file_copy.jpg"]
    },
    ...
  ]
}

4.2.3. Confirm File Written
Inspect the JSON file for structure and sample duplicates:

head /home/media/czkawka_output/duplicates.json
jq . /home/media/czkawka_output/duplicates.json | less # if jq is installed

4.3. Parse and Ingest into PostgreSQL
This step involves writing a Python script to read the duplicates.json file and populate your PostgreSQL database.

4.3.1. Script: ingest_duplicates.py
Create a file named ingest_duplicates.py (e.g., in /home/media/dedupe_scripts/) with the following content. Remember to update the DB_NAME, DB_USER, DB_PASSWORD, and DB_HOST variables.

import json
import psycopg2
from datetime import datetime
import os

# --- Configuration ---
# IMPORTANT: Replace these with your actual PostgreSQL database credentials.
# For production, consider using environment variables or a separate config file
# for security instead of hardcoding passwords.
DB_NAME = "your_database_name"
DB_USER = "your_username"
DB_PASSWORD = "your_password"
DB_HOST = "localhost"  # Or the IP/hostname of your PostgreSQL server
DB_PORT = "5432"

CZKAWKA_OUTPUT_FILE = "/home/media/czkawka_output/duplicates.json" # Updated path

def ingest_data():
    conn = None
    try:
        conn = psycopg2.connect(
            dbname=DB_NAME,
            user=DB_USER,
            password=DB_PASSWORD,
            host=DB_HOST,
            port=DB_PORT
        )
        cur = conn.cursor()

        print(f"Reading data from {CZKAWKA_OUTPUT_FILE}...")
        with open(CZKAWKA_OUTPUT_FILE, 'r') as f:
            data = json.load(f)

        if "entries" not in data or not data["entries"]:
            print("No duplicate entries found or invalid JSON structure.")
            return

        total_files_processed = 0
        total_duplicates_found = 0

        for entry in data["entries"]:
            file_hash = entry.get("hash")
            file_size = entry.get("size")
            paths = entry.get("paths", [])

            if not file_hash or not paths:
                print(f"Skipping malformed entry: {entry}")
                continue

            for full_path in paths:
                total_files_processed += 1
                try:
                    mtime_timestamp = os.path.getmtime(full_path)
                    mtime_dt = datetime.fromtimestamp(mtime_timestamp)
                except FileNotFoundError:
                    print(f"Warning: File not found on disk: {full_path}. Skipping this entry.")
                    continue
                except Exception as e:
                    print(f"Error getting mtime for {full_path}: {e}. Skipping this entry.")
                    continue

                try:
                    cur.execute(
                        """
                        INSERT INTO files (full_path, hash, size, mtime, scan_time)
                        VALUES (%s, %s, %s, %s, NOW())
                        ON CONFLICT (full_path) DO UPDATE
                        SET hash = EXCLUDED.hash,
                            size = EXCLUDED.size,
                            mtime = EXCLUDED.mtime,
                            scan_time = NOW();
                        """,
                        (full_path, file_hash, file_size, mtime_dt)
                    )
                    if len(paths) > 1: # This file is part of a duplicate group
                        total_duplicates_found += 1

                except Exception as e:
                    print(f"Error inserting/updating {full_path}: {e}")
                    conn.rollback()

        conn.commit()
        print(f"Ingestion complete. Total files processed: {total_files_processed}. Duplicate files (based on Czkawka groups): {total_duplicates_found}.")

    except FileNotFoundError:
        print(f"Error: Czkawka output file not found at {CZKAWKA_OUTPUT_FILE}. Please ensure it exists.")
    except json.JSONDecodeError:
        print(f"Error: Invalid JSON in {CZKAWKA_OUTPUT_FILE}. Check file content.")
    except psycopg2.Error as e:
        print(f"PostgreSQL connection or query error: {e}")
    except Exception as e:
        print(f"An unexpected error occurred: {e}")
    finally:
        if conn:
            conn.close()

if __name__ == "__main__":
    ingest_data()

4.3.2. Run the Ingestion Script
python3 /home/media/dedupe_scripts/ingest_duplicates.py

4.3.3. Verify Data in PostgreSQL
Use psql to query your database and confirm successful ingestion:

# Connect to your database
psql -h your_postgresql_host -U your_username -d your_database_name

# Count all entries
SELECT COUNT(*) FROM files;

# List hashes with duplicates
SELECT hash, COUNT(*) AS num_duplicates
FROM files
GROUP BY hash
HAVING COUNT(*) > 1;

# View paths for a specific duplicate hash (replace with an actual hash)
SELECT full_path, size, mtime
FROM files
WHERE hash = 'your_duplicate_hash_here';

4.4. Consolidated Copying to ZFS Archive (Stage 2)
The next major phase involves writing a Python script to perform the actual deduplication and consolidation. This script will:

Query PostgreSQL: Identify unique files and duplicate groups from the files table.

Select Canonical Copy: For each group of duplicates, select one "master" or "canonical" file to be copied to the ZFS archive. This selection can be based on criteria like oldest mtime, shortest path, or a preferred source share.

Copy to ZFS Archive: Copy the canonical file to the appropriate logical location within /mnt/archive.

Handle Duplicates: For all other files in a duplicate group (those not chosen as canonical), create hard links within the ZFS archive pointing to the copied canonical file. If cross-filesystem linking were needed (unlikely here), symlinks would be used.

Update PostgreSQL: Mark files as "processed" or "consolidated" in the database, and potentially record which file was the canonical copy and which were hard-linked.

This script will be significantly more complex and will involve careful error handling and logging.

4.4.1. Space Optimization on ZFS
Enable compression (e.g., LZ4) on the ZFS archive dataset to save additional space with minimal CPU cost.

Do not recommend turning on ZFS's block-level deduplication as it is extremely RAM-intensive (on the order of 20 GB of RAM per TB of storage). For a 7 TB archive, this would imply ~140 GB just for the dedupe table, which is likely impractical. Instead, rely on the file-level deduplication achieved through hard links.

4.4.2. Cleanup (Optional) and Verification
Optional dedupe pass on archive: After copying, you may run an additional dedupe pass on /mnt/archive (e.g., rdfind -makehardlinks true /mnt/archive) to collapse any duplicates missed during the consolidation process. This can turn any remaining file copies into symlinks to a single copy.

Verify integrity: Always verify a sample of files to ensure integrity (compare checksums in DB vs. on disk).

Delete/Archive duplicates from original shares: Optionally, after copying, you may remove duplicates from the original SMB shares if space is needed. Always verify that checksums match before deleting or linking to avoid data loss.

5. Automation
Once the full deduplication and consolidation workflow is stable, it can be automated using a cron job or systemd timer.

The automated script should implement:

Incremental Logic: Query the PostgreSQL DB to skip files that have already been processed or haven't changed (based on mtime and size).

Logging: Record all actions (files scanned, new files found, duplicates deduplicated, errors) to a log file.

Resource Management: Consider scheduling during off-peak hours to minimize I/O impact.

Pseudocode for Automation Script:

# /home/media/dedupe_scripts/deduplicate_and_consolidate.py

import psycopg2
import os
import shutil # For copying files

# Configuration (DB credentials, ZFS archive path)
DB_NAME = "your_database_name"
DB_USER = "your_username"
DB_PASSWORD = "your_password"
DB_HOST = "localhost"
DB_PORT = "5432"
ZFS_ARCHIVE_ROOT = "/mnt/archive"

def consolidate_files():
    conn = None
    try:
        conn = psycopg2.connect(dbname=DB_NAME, user=DB_USER, password=DB_PASSWORD, host=DB_HOST, port=DB_PORT)
        cur = conn.cursor()

        # 1. Get all unique hashes (potential canonical files)
        #    AND all duplicate groups
        cur.execute("""
            SELECT hash, array_agg(full_path ORDER BY full_path) AS paths,
                   COUNT(*) AS path_count
            FROM files
            WHERE consolidated IS NULL -- Or some other status flag (you'll need to add this column to the 'files' table)
            GROUP BY hash
            HAVING COUNT(*) >= 1;
        """)
        hash_groups = cur.fetchall()

        print(f"Found {len(hash_groups)} unique hash groups to process.")

        for hash_entry in hash_groups:
            file_hash, paths, path_count = hash_entry

            # Select the canonical path (e.g., first one in the list, or shortest path)
            # You might want more sophisticated logic here (e.g., preferred source share)
            canonical_path = paths[0]
            
            # Determine logical target path in ZFS archive (e.g., based on file date/type)
            # This is crucial for organizing the archive.
            # Example: /mnt/archive/photos/2023/01/image.jpg
            # You'll need to parse canonical_path to extract relevant info (e.g., mtime for date)
            target_dir = os.path.join(ZFS_ARCHIVE_ROOT, "logical_category", "year", "month") 
            os.makedirs(target_dir, exist_ok=True)
            target_file_path = os.path.join(target_dir, os.path.basename(canonical_path))

            # 2. Copy Canonical File (if not already there)
            if not os.path.exists(target_file_path):
                print(f"Copying canonical: {canonical_path} to {target_file_path}")
                try:
                    shutil.copy2(canonical_path, target_file_path) # copy2 preserves metadata
                except Exception as e:
                    print(f"ERROR: Could not copy {canonical_path} to {target_file_path}: {e}")
                    # Log this and continue or mark for retry
                    continue
            else:
                print(f"Canonical target already exists: {target_file_path}. Skipping copy.")
            
            # 3. Create Hard Links for Duplicates (if path_count > 1)
            if path_count > 1:
                print(f"Processing duplicates for hash: {file_hash}")
                for dupe_path in paths:
                    if dupe_path != canonical_path:
                        # Decide where to link the duplicate:
                        # Option A: Create a hard link in the original *source* location if needed (less common for consolidation)
                        # Option B: Create a hard link in *another* logical location within ZFS if multi-context is desired.
                        # For this script, we'll assume Option B or simply mark as processed.
                        
                        # Assuming we're not touching original shares, but only managing the ZFS archive:
                        # You could also link to a "duplicates" folder within the archive, or just track in DB.
                        # For now, let's just confirm the original paths are accounted for in the DB.
                        print(f"  Duplicate path: {dupe_path} (will be accounted for by DB status)")

            # 4. Update DB (mark as consolidated)
            # You will need to add a 'consolidated' BOOLEAN column to your 'files' table,
            # and potentially a 'canonical_path_in_archive' TEXT column.
            # Example SQL to add:
            # ALTER TABLE files ADD COLUMN consolidated BOOLEAN DEFAULT FALSE;
            # ALTER TABLE files ADD COLUMN canonical_path_in_archive TEXT;
            cur.execute("""
                UPDATE files
                SET consolidated = TRUE,
                    canonical_path_in_archive = %s
                WHERE hash = %s;
            """, (target_file_path, file_hash))
            conn.commit()

        print("Consolidation process complete.")

    except psycopg2.Error as e:
        print(f"PostgreSQL error: {e}")
    except Exception as e:
        print(f"An unexpected error occurred: {e}")
    finally:
        if conn:
            conn.close()

if __name__ == "__main__":
    consolidate_files()

6. Scripts
ingest_duplicates.py
(Content provided in section 4.3.1)

deduplicate_and_consolidate.py (Placeholder)
(Content provided in section 5 - Pseudocode for Automation Script)

7. Troubleshooting
czkawka_cli errors:

error: cannot install package czkawka_cli ... it requires rustc 1.X.X or newer: Your Rust compiler version is too old. Use rustup update and source $HOME/.cargo/env to ensure you're using the latest rustup-managed version.

error: unexpected argument '--exclude': Use --excluded-items instead.

error: unexpected argument '--file-format': Use --file-to-save and --save-results-as json (though the latter might be implied or removed in your specific version; stick to --file-to-save with .json extension).

error: unexpected argument '/mnt/backup-disk1': The --directories flag is repeatable, not comma-separated. Use --directories <path1> --directories <path2>.

Excluded Items Warning: Wildcard * is required: While czkawka_cli 9.0.0 documentation might state that wildcards are not required for --excluded-items, previous versions or specific build behavior might still expect them. The latest confirmation suggests they are not needed for /mnt/archive.

Included Directory ERROR: Not found even one correct path to included which is required: This often happens if the specified --directories paths do not exist or are unreadable, or if --excluded-items is incorrectly used, causing all included directories to be considered excluded. Double-check mount points and path syntax.

/tmp cleared on reboot: Files in /tmp are temporary and will be deleted on reboot. Always save czkawka_cli output to a persistent location like /home/media/czkawka_output/.

PostgreSQL connection issues: Verify DB_NAME, DB_USER, DB_PASSWORD, DB_HOST, and DB_PORT in your Python script. Ensure the PostgreSQL server is running and accessible from your VM.

Permission issues: Ensure the user running the scripts has read permissions on all SMB shares and read/write permissions on the ZFS archive mount point. Check uid and gid in mount options for SMB.
