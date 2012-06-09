---
layout: post
title: "Backuping and restoring PostgreSQL databases"
date: "2011-02-04 23:48:55"
comments: true
categories: postgresql bash unix backup restore database db
---

Recently I had a deal with backuping data from PostreSQL database. So I want to share two scripts I created, one to backup and one to restore databases.

The first one is for backup:

```bash backup.sh

#!/bin/sh

BASE_BACKUP_DIR="/backups/postgres/"

timeslot=$(date +"%F_%Hh%Mm%Ss")
backup_dir="${BASE_BACKUP_DIR}/${timeslot}"
databases=$(sudo -u postgres sh -c "psql  -c '\l'" 2> /dev/null | sed \
-n 4,/\eof/p | grep -v rows\) | awk '{print $1}' | egrep '^\w')

mkdir -p $backup_dir

for db in $databases; do
    sudo -u postgres sh -c "pg_dump -C ${db}" 2> /dev/null | xz -9e > $backup_dir/$db.xz
done

```

After executing script you'll have all you database dumps in `/backups/postgres/date_time` directory compressed with `xz`. Sure instead of `xz` you can use `gzip` or `bzip2`.

Okay. Now let's take a look at the restore script:

```bash restore.sh

#!/bin/sh

if [ $# -lt 1 ] ; then
    echo "Error: You should specify backup directory path" 1>&2
    exit 1
fi
backup_dir=$1
if [ ! -d $backup_dir ]; then
    echo "Error: directory ${backup_dir} does not exist" 1>&2
    exit 1
fi

for db_xz in $backup_dir/*.xz; do
    db_name=$(echo $db_xz | sed 's/\.xz$//; s/^.*\///g')
    sudo -u postgres sh -c "xz -cd ${db_xz} | psql"
done

```

It takes one parameter: a path to back directory with *.xz archives.

Both scripts should be executed with with root permissions, cause they use `sudo` to execute `pg_dump` and `pg_restore` commands with `postgres` user permissions. It allows avoiding asking a password for database. It's not the only way, but I decided to use this one.

