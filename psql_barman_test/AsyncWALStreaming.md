# MULTIPASS VM

##### VM: PSQL DB Server for Async WAL Streaming
```
multipass launch --name db-awal 20.04 -c 1 -m 1G -d 10G
multipass shell db-awal
```

##### VM: BARMAN Server
```
multipass launch --name bm-awal 20.04 -c 1 -m 1G -d 10G
multipass shell bm-awal
```

##### IP Adresses:
```
ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1  -d'/'
```

##### Replace All IP Addresses in Document
- db-awal: 172.29.120.80
- bm-awal: 172.29.126.157

##### MULTIPASS VM Deletion
```
multipass delete db-awal
multipass delete bm-awal
multipass purge
```

# POSTGRESQL DB SERVER INITIALIZATION (IP: 172.29.120.80)

##### Installation Requirements
```
sudo apt-get -y update
sudo apt-get -y upgrade
sudo apt-get -y install postgresql postgresql-contrib policycoreutils
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

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDNCTmaCHFntH/RgD7KYXmR3YGJQWQ37VUXJIMz6Q9vuDm93RudPXiTYhY1IllFvH5d0K8YuAXRCHjwBfPUIOTHdnhmwK3bH7j2MTbnTvwa0iNT2SgOH5EvtaJE8e30OLhJrUw+TeA6+P6iJe03iYxEojq2IR/vVta58zuywQi1eccIxhFJ/R8TwQLhN8BqPMClOOjBDBXePtygU6axiVSapBT+LVRWfJHXKU8M62T6nXVqaTc5wHxOT0Z5RPp5TpjQY/6XepeDeRGV2QstF4zdaEmIZE8nTX9J5RHlTt/gKmS24j5ZVFSoA2EGstVh+X+ZoZrmMYLzzqBuQyq5RbvblzEHue0yI6g+fPXoT+kSIpYm9hRoFprye2mb2wUlKkjM4egDJ7d3aK/7J1jIgDcCkQpLrp+ppcrXQBIoq0FWnnAQfcGFHGly+EICFk0L0hZI2mM8ly301bFkwsoTp6RDpWt/9hnAtcZNyasGeONJ233m6U9VBrfLlEf3aXK1Ac8= postgres@db-awal
```

##### Get/Save SSH-RSA Public Key From BARMAN SERVER VM
```
# rm -rf /var/lib/postgresql/.ssh/authorized_keys
# touch /var/lib/postgresql/.ssh/authorized_keys
# chmod 0600 /var/lib/postgresql/.ssh/authorized_keys
vi /var/lib/postgresql/.ssh/authorized_keys

#
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDAxFSNMNow6Y0kGSf9L5DPRGiBlBjsDS8XyQPHwlHSSxjwsFn/uXEwoJ+/h8AReIqLT6Ej7O9Rfo2SRkWrSzQAXpEocN/kfDAcLMXtg7olFTh1AjoA5b3187xZiJg45PGOC/TD1k27cUShr8hrIN3X/YQJ0YCJVPvBYdx2JSUYSoodcYRvXRHfv+noW7guMY98cZYEUxw1DtKSpmEHiJukWmdUf2+XRbUFuxY4vPknwTWNgGp+AYmg/bsHNsls/st0O9/6Ar2vfCRBuiPeejl9uAMhgtkrqoxwWBrZ7bj1Ga76Puq62qTTp+Ggiotwo3X0ur+u8gDEzvriO52dWCUdOMnNQtzCF3IHZA2GadB/pKEOWpoMTVIMe1fbWXlqsNMSLHFYqLee1gBjcHZOAMno7gw/IloTNiSyb4UUYJP7fHfBourZ8QNqNLTZ3yFOwyMWjr7UaeaduJR2M8SX5p9phc/oVfonU7NOus1/pCCvRAPSIsHjm4p+aLJSlbzCTk0= barman@bm-awal
#

chmod 700 /var/lib/postgresql/.ssh
chmod 600 /var/lib/postgresql/.ssh/authorized_keys
restorecon -r -vv /var/lib/postgresql/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
ssh barman@172.29.126.157
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
host    replication     all             *172.29.126.157/32       trust*
*host    replication     streaming_barman 172.29.126.157/32       trust*
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

\q
exit

sudo systemctl restart postgresql

sudo su - postgres
psql

select name,setting from pg_settings where name like 'listen_addresses%';

\q
exit
```

##### Check WAL Directory & MakeDir Test & Test Barman-Wal-Archive
```
ls -ltr /var/lib/postgresql/12/main/pg_wal/

mkdir -p /backup_test/backup_data
chown -R postgres:postgres /backup_test/backup_data

sudo su - postgres

psql
```

# BARMAN SERVER INITIALIZATION (IP: 172.29.126.157)
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

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDAxFSNMNow6Y0kGSf9L5DPRGiBlBjsDS8XyQPHwlHSSxjwsFn/uXEwoJ+/h8AReIqLT6Ej7O9Rfo2SRkWrSzQAXpEocN/kfDAcLMXtg7olFTh1AjoA5b3187xZiJg45PGOC/TD1k27cUShr8hrIN3X/YQJ0YCJVPvBYdx2JSUYSoodcYRvXRHfv+noW7guMY98cZYEUxw1DtKSpmEHiJukWmdUf2+XRbUFuxY4vPknwTWNgGp+AYmg/bsHNsls/st0O9/6Ar2vfCRBuiPeejl9uAMhgtkrqoxwWBrZ7bj1Ga76Puq62qTTp+Ggiotwo3X0ur+u8gDEzvriO52dWCUdOMnNQtzCF3IHZA2GadB/pKEOWpoMTVIMe1fbWXlqsNMSLHFYqLee1gBjcHZOAMno7gw/IloTNiSyb4UUYJP7fHfBourZ8QNqNLTZ3yFOwyMWjr7UaeaduJR2M8SX5p9phc/oVfonU7NOus1/pCCvRAPSIsHjm4p+aLJSlbzCTk0= barman@bm-awal
```

##### Get/Save SSH-RSA Public Key From BARMAN SERVER VM
```
# rm -rf /var/lib/barman/.ssh/authorized_keys
# touch /var/lib/barman/.ssh/authorized_keys
# chmod 0600 /var/lib/barman/.ssh/authorized_keys
vi /var/lib/barman/.ssh/authorized_keys

#
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDNCTmaCHFntH/RgD7KYXmR3YGJQWQ37VUXJIMz6Q9vuDm93RudPXiTYhY1IllFvH5d0K8YuAXRCHjwBfPUIOTHdnhmwK3bH7j2MTbnTvwa0iNT2SgOH5EvtaJE8e30OLhJrUw+TeA6+P6iJe03iYxEojq2IR/vVta58zuywQi1eccIxhFJ/R8TwQLhN8BqPMClOOjBDBXePtygU6axiVSapBT+LVRWfJHXKU8M62T6nXVqaTc5wHxOT0Z5RPp5TpjQY/6XepeDeRGV2QstF4zdaEmIZE8nTX9J5RHlTt/gKmS24j5ZVFSoA2EGstVh+X+ZoZrmMYLzzqBuQyq5RbvblzEHue0yI6g+fPXoT+kSIpYm9hRoFprye2mb2wUlKkjM4egDJ7d3aK/7J1jIgDcCkQpLrp+ppcrXQBIoq0FWnnAQfcGFHGly+EICFk0L0hZI2mM8ly301bFkwsoTp6RDpWt/9hnAtcZNyasGeONJ233m6U9VBrfLlEf3aXK1Ac8= postgres@db-awal
#

chmod 700 /var/lib/barman/.ssh
chmod 600 /var/lib/barman/.ssh/authorized_keys
restorecon -r -vv /var/lib/barman/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
ssh postgres@172.29.120.80
exit
```

##### Set Postgress User's Password
```
cat <<EOF | tee /var/lib/barman/.pgpass
172.29.120.80:5432:postgres:barman:welcome
172.29.120.80:5432:postgres:streaming_barman:welcome
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
```

##### Configure Barman (Using streaming-server.conf-template)
```
# sudo rm -rf /etc/barman.d/pg.conf
sudo cp /etc/barman.d/streaming-server.conf-template /etc/barman.d/pg.conf
sudo vi /etc/barman.d/pg.conf

# change '[streaming]' to 'pg'
# change 'conninfo = host=pg' to 'conninfo = host=172.29.120.80'
# change 'streaming_conninfo = host=pg' to 'conninfo = host=172.29.120.80'
# uncomment 'streaming_backup_name'
# uncomment 'create_slot'
# uncomment 'streaming_archiver_name'
# uncomment 'streaming_archiver_batch_size'
# uncomment 'path_prefix'
```

##### Test Connection to Psql
```
sudo su - barman
psql -c 'SELECT version()' -U barman -d postgres -h 172.29.120.80 -p 5432
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

pg 20220420T085614 - Wed Apr 20 08:56:17 2022 - Size: 23.5 MiB - WAL Size: 0 B
pg 20220420T085600 - Wed Apr 20 08:56:02 2022 - Size: 23.5 MiB - WAL Size: 50.6 KiB 
pg 20220420T085546 - Wed Apr 20 08:55:48 2022 - Size: 23.5 MiB - WAL Size: 50.2 KiB 
pg 20220420T085538 - Wed Apr 20 08:55:41 2022 - Size: 23.5 MiB - WAL Size: 51.1 KiB 

# pg 20220420T085614 Contains T3 & T4
# pg 20220420T085600 Contains T2 & T3
# pg 20220420T085546 Contains T1 & T2
# pg 20220420T085538 Contains T1
```

##### Backup to Local/Remote Disk
```
# 
# Backup to barman vm
#
# run on barman vm
ls -l /backup_test/backup_data
barman recover pg 20220420T085600 /backup_test/backup_data
ls -l /backup_test/backup_data
rm -rf /backup_test/backup_data/*

# 
# Backup to postresql vm
#
# run on postresql vm
\q
ls -l /backup_test/backup_data
# run on barman vm
barman recover --remote-ssh-command "ssh postgres@172.29.120.80" pg 20220420T085600 /backup_test/backup_data
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
barman show-backup pg 20220421T112227 | grep "End time"
# barman recover --remote-ssh-command "ssh postgres@172.29.120.80" \
#    pg latest /var/lib/postgresql/12/main
# barman recover --remote-ssh-command "ssh postgres@172.29.120.80" \
#    pg 20220420T163006 /var/lib/postgresql/12/main \
#    --target-time='2022-04-20 16:30:08.998470+03:00'
barman recover --remote-ssh-command "ssh postgres@172.29.120.80" \
    pg 20220421T112227 /var/lib/postgresql/12/main \
    --target-time='2022-04-21 11:22:29.924822+03:00'

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
