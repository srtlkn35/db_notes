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
- db-async: 172.29.119.179
- bms: 172.29.123.90

##### MULTIPASS VM Deletion
```
multipass delete db-async
multipass purge
```

# POSTGRESQL DB SERVER INITIALIZATION (IP: 172.29.119.179)

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

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDHcCcp3AJWsLWxeCpZMYdxa8b8ae1qntpz7seqKtYXq70VYvdsDzW7W0/0F/+1D27j9SD8ZBiQPz7mdFbQ8GguNzT+rsgYkVmyFQhR2WJ9BXsaTZ5yJU/qkxuE7W2tVeFSEpUJdj04FJmZwnFqFPaLUZD5De2r9a1flJhNssIk27z3RrgGEyOJfLlFn00NO2ckLGvZsgkm51nD0pXuUo26CsKTZbKtfakacM5dum+dk0U5C+Hj3CZKYJwKtsFpX/F0GmF3esF5vmjmJ00H0LW7VskJrmrmWQxCnVCXE5Fhh14Q7qd+sB69V3WHximr019N8c6uuB2+4trJo3EPlGDs00GilmJLqB5KFlOqGgQVyhExl2CKm+l4jkLjduat1tgZNXYopFhHCN5u2dl9TxYqAvrzba8sbY/aHXlwbprNbvcy8raOPPtpf1N4Xxw1aN/r+LGXfJ9dVevRnYKn+s2Ja2ac5NeOqVT9maBqKU39cj3NOKBRc2BVj98H8IfsRT8= postgres@db-async
```

##### Get/Save SSH-RSA Public Key From BARMAN SERVER VM
```
# rm -rf /var/lib/postgresql/.ssh/authorized_keys
# touch /var/lib/postgresql/.ssh/authorized_keys
# chmod 0600 /var/lib/postgresql/.ssh/authorized_keys
mkdir /var/lib/postgresql/.ssh
vi /var/lib/postgresql/.ssh/authorized_keys

#
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC71UafKsstN+iR60bwBIpnV02B6B/lk+ikVwuS+/yOWkmaCuBX++TAhZ40rgObqT+pkoO0fs+Saf28LmJq44A2tLWSWDv0FkINsdMAVeCl6OcljALFQmXRXRUlMCoXU99eXx8gI0/OCTKr1mbLjPExnFHBiozSnCpUqgTUGUkikko2YhwCd7Gqz1YTHif48E4jVOFfT6s3NPVOD5Nt1Pah+Q9l3R+DvgvmOVOxTxiVplOkmgmR+Y4+9IxkNEhq28C6BzL65+hoZczxGQnV4vQ6oAV5L0Fdsq9i8/I/acwXIQYzkn5s2RTTdoq0mDWDJlMpl7qt2v2SLDiEiDiwNvMbmWxKkBSBDkbkzxI7OXB3jZe2Y/vlzS8AqXAWttuW22KzDGG+dmbe2i9teGop7AHLf7ALSQh99v6vdw4smdfXN9q1jm/AopiU2tQff8qQW+AvWZRNWIqG+/9QZ+aez7lb9BHioUW0ppH/Ob1BRWZOQwVa6mu8qRT2y6+GwGlRCGc= barman@bms
#

chmod 700 /var/lib/postgresql/.ssh
chmod 600 /var/lib/postgresql/.ssh/authorized_keys
restorecon -r -vv /var/lib/postgresql/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
ssh barman@172.29.123.90
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
host    replication     all             *172.29.123.90/32       trust*
*host    replication     streaming_barman 172.29.123.90/32       trust*
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
