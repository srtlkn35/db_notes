# MULTIPASS VM

##### VM: PSQL DB Server for Async WAL Streaming
```
multipass launch --name db-rsync 20.04 -c 1 -m 1G -d 10G
multipass shell db-rsync
```

##### VM: BARMAN Server
```
multipass launch --name bm-rsync 20.04 -c 1 -m 1G -d 10G
multipass shell bm-rsync
```

##### IP Adresses:
```
ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1  -d'/'
```

##### Replace All IP Addresses in Document
- db-rsync: 172.29.114.90
- bm-rsync: 172.29.118.220

##### MULTIPASS VM Deletion
```
multipass delete db-rsync
multipass delete bm-rsync
multipass purge
```

# POSTGRESQL DB SERVER INITIALIZATION (IP: 172.29.114.90)

##### Installation Requirements
```
sudo apt-get -y update
sudo apt-get -y upgrade
sudo apt-get -y install postgresql postgresql-contrib barman-cli policycoreutils
```

##### Create/Set User Passwords (Passwords: 'welcome')
```
sudo passwd postgres

sudo su - postgres

createuser -s -P barman
createuser -s -P streaming_barman
```

##### Generate/Save SSH-RSA Public Key
```
ssh-keygen -t rsa
cat /var/lib/postgresql/.ssh/id_rsa.pub

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDJdqN6rT8qYijj81kfVqONHGP5OmaMMevsf4+uGZkqL5h7uI1lncL8zvgTwhH9zyHjf2vsTIX6H0Y5+XwZ3AoakaIiXIobe5iJL46qmqM6VkEz42BA6nDX8AIRjWjhIQoJg8ni27WTe53SNhu64zG3NlZW2OQ4iKAQYfgZp/SBz6qtfrDyQ0eptvmjXjY39i/tkdzKD89IC22LRju1fFRHE9XyE3IL32f4f5HwltvM/m3gszxI3efPJGwcuHLd+G9eCHZiAkHIlsIlsI6a4t/ZDh+RbNJlfH4TdCVgfN2HZ7a9GvrGNLQF/QzkSsxJ2XVWKHu4Krjv5ojirXkk59fLNa0vcFc/3AruxQQmYLTsaNTb6y8hvRWRGJ/YIFLlNK+PgfiXT3ajcimBgnn+3K68wOauIahPlBrT/cYvAVFpe+UrHbyfomde85Vo2gDjz9O2kg4y48kSLWKCbza7BR9YoWgPwkcK4LVCeFwMdjvwcnn63b9WM0wQbH0yq69vJbE= postgres@db-rsync
```

##### Get/Save SSH-RSA Public Key From BARMAN SERVER VM
```
# rm -rf /var/lib/postgresql/.ssh/authorized_keys
# touch /var/lib/postgresql/.ssh/authorized_keys
# chmod 0600 /var/lib/postgresql/.ssh/authorized_keys
vi /var/lib/postgresql/.ssh/authorized_keys

#
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCnqilkOFS2uxOH2VeWyBWZe24t1d6L/oVg1G3RuhvUdxrGVnO1cFc0YpCIhRxzCzCoEmTEM3acA2+xuExOi9hJO5m/c0/7AVCo3UThuAVYKgiZEmMubx+u7T7l1w0sI+j5RwnY3yhOAuTLALIedyeScuZDn6zLI1R068SnvHHnQH3VHH8kzP7713sG1NpdNMbety2ovQivVLLTihquNrducrVuRu8Hg/dHANeBKmWV+v7KYMYsyBGbFZde4tVqqVRo+DuAizylzSR0WHIjj8c04T93jKC2REWdpCFZ5VoTs0miZeHKxk19ueEfKeLYp9inQQ3maTZdxE/McM3maAMygFfbsYKKuMIjyll/4MUhs24DvdxO0x530BXepIfjBUyLqBb5eVb0O4bu0uk+G/SqDEYSK+QUbJXjxU3b4GwXPoWTreoRIosn1LhB3VY5ZouwVH0fMFmybjI/FErL7JHa7duKK8wjro7ioD2YZPG6JVynZLStrxoYGXFyWDXFnUU= barman@bm-rsync
#

chmod 700 /var/lib/postgresql/.ssh
chmod 600 /var/lib/postgresql/.ssh/authorized_keys
restorecon -r -vv /var/lib/postgresql/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
ssh barman@172.29.118.220
exit
```

##### Test PSQL (psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1)))
```
psql
\conninfo
SELECT version();
```

##### Priviliges to User 'barman'
```
psql
GRANT EXECUTE ON FUNCTION pg_start_backup(text, boolean, boolean) to barman;
GRANT EXECUTE ON FUNCTION pg_stop_backup() to barman;
GRANT EXECUTE ON FUNCTION pg_stop_backup(boolean, boolean) to barman;
GRANT EXECUTE ON FUNCTION pg_switch_wal() to barman;
GRANT EXECUTE ON FUNCTION pg_create_restore_point(text) to barman;
GRANT pg_read_all_settings TO barman;
GRANT pg_read_all_stats TO barman;
```

##### Configure PSQL
```
SHOW hba_file;
# /etc/postgresql/12/main/pg_hba.conf

\q
exit
sudo su
vi /etc/postgresql/12/main/pg_hba.conf
```

##### Changes are Marked with '*'
```
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             *0.0.0.0/0*             md5
# IPv6 local connections:
host    all             all             ::1/128                 *ident*
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             *172.29.118.220/32       trust*
*host    replication     streaming_barman 172.29.118.220/32       trust*
host    replication     all             ::1/128                 *ident*
```

##### Restart PSQL
```
sudo systemctl restart postgresql
```

##### Configure PSQL
```
sudo su - postgres
psql

select name,setting from pg_settings where name like 'listen_addresses%';
alter system set listen_addresses='*';

select name,setting from pg_settings where name like 'archive%';
alter system set archive_mode=on;

# WAL ARCHIVE OR RSYNC OPTION
# WAL ARCHIVE
alter system set archive_command='barman-wal-archive 172.29.118.220 pg %p';
# RSYNC (PATH FOUND FROM CMD OUTPUT: barman show-server pg, incoming_wals_directory)
# alter system set archive_command='rsync -a %p barman@172.29.118.220:/var/lib/barman/pg/incoming/%f';

\q
exit

sudo systemctl restart postgresql

sudo su - postgres
psql

select name,setting from pg_settings where name like 'listen_addresses%';
select name,setting from pg_settings where name like 'archive%';

\q
exit
```

##### Check WAL Directory & MakeDir Test & Test Barman-Wal-Archive
```
ls -ltr /var/lib/postgresql/12/main/pg_wal/

mkdir -p /backup_test/backup_data
chown -R postgres:postgres /backup_test/backup_data

sudo su - postgres
barman-wal-archive --test 172.29.118.220 pg DUMMY

psql
```

# BARMAN SERVER INITIALIZATION (IP: 172.29.118.220)
```
sudo apt-get -y update
sudo apt-get -y upgrade
sudo apt-get -y install barman barman-cli policycoreutils
```

##### Create/Set User Passwords (Passwords: 'welcome')
```
sudo passwd barman

sudo su - barman
```

##### Generate/Save SSH-RSA Public Key
```
ssh-keygen -t rsa
cat /var/lib/barman/.ssh/id_rsa.pub

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCnqilkOFS2uxOH2VeWyBWZe24t1d6L/oVg1G3RuhvUdxrGVnO1cFc0YpCIhRxzCzCoEmTEM3acA2+xuExOi9hJO5m/c0/7AVCo3UThuAVYKgiZEmMubx+u7T7l1w0sI+j5RwnY3yhOAuTLALIedyeScuZDn6zLI1R068SnvHHnQH3VHH8kzP7713sG1NpdNMbety2ovQivVLLTihquNrducrVuRu8Hg/dHANeBKmWV+v7KYMYsyBGbFZde4tVqqVRo+DuAizylzSR0WHIjj8c04T93jKC2REWdpCFZ5VoTs0miZeHKxk19ueEfKeLYp9inQQ3maTZdxE/McM3maAMygFfbsYKKuMIjyll/4MUhs24DvdxO0x530BXepIfjBUyLqBb5eVb0O4bu0uk+G/SqDEYSK+QUbJXjxU3b4GwXPoWTreoRIosn1LhB3VY5ZouwVH0fMFmybjI/FErL7JHa7duKK8wjro7ioD2YZPG6JVynZLStrxoYGXFyWDXFnUU= barman@bm-rsync
```

##### Get/Save SSH-RSA Public Key From BARMAN SERVER VM
```
# rm -rf /var/lib/barman/.ssh/authorized_keys
# touch /var/lib/barman/.ssh/authorized_keys
# chmod 0600 /var/lib/barman/.ssh/authorized_keys
vi /var/lib/barman/.ssh/authorized_keys

#
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDJdqN6rT8qYijj81kfVqONHGP5OmaMMevsf4+uGZkqL5h7uI1lncL8zvgTwhH9zyHjf2vsTIX6H0Y5+XwZ3AoakaIiXIobe5iJL46qmqM6VkEz42BA6nDX8AIRjWjhIQoJg8ni27WTe53SNhu64zG3NlZW2OQ4iKAQYfgZp/SBz6qtfrDyQ0eptvmjXjY39i/tkdzKD89IC22LRju1fFRHE9XyE3IL32f4f5HwltvM/m3gszxI3efPJGwcuHLd+G9eCHZiAkHIlsIlsI6a4t/ZDh+RbNJlfH4TdCVgfN2HZ7a9GvrGNLQF/QzkSsxJ2XVWKHu4Krjv5ojirXkk59fLNa0vcFc/3AruxQQmYLTsaNTb6y8hvRWRGJ/YIFLlNK+PgfiXT3ajcimBgnn+3K68wOauIahPlBrT/cYvAVFpe+UrHbyfomde85Vo2gDjz9O2kg4y48kSLWKCbza7BR9YoWgPwkcK4LVCeFwMdjvwcnn63b9WM0wQbH0yq69vJbE= postgres@db-rsync
#

chmod 700 /var/lib/barman/.ssh
chmod 600 /var/lib/barman/.ssh/authorized_keys
restorecon -r -vv /var/lib/barman/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
ssh postgres@172.29.114.90
exit
```

##### Set Postgress User's Password
```
cat <<EOF | tee /var/lib/barman/.pgpass
172.29.114.90:5432:postgres:barman:welcome
172.29.114.90:5432:postgres:streaming_barman:welcome
EOF

cat .pgpass
chmod 0600 /var/lib/barman/.pgpass
```

##### MakeDir Test
```
exit
sudo su
mkdir -p /backup_test/backup_data
chown -R barman:barman /backup_test/backup_data
```

##### Configure Barman
```
sudo vi /etc/barman.conf

# uncomment 'compression = gzip'
# change 'post_backup_script=barman generate-manifest ${BARMAN_SERVER} ${BARMAN_BACKUP_ID}'
```

##### Configure Barman (Using streaming-server.conf-template)
```
# sudo rm -rf /etc/barman.d/pg.conf
sudo cp /etc/barman.d/ssh-server.conf-template /etc/barman.d/pg.conf
sudo vi /etc/barman.d/pg.conf

# change '[ssh]' to 'pg'
# change 'ssh postgres@pg' to 'ssh postgres@172.29.114.90'
# change 'conninfo = host=pg' to 'conninfo = host=172.29.114.90'
# uncomment 'reuse_backup'
# uncomment 'path_prefix'

# WAL ARCHIVE OR RSYNC OPTION
# WAL ARCHIVE
# add streaming_conninfo below conninfo
streaming_conninfo  =host=172.29.114.90 user=streaming_barman
# RSYNC
#DO NOTHING
```

##### Test Connection to Psql
```
sudo su - barman
psql -c 'SELECT version()' -U barman -d postgres -h 172.29.114.90 -p 5432
```

##### Test Barman
```
barman check pg
barman switch-wal --force --archive pg
barman check pg
barman replication-status pg
barman list-server

# barman backup pg --wait

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
barman backup pg --wait
```

##### Add Table T2 to Psql & Run Backup Barman
```
# run on postresql vm
create table T2 ( id1 int , id2 int );
insert into  T2 values (2,2);
\d

# run on barman vm
barman backup pg --wait
```

##### Add Table T3 to Psql & Drop T1 from Psql & Run Backup Barman
```
# run on postresql vm
drop table T1;
create table T3 ( id1 int , id2 int );
insert into  T3 values (3,2);
\d

# run on barman vm
barman backup pg --wait
```

##### Add Table T4 to Psql & Drop T2 from Psql & Run Backup Barman
```
# run on postresql vm
drop table T2;
create table T4 ( id1 int , id2 int );
insert into  T4 values (4,2);
\d

# run on barman vm
barman backup pg --wait
```

##### Now we Have 4 Backup
```
barman list-backup pg

pg 20220420T102951 - Wed Apr 20 08:56:17 2022 - Size: 23.5 MiB - WAL Size: 0 B
pg 20220420T123816 - Wed Apr 20 08:56:02 2022 - Size: 23.5 MiB - WAL Size: 50.6 KiB 
pg 20220420T123805 - Wed Apr 20 08:55:48 2022 - Size: 23.5 MiB - WAL Size: 50.2 KiB 
pg 20220420T102904 - Wed Apr 20 08:55:41 2022 - Size: 23.5 MiB - WAL Size: 51.1 KiB 

# pg 20220420T102951 Contains T3 & T4
# pg 20220420T123816 Contains T2 & T3
# pg 20220420T123805 Contains T1 & T2
# pg 20220420T102904 Contains T1
```

##### Backup to Local/Remote Disk
```
# 
# Backup to barman vm
#
# run on barman vm
ls -l /backup_test/backup_data
barman recover pg 20220420T123816 /backup_test/backup_data
ls -l /backup_test/backup_data
rm -rf /backup_test/backup_data/*

# 
# Backup to postresql vm
#
# run on postresql vm
\q
ls -l /backup_test/backup_data
# run on barman vm
barman recover --remote-ssh-command "ssh postgres@172.29.114.90" pg 20220420T123816 /backup_test/backup_data
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
barman list-backup pg
barman show-backup pg 20220422T130738 | grep "End time"
# barman recover --remote-ssh-command "ssh postgres@172.29.114.90" \
#    pg 20220420T163006 /var/lib/postgresql/12/main
# barman recover --remote-ssh-command "ssh postgres@172.29.114.90" \
#    pg 20220420T163006 /var/lib/postgresql/12/main \
#    --target-time='2022-04-20 12:38:10.067043+03:00'
barman recover --remote-ssh-command "ssh postgres@172.29.114.90" \
    pg 20220422T130738 /var/lib/postgresql/12/main \
    --target-time='2022-04-22 13:07:43.677020+03:00'

# run on postresql vm
# start postresql
ls -ltr /var/lib/postgresql/12/main
sudo systemctl start postgresql
sudo su - postgres
psql
\d

# run on barman vm
barman check pg
barman list-server
tail -100 /var/log/barman/barman.log
```
##### Related Commands
```
barman check pg

barman list-backup pg
barman show-backup pg latest

barman backup pg
barman backup pg --wait
barman backup --reuse=link pg

barman replication-status pg
barman show-server pg

tail -100 /var/log/barman/barman.log

barman cron
# barman receive-wal pg
barman switch-wal --force --archive pg

barman receive-wal --drop-slot pg
barman receive-wal --create-slot pg
barman receive-wal --reset pg
rm -rf /var/lib/barman/*

barman receive-wal --stop pg
```

##### Errors
```
# https://github.com/EnterpriseDB/barman/issues/281
# https://github.com/EnterpriseDB/barman/issues/282
# https://dba.stackexchange.com/questions/193364/error-receiving-wal-files-with-barman
# replication slot: FAILED (slot 'barman' not initialised: is 'receive-wal' running?)
# receive-wal running: FAILED (See the Barman log file for more details)
# barman@bm-awal:~$ barman receive-wal pg
# Starting receive-wal for server pg
# pg: pg_receivewal: starting log streaming at 0/C000000 (timeline 1)
# pg: pg_receivewal: error: unexpected termination of replication stream: ERROR:  requested starting point 0/C000000 is ahead of the WAL flush position of this server 0/8007FF8
# pg: pg_receivewal: error: disconnected
# ERROR: ArchiverFailure:pg_receivexlog terminated with error code: 1

# sudo vi /etc/barman.d/pg.conf İÇERİSİNDE
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

##### Links & Courses
```
# BARMAN | PostgreSQL Backup & Restore | Streaming Backup | Part -1
https://www.youtube.com/watch?v=hburMB0Op3A&t=52s

# PostgreSQL Backup & Restore Using Barman | rsync/SSH Backup | Part -2
https://www.youtube.com/watch?v=OOZOzRY9bxk
```
