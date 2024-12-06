# Shell Scripts - Server Maintenance Scripts

## Overview

Shell scripts containing useful Linux commands executed in sequence for maintaining certain aspects of the server and projects.

## Usage

The scripts executed from server terminal. Any necessary super user privileges are handled inside scripts to remove the need of `sudo` command.

### System Backup

Creates a backup of the system as an image file, shrinks it and moves to the backup drive.

```shell
#!/bin/bash

backup_date=$(date +"%d-%m-%Y")
backup_name=$(date +"H1V3_Image-Backup_%Y-%m-%d.img")
mount_state=$(bash BackupUSB_Mount.sh)
mount_path="{BACKUP_PATH}"

# Get last backup date
last_backup=$(/usr/bin/ls $mount_path/*H1V3_Backup* -Art | tail -n 1)
last_backup_raw=$(echo "$last_backup" | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}')
last_backup_date=$(echo "$last_backup_raw" | awk -F- '{print $2"-"$3"-"$1}')
echo "Last backup was: $last_backup_date"

# Check if backup drive mounted
if [ "$mount_state" == "Backup USB is already mounted at $mount_path." ] || [ "$mount_state" == "Backup USB mounted successfully at $mount_path." ]; then
    echo "Backup USB ready, beginning backup..."
    # Create copy of system image
    sudo dd if=/dev/mmcblk0 of={BACKUP_PATH}$backup_name bs=1M status=progress
    
    echo "-- Shrinking Backup --"
    # Shrink image file to save space
    sudo pishrink.sh -z {BACKUP_PATH}$backup_name
    sleep 5
    
    echo "Backup Complete"
elif [ "$mount_state" == "Failed to mount the Backup USB." ]; then
    echo "!! Backup USB failed to mount, canceling backup !!"
else
    echo "Unexpected output: $mount_state"
fi
```

### Docker Backup

Creates a backup of  running Docker containers. Moves to Backup drive if available.

```shell
backup_date=$(date +%Y-%m-%d)
backup_name=$"Docker_General_Backup_$backup_date.tar"
backup_dir={BACKUP_PATH}

echo "Backing Up Docker Running Containers... - $backup_name"
# Loop through all running containers
for container in $(docker ps -q); do
    # Get container name
    container_name=$(docker inspect --format='{{.Name}}' $container | cut -c2-)
    echo "- Backing Up: $container_name -"
    # Commit the container to an image
    sudo docker commit -p $container ${container_name}_backup
    # Save the image to a tar file
    sudo docker save -o $backupDir/${container_name}_Backup_$backup_date.tar ${container_name}_backup
    # Remove the temporary image
    sudo docker rmi ${container_name}_backup
done
sleep 5

# Compress all tar files into a single archive
sudo tar -cJf $backup_dir/$backup_name.xz -C $backup_dir .

# Mount Backup Storages
sudo BackupUSB_Mount.sh

# Move to available Backup Storage
if grep -qs "{BACKUP_PATH}" /proc/mounts; then
    echo "Moving to Backup USB..."
    sudo mv {BACKUP_PATH}$backup_name {BACKUP_PATH}server
    echo "Moved to Backup USB"
else
    echo "No backup storage mounted. Docker Backup stored locally."

echo " - Docker Backup Complete - "
```

### OliveTin Refresh

Generates and overwrites OliveTin configuration file with scripts inside the pre-defined scripts directory

```shell
#!/bin/bash

# Define the directory containing the scripts
SCRIPTS_DIR="{SCRIPTS_DIRECTORY}"
# Define the output YAML file
OUTPUT_FILE="/etc/OliveTin/config.yaml"

# Start the actions list
cat OliveTin_base.txt > $OUTPUT_FILE

# Loop over the scripts in the directory
for SCRIPT in $SCRIPTS_DIR/*.sh; do
  # Get the filename without the directory
  FILENAME=$(basename $SCRIPT)
  # Remove the .sh extension
  FILENAME=${FILENAME%.sh}
  # Replace underscores with spaces in the filename for the title
  TITLE=${FILENAME//_/ }
  # Capitalize the second word
  TITLE=$(echo $TITLE | awk '{for(i=1;i<=NF;i++) {if (i == 2) {$i=toupper(substr($i,1,1)) substr($i,2)}; printf "%s ", $i}; printf "\n"}')
  ICON="${TITLE%% *}"

  # Add the script to the actions list
  echo "  - title: \"$TITLE\""                            >> $OUTPUT_FILE
  echo "    shell: .$SCRIPT"                              >> $OUTPUT_FILE
  echo "    popupOnStart: $POPUP"                         >> $OUTPUT_FILE
  echo "    maxConcurrent: 1"                             >> $OUTPUT_FILE
done

#make all scripts executable
sudo chmod +x .$SCRIPTS_DIR/*

#restart OliveTin
sudo systemctl restart OliveTin

exit 0
```

[Other projects on H1V3](../README.md)
