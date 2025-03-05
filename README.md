#!/bin/bash

# --- Configuration ---
SOURCE_PORT=4432
TARGET_PORT=5432
DATABASE_NAME="CXO_dashboard"
BACKUP_DIR="$HOME/Desktop/pg_backup"
USERNAME="your_username" # Replace with your PostgreSQL username
#PASSWORD="your_password" # REMOVE THIS LINE!  Use environment variable instead.

# --- Check if password environment variable is set ---
if [ -z "$PGPASSWORD" ]; then
  echo "Error: PGPASSWORD environment variable not set."
  echo "Please set the PGPASSWORD environment variable before running this script."
  exit 1
fi

# --- Create Backup Directory if it doesn't exist ---
mkdir -p "$BACKUP_DIR"

# --- Generate Timestamp for Backup Filename ---
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DATABASE_NAME}_${TIMESTAMP}.sql"

# --- Dump the Database from the Source Instance ---
echo "Dumping database from source instance (port $SOURCE_PORT)..."
pg_dump -U "$USERNAME" -p "$SOURCE_PORT" -c -f "$BACKUP_FILE" "$DATABASE_NAME"
if [ $? -ne 0 ]; then
  echo "Error: pg_dump failed.  Please check your source database connection and credentials."
  exit 1
fi

# --- Drop the Database on the Target Instance if it Exists ---
echo "Dropping database on target instance (port $TARGET_PORT) if it exists..."
psql -U "$USERNAME" -p "$TARGET_PORT" -c "DROP DATABASE IF EXISTS \"$DATABASE_NAME\";"
if [ $? -ne 0 ]; then
  echo "Error: Failed to drop the database.  Check your target database connection and credentials."
  exit 1
fi

# --- Create the Database on the Target Instance ---
echo "Creating database on target instance (port $TARGET_PORT)..."
psql -U "$USERNAME" -p "$TARGET_PORT" -c "CREATE DATABASE \"$DATABASE_NAME\";"
if [ $? -ne 0 ]; then
  echo "Error: Failed to create the database.  Check your target database connection and credentials."
  exit 1
fi

# --- Restore the Database on the Target Instance ---
echo "Restoring database on target instance (port $TARGET_PORT)..."
psql -U "$USERNAME" -p "$TARGET_PORT" -d "$DATABASE_NAME" -f "$BACKUP_FILE"
if [ $? -ne 0 ]; then
  echo "Error: Failed to restore the database. Check your target database connection and credentials."
  exit 1
fi

echo "Database copy and backup completed successfully!"
echo "Backup file: $BACKUP_FILE"

exit 0
