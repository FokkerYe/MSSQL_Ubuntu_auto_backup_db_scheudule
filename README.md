# MSSQL_Ubuntu_auto_backup_db_scheudule
# SQL Server Backup Script Setup and Configuration

## 1. Store Credentials in a Secure File

Create a directory to store your credentials securely.

```bash
mkdir -p ~/.db_credentials
```
## 2. Set the directory permissions to restrict access to the credentials.
```
chmod 700 ~/.db_credentials
```


## 3. Create a hidden file to store your credentials.
```
nano ~/.db_credentials/.mssql_credentials
```

### Add your credentials to the file in the following format:
```
SQLCMD_USER="your_username"
SQLCMD_PASSWORD="your_password"
```

## Save and close the file, then set its permissions to ensure only you can read it.
```
chmod 600 ~/.db_credentials/.mssql_credentials
```
## 2. Create Backup Directory and Set Permissions

## Create the backup directory.
```
sudo mkdir -p /var/opt/mssql/backups/full
```
### Change the ownership of the backup directory.
```
sudo chown -R mssql:mssql /var/opt/mssql/backups
```
## Set the appropriate permissions for the backup directory.
```
sudo chmod 770 -R /var/opt/mssql/backups
```
### 3. Create Script Directory

### Create a directory for your backup script and navigate to it.
```
mkdir autoschedule
cd autoschedule
```
## 4. Create and Edit Backup Script

### Create and edit the backup script file.
```
nano full_backup.sh
```
### Paste the following script into the full_backup.sh file:
```
#!/bin/bash

# Source the credentials
source ~/.db_credentials/.mssql_credentials

# Variables
BACKUP_DIR="/var/opt/mssql/backups"
FULL_BACKUP_DIR="$BACKUP_DIR/full"
DATABASES=("TestDB")
RETENTION_DAYS=1

# Function to check if a database exists
database_exists() {
  echo "Checking if database '$1' exists..."
  /opt/mssql-tools/bin/sqlcmd -S your_private_ip -U $SQLCMD_USER -P $SQLCMD_PASSWORD -Q "IF DB_ID('$1') IS NOT NULL PRINT 'EXISTS'" | grep -q EXISTS
}

# Function to create a directory if it doesn't exist
create_directory() {
  [ ! -d "$1" ] && mkdir -p "$1"
}

# Function to delete old backups (Only keep last 1 day's backups)
cleanup_old_backups() {
  echo "Deleting backups older than $RETENTION_DAYS days..."
  find "$1" -type f -name "*.bak" -mtime +$RETENTION_DAYS -exec rm -f {} \; -exec echo "Deleted: {}" >> /var/log/mssql_backup_cleanup.log \;
  echo "Cleanup complete."
}

# Create backup directories if they don't exist
create_directory "$FULL_BACKUP_DIR"

# Clean up old backups
cleanup_old_backups "$FULL_BACKUP_DIR"

# Create full backups
for DATABASE_NAME in "${DATABASES[@]}"; do
  if database_exists "$DATABASE_NAME"; then
    FULL_BACKUP_FILE="$FULL_BACKUP_DIR/$DATABASE_NAME-full-$(date +%Y-%m-%d_%H-%M-%S).bak"
    echo "Starting full backup for database '$DATABASE_NAME'..."
    /opt/mssql-tools/bin/sqlcmd -S your_private_ip -U $SQLCMD_USER -P $SQLCMD_PASSWORD -Q "BACKUP DATABASE [$DATABASE_NAME] TO DISK='$FULL_BACKUP_FILE' WITH FORMAT, INIT, NAME='$DATABASE_NAME - Full Backup', SKIP, NOREWIND, NOUNLOAD, STATS=10"
    echo "Full backup of database '$DATABASE_NAME' completed successfully."
  else
    echo "Database '$DATABASE_NAME' does not exist. Skipping backup."
  fi
done
```
## Save and close the file.

### Make the script executable.
```
chmod +x full_backup.sh
```
### 5. Schedule the Backup Script

### You can schedule the script to run automatically by adding it to cron. To open the cron file for editing, run:
```
crontab -e
```
### Then, add a cron job to run the backup script at your desired interval (e.g., daily at midnight).
```
0 0 * * * /path/to/your/script/full_backup.sh
```
## Save and exit the crontab editor.
