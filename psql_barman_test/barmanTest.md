# Test Connection to Psql

##### Replace All IP Addresses in Document
```
Current Test:
- db_name: pg_vssh
- db_ip: 172.24.130.192
- bms: 172.24.139.61
```

```
sudo su - barman
psql -c 'SELECT version()' -U barman -d postgres -h 172.24.130.192 -p 5432
```

##### Test Barman
```
barman check pg_vssh

barman receive-wal --drop-slot pg_vssh
barman receive-wal --create-slot pg_vssh
barman receive-wal --reset pg_vssh
barman switch-wal --force --archive pg_vssh

# barman switch-wal --force --archive pg_vssh
# barman switch-wal --archive --archive-timeout 20 pg_vssh
barman switch-wal --archive pg_vssh
barman check pg_vssh
barman replication-status pg_vssh
barman show-server pg_vssh
barman list-server
barman list-backup all

ls -ltr /var/lib/barman/*/incoming
ls -ltr /var/lib/barman/*/errors
ls -ltr /var/lib/barman/*/identity.json
ls -ltr /var/lib/barman/*/base
ls -ltr /var/lib/barman/*/streaming
ls -ltr /var/lib/barman/*/wals

ls -ltr /var/lib/barman/*/incoming
ls -ltr /var/lib/barman/*/streaming
ls -ltr /var/lib/barman/*/wals


# barman backup pg_vssh --wait

tail -100 /var/log/barman/barman.log
```


# Backup & Recovery Demo
##### Add Table testtablev? to Psql & Run Backup Barman
```
# run on postresql vm
DROP TABLE testtablev2;
CREATE TABLE testtablev4 ( name varchar, age int, joindate varchar );
INSERT INTO testtablev4 (name, age, joindate)
SELECT substr(md5(random()::text), 1, 10),
       (random() * 70 + 10)::integer,
       DATE '2018-01-01' + (random() * 700)::integer
FROM generate_series(1, 1000);
SELECT * FROM testtablev4 LIMIT 10;
\d

# run on barman vm
barman backup pg_vssh --wait

ls -ltr /var/lib/barman/pg_vssh/incoming 
ls -ltr /var/lib/barman/pg_vssh/base
ls -ltr /var/lib/barman/pg_vssh/streaming
ls -ltr /var/lib/barman/pg_vssh/wals

rm -rf /var/lib/barman/pg_vssh/incoming/*
rm -rf /var/lib/barman/pg_vssh/base/*
rm -rf /var/lib/barman/pg_vssh/streaming/*
rm -rf /var/lib/barman/pg_vssh/wals/*


# Scenario
# Time T1: pg_vssh 20220426T120313 - Tue Apr 26 12:03:16 2022
# run on postresql vm
CREATE TABLE testtablev1 ( name varchar, age int, joindate varchar );
INSERT INTO testtablev1 (name, age, joindate)
SELECT substr(md5(random()::text), 1, 10),
       (random() * 70 + 10)::integer,
       DATE '2018-01-01' + (random() * 700)::integer
FROM generate_series(1, 1000);
SELECT * FROM testtablev1 LIMIT 10;
\d

# run on barman vm
barman backup pg_vssh --wait

# Time T2
CREATE TABLE testtablev2 ( name varchar, age int, joindate varchar );
INSERT INTO testtablev2 (name, age, joindate)
SELECT substr(md5(random()::text), 1, 10),
       (random() * 70 + 10)::integer,
       DATE '2018-01-01' + (random() * 700)::integer
FROM generate_series(1, 1000);
SELECT * FROM testtablev2 LIMIT 10;
\d

# Time T3
DROP TABLE testtablev1;
CREATE TABLE testtablev3 ( name varchar, age int, joindate varchar );
INSERT INTO testtablev3 (name, age, joindate)
SELECT substr(md5(random()::text), 1, 10),
       (random() * 70 + 10)::integer,
       DATE '2018-01-01' + (random() * 700)::integer
FROM generate_series(1, 1000);
SELECT * FROM testtablev3 LIMIT 10;
\d

# run on barman vm
barman backup pg_vssh --wait

# Time T4
CREATE TABLE testtable4 ( name varchar, age int, joindate varchar );
INSERT INTO testtable4 (name, age, joindate)
SELECT substr(md5(random()::text), 1, 10),
       (random() * 70 + 10)::integer,
       DATE '2018-01-01' + (random() * 700)::integer
FROM generate_series(1, 1000);
SELECT * FROM testtable4 LIMIT 10;
\d

# Time T5
DROP TABLE testtablev2;
CREATE TABLE testtable5 ( name varchar, age int, joindate varchar );
INSERT INTO testtable5 (name, age, joindate)
SELECT substr(md5(random()::text), 1, 10),
       (random() * 70 + 10)::integer,
       DATE '2018-01-01' + (random() * 700)::integer
FROM generate_series(1, 1000);
SELECT * FROM testtable5 LIMIT 10;
\d

# run on barman vm
barman backup pg_vssh --wait

```

##### Now we Have 4 Backup
```
barman list-backup pg_vssh

pg_vssh 20220420T102951 - Wed Apr 20 08:56:17 2022 - Size: 23.5 MiB - WAL Size: 0 B
pg_vssh 20220420T123816 - Wed Apr 20 08:56:02 2022 - Size: 23.5 MiB - WAL Size: 50.6 KiB 
pg_vssh 20220420T123805 - Wed Apr 20 08:55:48 2022 - Size: 23.5 MiB - WAL Size: 50.2 KiB 
pg_vssh 20220420T102904 - Wed Apr 20 08:55:41 2022 - Size: 23.5 MiB - WAL Size: 51.1 KiB 

# pg_vssh 20220420T102951 Contains v3 & v4
# pg_vssh 20220420T123816 Contains v2 & v3
# pg_vssh 20220420T123805 Contains v1 & v2
# pg_vssh 20220420T102904 Contains v1
```

##### Test Point-in-time recovery from Barman (PITR)
```
# Connect to pg_vssh and shut down the database cluster
ssh postgres@172.24.130.192 /usr/lib/postgresql/12/bin/pg_ctl \
    --pgdata=/etc/postgresql/12/main stop

# Back up the corrupt data directory, just in case something goes wrong. 
# Then delete the corrupt data.
ssh postgres@172.24.130.192 "rm -rf /var/lib/postgresql/12/remote_backup/* \
    && cp -a /var/lib/postgresql/12/main \
    /var/lib/postgresql/12/remote_backup \
    && rm -rf /var/lib/postgresql/12/main/*"

# Use Barman's recover command to connect to pg_vssh and restore the latest backup
barman recover --remote-ssh-command 'ssh postgres@172.24.130.192' \
        pg_vssh latest \
        /var/lib/postgresql/12/main

# Restart the server:
ssh postgres@172.24.130.192 "/usr/lib/postgresql/12/bin/pg_ctl \
    --pgdata=/etc/postgresql/12/main \
    -l /var/log/postgresql/postgresql-12-main.log \
    start \
    ; tail /var/log/postgresql/postgresql-12-main.log"

# Recover from Old Data
ssh postgres@172.24.130.192 "rm -rf /var/lib/postgresql/main/* \
    && cp -a /var/lib/postgresql/12/remote_backup \
    /var/lib/postgresql/main \
    && rm -rf /var/lib/postgresql/12/remote_backup/*"
```

##### Test Point-in-time recovery (PITR)
```
# run on postresql vm
# stop postresql & delete db data
\q
exit
sudo systemctl stop postgresql
rm -rf /var/lib/postgresql/12/*
ls -ltr /var/lib/postgresql/12

# run on barman vm
# start recover
#
barman list-backup pg_vssh
barman show-backup pg_vssh 20220426T104119 | grep "End time"
barman recover --remote-ssh-command "ssh postgres@172.24.130.192" \
    pg_vssh latest /var/lib/postgresql/12/main
barman recover --remote-ssh-command "ssh postgres@172.24.130.192" \
    --get-wal \
    pg_vssh latest /var/lib/postgresql/12/main
barman recover --remote-ssh-command "ssh postgres@172.24.130.192" \
    pg_vssh 20220426T104119 /var/lib/postgresql/12/main \
    --target-time='2022-04-26 10:41:25.783844+03:00'
    
# run on postresql vm
# start postresql
ls -ltr /var/lib/postgresql/12/main
sudo systemctl start postgresql
sudo su - postgres
psql
\d

# run on barman vm
barman check pg_vssh
barman list-server
tail -100 /var/log/barman/barman.log
```

##### Backup to Local/Remote Disk
```
## 
## Backup to barman vm
##
## run on barman vm
#ls -l /backup_test/backup_data
#barman recover pg_vssh 20220420T123816 /backup_test/backup_data
#ls -l /backup_test/backup_data
#rm -rf /backup_test/backup_data/*
## 
## Backup to postresql vm
##
## run on postresql vm
#\q
#ls -l /backup_test/backup_data
## run on barman vm
#barman recover --remote-ssh-command "ssh postgres@172.24.130.192" pg_vssh 20220420T123816 /backup_test/backup_data
## run on postresql vm
#ls -l /backup_test/backup_data
#rm -rf /backup_test/backup_data/*
```

##### Automatizing Backups
```
sudo su - barman
crontab -e
15 * * * * /usr/bin/barman backup pg_vssh
crontab -l
```

```
barman list-backup pg_vssh
```

# Related Commands
```
barman check pg_vssh

barman list-backup pg_vssh
barman show-backup pg_vssh latest

barman backup pg_vssh
barman backup pg_vssh --wait
barman backup --reuse=link pg_vssh

barman replication-status pg_vssh
barman show-server pg_vssh

tail -100 /var/log/barman/barman.log

barman cron

# barman receive-wal pg_vssh
barman receive-wal --drop-slot pg_vssh
barman receive-wal --create-slot pg_vssh
barman receive-wal --reset pg_vssh
barman switch-wal --force --archive pg_vssh


rm -rf /var/lib/barman/*
barman receive-wal --stop pg_vssh
```

# Errors
```
# https://github.com/EnterpriseDB/barman/issues/281
# https://github.com/EnterpriseDB/barman/issues/282
# https://dba.stackexchange.com/questions/193364/error-receiving-wal-files-with-barman
# replication slot: FAILED (slot 'barman' not initialised: is 'receive-wal' running?)
# receive-wal running: FAILED (See the Barman log file for more details)
# barman@bm-awal:~$ barman receive-wal pg_vssh
# Starting receive-wal for server pg_vssh
# pg_vssh: pg_receivewal: starting log streaming at 0/C000000 (timeline 1)
# pg_vssh: pg_receivewal: error: unexpected termination of replication stream: ERROR:  requested starting point 0/C000000 is ahead of the WAL flush position of this server 0/8007FF8
# pg_vssh: pg_receivewal: error: disconnected
# ERROR: ArchiverFailure:pg_receivexlog terminated with error code: 1

barman receive-wal --drop-slot pg_vssh
barman receive-wal --create-slot pg_vssh

```

# psql db
```
vi /etc/postgresql/12/main/pg_hba.conf
```
# barman
```
vi /etc/barman.conf
vi /etc/barman.d/pg_vssh.conf
```

##### Links & Courses
```
# Binary Replication Tools Comparison
https://wiki.postgresql.org/wiki/Binary_Replication_Tools

# BARMAN | PostgreSQL Backup & Restore | Streaming Backup | Part -1
https://www.youtube.com/watch?v=hburMB0Op3A&t=52s

# PostgreSQL Backup & Restore Using Barman | vssh/SSH Backup | Part -2
https://www.youtube.com/watch?v=OOZOzRY9bxk

# Single Server Streaming Example
https://www.enterprisedb.com/docs/supported-open-source/barman/single-server-streaming/step01-db-setup/
```
