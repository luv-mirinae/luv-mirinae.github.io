---
title: Ubuntu 환경에서 tar로 전체 시스템 백업하기
layout: page
parent: Linux
---

# Ubuntu 환경에서 tar로 전체 시스템 백업하기

---

아래 스크립트는 /media/mirinae/backup 디렉토리에 backup-YYYYmmdd_HHMMSS.tar.gz 백업 파일을 생성하는 스크립트입니다.

```bash
#!/bin/bash

# /usr/local/bin/backup.sh

CUR_DATE=`date "+%Y%m%d_%H%M%S"`
FILE_NAME=`echo backup-"$CUR_DATE".tar.gz`
FILE_DIR="/media/mirinae/backup/"

if [ "$EUID" -ne 0 ]; then
        echo "  Please run as root."
        echo "  Usage: sudo backup.sh"
        exit 0
fi

echo
echo "  Waiting for backup..."
echo "  Backup file name is: $FILE_NAME"
echo
sleep 3

cd /
sudo tar -cvpzf $FILE_NAME \
        --exclude=/$FILE_NAME \
        --exclude=/proc \
        --exclude=/tmp \
        --exclude=/mnt \
        --exclude=/dev \
        --exclude=/sys \
        --exclude=/run \
        --exclude=/media \
        --exclude=/var/cache/apt/archives \
        --exclude=/usr/src/linux-headers* \
        --exclude=/home/*/.gvfs \
        --exclude=/home/*/.cache \
        --exclude=/home/*/.local/share/Trash \
        /

sudo mv /$FILE_NAME $FILE_DIR

if [ -f "$FILE_DIR$FILE_NAME" ]; then
        echo "  Backup is completed successfully."
        echo "  Location: $FILE_DIR$FILE_NAME"
        exit 1
else
        echo "  Backup failed."
        exit 0
fi
```

아래 스크립트는 /media/mirinae/backup 디렉토리에서 위 backup.sh 스크립트로 백업한 파일을 찾아 복원하는 스크립트입니다.

```bash
#!/bin/bash

# /usr/local/bin/restore.sh

WORK_DIR="/media/mirinae/backup/"
RESTORE_DIR="/"

if [ "$EUID" -ne 0 ]; then
        echo "  Please run as root."
        echo "  Usage: sudo restore.sh <file_name>"
        exit 0
fi

if [ $# -ne 1 ]; then
        echo "  Usage: sudo restore.sh <file_name>"
        exit 0
fi

FILE_NAME=$1

echo $FILE_NAME

if [ -f $FILE_NAME ]; then
        echo
        echo "  Waiting for restore..."
        echo "  File: $FILE_NAME"
        echo
        sleep 3
        cd $WORK_DIR
        tar -xvpzf $FILE_NAME -C $RESTORE_DIR --numeric-owner
        mkdir /proc /tmp /mnt /dev /sys /run /media
        echo
        echo "  Restore is completed successfully."
else
        echo
        echo "  File not found."
        exit 0
fi
```

backup.sh 스크립트를 crontab에 등록하여 자동으로 백업이 수행되게 할 수 있습니다.

```bash
sudo crontab -e
```

```bash
# 매주 월요일 오전 4시에 전체 백업 수행
0 4 * * 1 /usr/local/bin/backup.sh >> /var/log/tar-backup.log 2>&1
```
