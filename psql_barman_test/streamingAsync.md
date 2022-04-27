# MULTIPASS VM

##### VM: PSQL DB Server for Streaming Async
```
multipass launch --name db-async 20.04 -c 1 -m 1G -d 10G
multipass shell db-async
```

##### IP Adresses:
```
ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1  -d'/'
```

##### Replace All IP Addresses in Document
- db-async: 172.24.138.9
- bms: 172.24.139.61

##### MULTIPASS VM Deletion
```
multipass delete db-async
multipass purge
```

# POSTGRESQL DB SERVER INITIALIZATION (IP: 172.24.138.9)

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

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDgQbU0psPS9FUMLa/3X0LBRJPzJlembx+AvZf+BzKRyKJ+lMp6lo/pM1IwsKt9gL8yo73rwl6SCqYh2B+eMd8q+CrIP06TWLpVO6UkbQHTLLEk2725ioxhn5i4f1yjL7aWC4H+Mn7cdKgiaL4v0El/2rniZrQivpXSWfqAjfxr7vAJoVvpeXHUvImWxMehlldw1eraG6eKQBYi5QJeivVxHPZePhvjZFm3zJ5zuBaDf5Nkq+jSo6dpnh+ijNFiGDwPqClz/iujTP5/uieoCbd7mbj6/H9ItlSu1YS9u176vUlMkKqsDH8Zes9ToZJeo38QKM7JrSf3InCGyOvZO4uQkNLQ6Hwnmzv+1HnSWea7SCOggSTPi5hCleq0FmisffwzNpun7qzioKMBpIYa0+QthMF1Bcsc7U64IyjqYbenSIsuC3wp/m02uNWWDs+FAnVw/BjFpZ6MLEKpr3TZchLHXPjgYYLhGFeUN31iOC+IqaEENStQomV5DJEDoHjPKZE= postgres@db-async
```

##### Get/Save SSH-RSA Public Key From BARMAN SERVER VM
```
# rm -rf /var/lib/postgresql/.ssh/authorized_keys
# touch /var/lib/postgresql/.ssh/authorized_keys
# chmod 0600 /var/lib/postgresql/.ssh/authorized_keys
mkdir /var/lib/postgresql/.ssh
vi /var/lib/postgresql/.ssh/authorized_keys

#
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC58O1t58kNll+AzzC64zyFjBHm7cLsGL0uax6ZdqWwFrgyjIFkAYiKCzFLP+KD1BH+G2xxJCm5Yj7HP/2lRRqNlrl4ZuzppSNn0/WDXz+ozxd5NBHwtDc0tetzkbHWTX0Ux8asyTPhDotlOzxYG+YSeLvWvK5K9zuU6DjRvK4nGmaGEc6nSRKe6Pz03U8b3IATMcArJqJmIT70D9w/vAUmVzX9VRH5qNnBcIOCFfSpqGJanl2902w8e4z+cyQdClbPb96KCO3AwxgqXp+Qma8MCjthJzwpimRLYKqGnl6ZHIXNn+wOP3nwK6G71HZaEZ3gp1SqyNz1142015LPiDeT4VOfc2kcmWq307kXnqJfsdNuujK72lWPThOR5q5LTizN3cl7ofTzctjB0M6xK8etjYWIaQzvbXdinFiVjCXlkUtJsGnMsUX/f+vqPnNIZhI9PFEQIDtYEf10N+6Z98P5VehezfXRo6VsCRkJZUvJPQTKCOmzD7ZVQ638NOI7LmU= barman@bms
#

chmod 700 /var/lib/postgresql/.ssh
chmod 600 /var/lib/postgresql/.ssh/authorized_keys
restorecon -r -vv /var/lib/postgresql/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
ssh barman@172.24.139.61
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
SHOW data_directory;
# /var/lib/postgresql/12/main

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
host    replication     all             *172.24.139.61/32       trust*
*host    replication     streaming_barman 172.24.139.61/32       trust*
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

# POSTGRESQL DB DURDUR/BASLAT
```
\q
exit
sudo systemctl restart postgresql
sudo su - postgres
psql
\d
```

# SAVE BASE DATA AS MAIN PSQL RECOVER
```
\q
exit
sudo systemctl stop postgresql
rm -rf /backup_test/backup_data/*
cp -a /var/lib/postgresql/12/main /backup_test/backup_data
ls -l /backup_test/backup_data/main
sudo systemctl start postgresql
sudo su - postgres
psql
\d
```

# RESTORE BASE DATA AS MAIN PSQL RECOVER
```
\q
exit
sudo systemctl stop postgresql
rm -rf /var/lib/postgresql/12/*
cp -a /backup_test/backup_data/* /var/lib/postgresql/12
ls -ltr /var/lib/postgresql/12
sudo systemctl start postgresql
sudo su - postgres
psql
\d
```

# Insert Random Data
```
INSERT INTO milliontable (name, age, joindate)
SELECT substr(md5(random()::text), 1, 10),
       (random() * 70 + 10)::integer,
       DATE '2018-01-01' + (random() * 700)::integer
FROM generate_series(1, 1000000);
```
