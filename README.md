# mynas

```shell
lsblk

cryptsetup -v luksOpen /dev/sdb1 kdata_crypt-0
mount -o subvolid=5 /dev/mapper/kdata_crypt-0 /kdata

echo 'kdata_crypt-0 UUID=6de57bb7-481e-4162-a765-884d3c4d5128 none luks' >> /etc/crypttab
echo '/dev/disk/by-id/dm-name-kdata_crypt-0 /kdata btrfs defaults 0 1' >> /etc/fstab

```

create /root/snapshot.sh with the following content:

```shell
#!/bin/bash
set -Eeo pipefail

if [[ $EUID -ne 0 ]]; then
  echo "This script must be run as root"
  exit 1
fi

local_snapshot() {
    echo "local_snapshot..."
    find "/.snapshots/home/" -maxdepth 1 -iname "auto-*" | while read file; do
      timestamp=${file#*-}
      let "tDiff=$(date +%s)-$timestamp"
      if [[ "$tDiff" -ge 86400 ]]; then
        btrfs subvol delete "$file"
      fi
    done
    btrfs subvol snapshot -r /home /.snapshots/home/auto-$(date +%s)
}

remote_snapshot() {
    echo "remote_snapshot..."
    find "/kdata/backup/" -maxdepth 1 -iname "home-*" | while read file; do
      timestamp=${file#*-}
      let "tDiff=$(date +%s)-$timestamp"
      if [[ "$tDiff" -ge 864000 ]]; then
        btrfs subvol delete "$file"
      fi
    done
    btrfs subvol snapshot -r /home /.snapshots/home/home-new && sync && \
    btrfs send -p /.snapshots/home/home-base /.snapshots/home/home-new | btrfs receive /kdata/backup && \
    btrfs subvol delete /.snapshots/home/home-base && \
    mv /.snapshots/home/home-new /.snapshots/home/home-base && \
    mv /kdata/backup/home-new /kdata/backup/home-$(stat -c %Y "/kdata/backup/home-new")
}

if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <local|remote>"
    exit 1
fi

MODE=$1
case $MODE in
    local)
        local_snapshot
        ;;
    remote)
        remote_snapshot
        ;;
    *)
        echo "Invalid parameter: $MODE. Accepts only 'local' or 'remote'."
        exit 1
        ;;
esac

exit 0
```