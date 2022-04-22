# Test Connection to Psql

##### Replace All IP Addresses in Document
Current Test:
- db_name: pg_vssh
- db_ip: 172.29.112.36
- bms: 172.29.123.90

```
sudo su - barman
psql -c 'SELECT version()' -U barman -d postgres -h 172.29.112.36 -p 5432
```

##### Test Barman
```
barman check pg_vssh
barman switch-wal --force --archive pg_vssh
barman check pg_vssh
barman replication-status pg_vssh
barman list-server

# barman backup pg_vssh --wait

tail -100 /var/log/barman/barman.log
```


# Backup & Recovery Demo
##### Add Table T1 to Psql & Run Backup Barman
```
# run on postresql vm
create table T1 ( id1 int , id2 int );
insert into  T1 values (3,2);
\d

# run on barman vm
barman backup pg_vssh --wait
```

##### Add Table T2 to Psql & Run Backup Barman
```
# run on postresql vm
create table T2 ( id1 int , id2 int );
insert into  T2 values (2,2);
\d

# run on barman vm
barman backup pg_vssh --wait
```

##### Add Table T3 to Psql & Drop T1 from Psql & Run Backup Barman
```
# run on postresql vm
drop table T1;
create table T3 ( id1 int , id2 int );
insert into  T3 values (3,2);
\d

# run on barman vm
barman backup pg_vssh --wait
```

##### Add Table T4 to Psql & Drop T2 from Psql & Run Backup Barman
```
# run on postresql vm
drop table T2;
create table T4 ( id1 int , id2 int );
insert into  T4 values (4,2);
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

# pg_vssh 20220420T102951 Contains T3 & T4
# pg_vssh 20220420T123816 Contains T2 & T3
# pg_vssh 20220420T123805 Contains T1 & T2
# pg_vssh 20220420T102904 Contains T1
```

##### Backup to Local/Remote Disk
```
# 
# Backup to barman vm
#
# run on barman vm
ls -l /backup_test/backup_data
barman recover pg_vssh 20220420T123816 /backup_test/backup_data
ls -l /backup_test/backup_data
rm -rf /backup_test/backup_data/*

# 
# Backup to postresql vm
#
# run on postresql vm
\q
ls -l /backup_test/backup_data
# run on barman vm
barman recover --remote-ssh-command "ssh postgres@172.29.112.36" pg_vssh 20220420T123816 /backup_test/backup_data
# run on postresql vm
ls -l /backup_test/backup_data
rm -rf /backup_test/backup_data/*
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
barman show-backup pg_vssh 20220422T130738 | grep "End time"
# barman recover --remote-ssh-command "ssh postgres@172.29.112.36" \
#    pg_vssh 20220420T163006 /var/lib/postgresql/12/main
# barman recover --remote-ssh-command "ssh postgres@172.29.112.36" \
#    pg_vssh 20220420T163006 /var/lib/postgresql/12/main \
#    --target-time='2022-04-20 12:38:10.067043+03:00'
barman recover --remote-ssh-command "ssh postgres@172.29.112.36" \
    pg_vssh 20220422T130738 /var/lib/postgresql/12/main \
    --target-time='2022-04-22 13:07:43.677020+03:00'

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

##### Automatizing Backups
```
sudo su - barman
crontab -e
15 * * * * /usr/bin/barman backup pg_vssh
crontab -l
```
barman list-backup pg_vssh

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
barman switch-wal --force --archive pg_vssh

barman receive-wal --drop-slot pg_vssh
barman receive-wal --create-slot pg_vssh
barman receive-wal --reset pg_vssh
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

# sudo vi /etc/barman.d/pg_vssh.conf İÇERİSİNDE
# path_prefix = "/usr/pgsql-12/bin"
sudo su - barman
export PATH=$PATH:/usr/pgsql-12/bin; barman cron >/dev/null 2>&1

# POSTGRESQL DB DURDUR/BASLAT
\q
exit
# sudo systemctl stop postgresql
# sudo systemctl start postgresql
sudo systemctl restart postgresql
sudo su - postgres
psql
\d

# wal_level=replica
# archive_command=(disabled)(?)
# archive_mode=off(?)
# max_wal_sender=2(?)
# max_replication_slots=2(?)
select name,setting from pg_settings where name like 'wal_level%';
select name,setting from pg_settings where name like 'archive%';
select name,setting from pg_settings where name like 'max_wal_sender%';
select name,setting from pg_settings where name like 'max_replication_slots%';
```

# psql db
vi /etc/postgresql/12/main/pg_hba.conf
# barman
vi /etc/barman.conf
vi /etc/barman.d/pg_vssh.conf

##### Links & Courses
```
# BARMAN | PostgreSQL Backup & Restore | Streaming Backup | Part -1
https://www.youtube.com/watch?v=hburMB0Op3A&t=52s

# PostgreSQL Backup & Restore Using Barman | vssh/SSH Backup | Part -2
https://www.youtube.com/watch?v=OOZOzRY9bxk
```
