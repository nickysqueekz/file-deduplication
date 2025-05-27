# ZFS Backup Deduplication and Consolidation

This repository provides a robust strategy and accompanying scripts for deduplicating and consolidating mixed file types, primarily photos, from multiple SMB backup shares onto a single ZFS mirrored archive dataset. The solution leverages `czkawka_cli` for high-performance duplicate detection and a PostgreSQL database for comprehensive inventory and incremental processing.

## Table of Contents

1.  [Overview](#overview)
2.  [Strategy](#strategy)
3.  [Prerequisites](#prerequisites)
4.  [Setup Guide](#setup-guide)
    * [1. Prepare Environment and PostgreSQL Schema](#1-prepare-environment-and-postgresql-schema)
    * [2. Run Initial Czkawka Duplicate Scan](#2-run-initial-czkawka-duplicate-scan)
    * [3. Ingest Czkawka Output into PostgreSQL](#3-ingest-czkawka-output-into-postgresql)
    * [4. Develop Consolidation Logic (Next Step)](#4-develop-consolidation-logic-next-step)
5.  [Automation](#automation)
6.  [Scripts](#scripts)
    * [`ingest_duplicates.py`](#ingest_duplicatespy)
    * [`deduplicate_and_consolidate.py` (Placeholder)](#deduplicate_and_consolidatepy-placeholder)
7.  [Troubleshooting](#troubleshooting)

## Overview

The goal is to merge approximately 7TB of mixed files (mostly photos) from various SMB-mounted backup shares into a single ZFS mirror dataset, eliminating all duplicate files. This process is designed to be repeatable and incremental, ensuring efficient handling of new or changed files over time.

## Strategy

The deduplication and consolidation is a two-stage process:

1.  **Stage 1: Internal Deduplication Across Backup Sources:** Identify and remove identical files existing within the backup sources (`/mnt/backup-*`) before considering the master archive. This ensures that only unique content is processed for consolidation.
    * **Tool:** `czkawka_cli` is used for high-speed content-based duplicate detection.
    * **Inventory:** A PostgreSQL database stores metadata (path, hash, size, mtime) for all scanned files, enabling efficient querying and incremental processing.
2.  **Stage 2: Cross-Deduplication Against the Master Archive:** Detect files in the backup sources that are already present in the ZFS archive (`/mnt/archive`). This prevents re-importing or duplicating data that has already been consolidated.
    * **Consolidation:** Unique files are copied to the ZFS archive. Duplicates are handled by creating hard links to the single canonical copy within the ZFS filesystem, saving disk space.
    * **Logical Organization:** Files in the archive are organized logically (e.g., by year/month for photos) rather than strictly preserving original share paths.

**Why `czkawka_cli` and PostgreSQL?**

* `czkawka_cli` is chosen for its excellent performance on large datasets, multithreaded scanning, and ability to output results in JSON format, which is easily parsable for scripting.
* PostgreSQL provides a robust, scalable, and queryable index for all scanned files, allowing for efficient change tracking, audit trails, and quick identification of duplicate groups. It eliminates the need to re-hash the entire 7TB on each run.

**ZFS Configuration Note:**
ZFS block-level deduplication is *not* recommended due to its extremely high RAM requirements (approximately 20 GB of RAM per TB of storage). Instead, file-level deduplication using hard links within the ZFS dataset achieves the same space-saving goal with significantly lower resource overhead. LZ4 compression is recommended for additional space savings with minimal CPU cost.

## Prerequisites

* **Proxmox VE Host:** Running your Proxmox server.
* **Linux VM (e.g., Linux Mint):** A dedicated Linux VM for running the deduplication workflow. Installing on a VM is preferred over the PVE host for security, isolation, and system integrity. Linux Mint is fully compatible.
    * **VM Resources:** 2-4 vCPU, 4-8 GB RAM (adjust as needed for scanning speed).
    * **Disk:** No need for large internal storage; data remains on mounted shares.
* **SMB Backup Shares:** Multiple SMB shares with your backup files (e.g., `/mnt/backup-disk`, `/mnt/backup-disk1`, etc.).
* **ZFS Archive Share:** A ZFS mirrored dataset on your Proxmox host, exported via NFS or SMB, to serve as the consolidated archive (e.g., `/mnt/archive`).
* **PostgreSQL Server:** An existing PostgreSQL database server available on the Proxmox host or accessible from the VM.
* **Required Packages on VM:**
    * `build-essential`
    * `cargo` (for Czkawka)
    * `cifs-utils` (for mounting SMB shares)
    * `postgresql-client` (for psql and client libraries)
    * `python3`
    * `python3-psycopg2` (Python library for PostgreSQL)
    * `rustup` (for managing Rust versions for Czkawka)
    * `jq` (for pretty-printing JSON in terminal)

## Setup Guide

### 1. Prepare Environment and PostgreSQL Schema

#### 1.1 Install Czkawka CLI and Rustup on Mint VM

First, install `rustup` to manage your Rust toolchain, which is required by `czkawka_cli`.

```bash
# Install rustup
curl [https://sh.rustup.rs](https://sh.rustup.rs) -sSf | sh
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
