# PostgreSQL Backup — AI Lab

  Automated daily backup of the n8n PostgreSQL database.

  ## Setup

  - **Script:** `/opt/vps-security/postgres-backup.sh`
  - **Schedule:** Daily at 3:30 AM UTC (cron)
  - **Output:** `/opt/backups/postgres/n8n_YYYY-MM-DD.sql.gz`
  - **Retention:** 7 days (older files deleted automatically)
  - **Notification:** Telegram alert on success and failure

  ## How it works

  1. `docker exec` runs `pg_dump` inside the `n8n-postgres-1` container
  2. Output is piped through `gzip` to compress the backup
  3. Script verifies the file exists and is non-empty before reporting success
  4. Files older than 7 days are deleted via `find -mtime`

  ## Manual run

  ```bash
  bash /opt/vps-security/postgres-backup.sh

  Verify latest backup

  ls -lh /opt/backups/postgres/
  cat /var/log/vps-security/postgres-backup-$(date +%Y-%m-%d).log
