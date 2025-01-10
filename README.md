# Speedy-Bcache-on-Proxmox
Under Proxmox's condition, this serves a guidance on what to do to truely speedup a storage space for vm to use

# Complete Bcache Setup Guide

## 1. Prerequisites

Install required packages:
```bash
apt update
apt install bcache-tools mdadm 
# zfs is already there, no need to install
```

## 2. Create RAID Array

Create a RAID1 array from two drives: (this is the slow part of the final product, I opt for mirror to reduce the lighthood to fail, but single point of failure remains at SSD and writeback, so a good UPS and rsync backup is required)
```bash
# First, clear any existing filesystems
wipefs -a /dev/sda  # check your drive
wipefs -a /dev/sdb

# Create the RAID1 array
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb
```

## 3. Setup Bcache

First, clear any existing bcache signatures:
```bash
# Clear cache device
wipefs -a /dev/nvme2n1  # my ssd is at nvme2n1
# Clear backing device if needed
wipefs -a /dev/md0
```

Create bcache devices:
```bash
# Create cache device
make-bcache -C /dev/nvme2n1

# Get cache UUID
CACHE_UUID=$(bcache-super-show /dev/nvme2n1 | grep cset.uuid | awk '{print $2}')
echo "Cache UUID: $CACHE_UUID"

# Create backing device
make-bcache -B /dev/md0
```

## 4. Register and Attach Devices

Register both devices:
```bash
echo /dev/nvme2n1 > /sys/fs/bcache/register
echo /dev/md0 > /sys/fs/bcache/register
```

Make sure cache is attached:
```bash
echo $CACHE_UUID > /sys/block/bcache0/bcache/attach
```

## 5. Configure Bcache Parameters

Set optimal parameters: We will set them properly in the config files in the persistent section, this is a playground for you to tune
```bash
# Set cache mode to writeback
echo writeback > /sys/block/bcache0/bcache/cache_mode

# Disable sequential cutoff
echo 0 > /sys/fs/bcache/$CACHE_UUID/bdev0/sequential_cutoff

# Set readahead
echo 8192 > /sys/block/bcache0/bcache/readahead
blockdev --setra 8192 /dev/bcache0

# Set congestion thresholds
echo 40000 > /sys/fs/bcache/$CACHE_UUID/congested_write_threshold_us
echo 10000 > /sys/fs/bcache/$CACHE_UUID/congested_read_threshold_us
```

## 6. Create ZFS Pool

Create ZFS pool on bcache device:
```bash
zpool create tank /dev/bcache0
```
The you can use GUI to add this ZFS under Data Center's Storage Section

## 7. Make Configuration Persistent

1. Add bcache module to load at boot:
```bash
echo "bcache" | tee /etc/modules-load.d/bcache.conf
```

2. Create bcache settings script (see next bcache-settings.sh)

3. Create systemd service (see next bcache-settings.service)

4. Enable the service:
```bash
chmod +x /usr/local/bin/bcache-settings.sh
systemctl enable bcache-settings
systemctl start bcache-settings
```

## 8. Verification

Check bcache status:
```bash
# Check if devices are properly registered
ls -l /sys/fs/bcache/

# Check cache mode
cat /sys/block/bcache0/bcache/cache_mode

# Check if ZFS pool is mounted
zpool status tank
```

Monitor bcache performance:
```bash
#!/bin/bash
while true; do
    clear
    echo "=== Bcache Monitor ==="
    echo "Time: $(date '+%H:%M:%S')"
    echo
    echo "Cache Stats:"
    echo "------------"
    for uuid in /sys/fs/bcache/*-*-*-*-*; do
        if [ -d "$uuid" ]; then
            echo "Cache Set: $(basename $uuid)"
            echo "Hits:    $(cat $uuid/stats_total/cache_hits)"
            echo "Misses:  $(cat $uuid/stats_total/cache_misses)"
            echo "Bypass:  $(cat $uuid/stats_total/cache_bypass_hits)"
            echo "Dirty:   $(cat $uuid/stats_total/dirty_data)"
        fi
    done
    
    echo
    echo "Device Status:"
    echo "--------------"
    for dev in /sys/block/bcache*; do
        if [ -d "$dev" ]; then
            dev_name=$(basename $dev)
            echo "Device: $dev_name"
            echo "State:  $(cat $dev/bcache/state)"
            echo "Mode:   $(cat $dev/bcache/cache_mode)"
        fi
    done
    
    sleep 2
done
```

## 9. Common Issues and Troubleshooting

1. If bcache device doesn't appear after reboot:
   - Check if bcache module is loaded: `lsmod | grep bcache`
   - Manually register devices using the register commands in step 4

2. If cache isn't attaching:
   - Verify UUID is correct: `bcache-super-show /dev/nvme2n1`
   - Manually attach using the attach command in step 4

3. If ZFS pool doesn't import automatically:
   - Manual import: `zpool import tank`
   - Check service logs: `journalctl -u bcache-settings`

## 10. Performance Testing

Test bcache performance:
```bash
# Sequential write test
fio --name=seq-write \
    --directory=/tank/test \
    --ioengine=posixaio \
    --rw=write \
    --bs=1m \
    --direct=1 \
    --size=4g \
    --numjobs=1 \
    --runtime=60 \
    --group_reporting

# Mixed random read/write test
fio --name=mixed-randread-write \
    --directory=/tank/test \
    --ioengine=posixaio \
    --rw=randrw \
    --bs=4k \
    --direct=1 \
    --size=4g \
    --numjobs=4 \
    --runtime=60 \
    --rwmixread=70 \
    --group_reporting
```

# Bcache Configuration Files (To make it reboot safe)

## 1. Systemd Service File
Create the service file at `/etc/systemd/system/bcache-settings.service`:

```ini
[Unit]
Description=Configure bcache settings at boot
After=local-fs.target
After=systemd-modules-load.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/sleep 15
ExecStart=/usr/local/bin/bcache-settings.sh

[Install]
WantedBy=multi-user.target
```

## 2. Bcache Settings Script
Create the script file at `/usr/local/bin/bcache-settings.sh`:

```bash
#!/bin/bash
echo 40000 > /sys/fs/bcache/058324af-f00d-4cf0-a3b6-c7a381d1c33f/congested_write_threshold_us
echo 10000 > /sys/fs/bcache/058324af-f00d-4cf0-a3b6-c7a381d1c33f/congested_read_threshold_us
echo 0 > /sys/fs/bcache/058324af-f00d-4cf0-a3b6-c7a381d1c33f/bdev0/sequential_cutoff

# Set writeback parameters
echo 50 > /sys/block/bcache0/bcache/writeback_percent
echo 30 > /sys/block/bcache0/bcache/writeback_delay

sudo blockdev --setra 8192 /dev/bcache0
```

## 3. Setup Instructions

1. Save both files in their respective locations
2. Make the script executable:
```bash
chmod +x /usr/local/bin/bcache-settings.sh
```

3. Enable and start the service:
```bash
systemctl enable bcache-settings
systemctl start bcache-settings
```

4. Verify the service status:
```bash
systemctl status bcache-settings
```

## 4. Additional Configuration

Add bcache module to load at boot:
```bash
echo "bcache" | tee /etc/modules-load.d/bcache.conf
```

## 5. Verification Commands

Check if settings are applied:
```bash
# Check cache mode
cat /sys/block/bcache0/bcache/cache_mode

# Check readahead
cat /sys/block/bcache0/bcache/readahead

# Check sequential cutoff
cat /sys/fs/bcache/${CACHE_UUID}/bdev0/sequential_cutoff

# Check congestion thresholds
cat /sys/fs/bcache/${CACHE_UUID}/congested_write_threshold_us
cat /sys/fs/bcache/${CACHE_UUID}/congested_read_threshold_us
```
