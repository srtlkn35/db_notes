# MULTIPASS VM

##### VM: BARMAN Server
```
multipass launch --name bms 20.04 -c 1 -m 1G -d 30G
multipass shell bms
```

##### IP Adresses:
```
ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1  -d'/'
```

##### Replace All IP Addresses in Document
- db-async: 172.29.119.179
- db-sync: 172.29.113.229
- db-vssh: 172.29.112.36
- bms: 172.29.123.90

##### MULTIPASS VM Deletion
```
multipass delete bms
multipass purge
```


# BARMAN SERVER INITIALIZATION (IP: 172.29.123.90)
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

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC71UafKsstN+iR60bwBIpnV02B6B/lk+ikVwuS+/yOWkmaCuBX++TAhZ40rgObqT+pkoO0fs+Saf28LmJq44A2tLWSWDv0FkINsdMAVeCl6OcljALFQmXRXRUlMCoXU99eXx8gI0/OCTKr1mbLjPExnFHBiozSnCpUqgTUGUkikko2YhwCd7Gqz1YTHif48E4jVOFfT6s3NPVOD5Nt1Pah+Q9l3R+DvgvmOVOxTxiVplOkmgmR+Y4+9IxkNEhq28C6BzL65+hoZczxGQnV4vQ6oAV5L0Fdsq9i8/I/acwXIQYzkn5s2RTTdoq0mDWDJlMpl7qt2v2SLDiEiDiwNvMbmWxKkBSBDkbkzxI7OXB3jZe2Y/vlzS8AqXAWttuW22KzDGG+dmbe2i9teGop7AHLf7ALSQh99v6vdw4smdfXN9q1jm/AopiU2tQff8qQW+AvWZRNWIqG+/9QZ+aez7lb9BHioUW0ppH/Ob1BRWZOQwVa6mu8qRT2y6+GwGlRCGc= barman@bms
```

##### Get/Save SSH-RSA Public Key From BARMAN SERVER VM
```
# rm -rf /var/lib/barman/.ssh/authorized_keys
# touch /var/lib/barman/.ssh/authorized_keys
# chmod 0600 /var/lib/barman/.ssh/authorized_keys
mkdir /var/lib/barman/.ssh
vi /var/lib/barman/.ssh/authorized_keys

# postgres@db-async
# postgres@db-sync
# postgres@db-vssh
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDHcCcp3AJWsLWxeCpZMYdxa8b8ae1qntpz7seqKtYXq70VYvdsDzW7W0/0F/+1D27j9SD8ZBiQPz7mdFbQ8GguNzT+rsgYkVmyFQhR2WJ9BXsaTZ5yJU/qkxuE7W2tVeFSEpUJdj04FJmZwnFqFPaLUZD5De2r9a1flJhNssIk27z3RrgGEyOJfLlFn00NO2ckLGvZsgkm51nD0pXuUo26CsKTZbKtfakacM5dum+dk0U5C+Hj3CZKYJwKtsFpX/F0GmF3esF5vmjmJ00H0LW7VskJrmrmWQxCnVCXE5Fhh14Q7qd+sB69V3WHximr019N8c6uuB2+4trJo3EPlGDs00GilmJLqB5KFlOqGgQVyhExl2CKm+l4jkLjduat1tgZNXYopFhHCN5u2dl9TxYqAvrzba8sbY/aHXlwbprNbvcy8raOPPtpf1N4Xxw1aN/r+LGXfJ9dVevRnYKn+s2Ja2ac5NeOqVT9maBqKU39cj3NOKBRc2BVj98H8IfsRT8= postgres@db-async
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDDN/P6PMx9WmKylphOhiz+9rgi+XBSS/OqsA1AwECCApaHripBK2RCqfJnbP0LzbBgz8/x9e17pgd1K7NtRGy7EWX5+4JKq+FGmwJDJbts1IEUKmNx4ifwxsI1TxJaPW4YYNt00cVFM4A59qwVGmtaLqpD4vvG13bu8QgXvdjT/uJjfZHyBEra3Aoz13PQE3CdwJ96Q9nB6Wbt1idNKPsE9+EGe6U8RJQBxkAq3KXlYhd2fxRPvL2nsB74yj8M2t12Ibpo1gVfKP60RulJseXBjLJ2kNGR9cgKgBYDig5YegRsDC9+nEZV4Sj3lknmm7Jycuc25tQTe6D0lJTT+6r8x3LM3G4cyGUjnmS/plYvKZVtNJPV9stU/I8Y1UR/mnBut1YJJkFJ+RU8SwkmZPXhGdSNbgalZB88UZVlsUBc0j2NrqW7bWsgdlIm7sGWgR0h5DRb6jdRSl77eiy4Hkce4tPYjxdIDWheYIMg1FAJUMu25uY4JHAQEx56EBpCYlM= postgres@db-sync
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC8uKzHN+qMaSFM3vxwV/VNCq1vLgY6Rwc80KFgUZb0oHKCuNiuVHrYfgCmYZofAjZhA8GM2hi7PX4Rv43XBwYsqqVFaAWfa2XqtO6YvtJXg356B8++O+ZOidq2huBRg86iDBtARrNvtwSZve7IkPWkROLK4LHp0bNcRl81AwIzjparUqM+a18CWY42bYrIl0JuxrHgeYf+B1/mEvXesa9gvK6iukRGTzAF5o5Gt7ikYdqvGPagdCQo4Lfakfyzt8gQwx2tEN/FpRYGoNVDj7sR4UPkicVv++urQ716kEW7Ma/QQUt+IKcNmmsjin9qneHycJ72nykCBx6hQqS00z2K8e+4knVo9URag3vyXDSJn/QcMzgjc2P8wG3uS+OSjdud7njTyIKr8Me0rvpSrkeEAooAZJpHm2AL82zhQTRmOh/ScR0dKKEQed97Q+SdtOG7CDBQvqXX3RuJ3FH5TG00AFzD6pEoVDrixzB7tGa9NLoRuv8/lR6WZ7u+NOdYXP8= postgres@db-vssh
#

chmod 700 /var/lib/barman/.ssh
chmod 600 /var/lib/barman/.ssh/authorized_keys
restorecon -r -vv /var/lib/barman/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
# postgres@db-async
ssh postgres@172.29.119.179
exit

# postgres@db-sync
ssh postgres@172.29.113.229
exit

# postgres@db-vssh
ssh postgres@172.29.112.36
exit
```

##### Set Postgress User's Password
```
cat <<EOF | tee /var/lib/barman/.pgpass
172.29.119.179:5432:postgres:barman:welcome
172.29.119.179:5432:postgres:streaming_barman:welcome
172.29.113.229:5432:postgres:barman:welcome
172.29.113.229:5432:postgres:streaming_barman:welcome
172.29.112.36:5432:postgres:barman:welcome
172.29.112.36:5432:postgres:streaming_barman:welcome
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
## change 'post_backup_script=barman generate-manifest ${BARMAN_SERVER} ${BARMAN_BACKUP_ID}'
```

# streamingAsync
##### Configure Barman for streamingAsync (Using streaming-server.conf-template)
```
# sudo rm -rf /etc/barman.d/pg_async.conf
sudo cp /etc/barman.d/streaming-server.conf-template /etc/barman.d/pg_async.conf
sudo vi /etc/barman.d/pg_async.conf

# change '[streaming]' to 'pg_async'
# change 'description = "Example of PostgreSQL Database (Streaming Async)"'
# change 'conninfo = host=pg_async' to 'conninfo = host=172.29.119.179'
# change 'streaming_conninfo = host=pg_async' to 'conninfo = host=172.29.119.179'
# uncomment 'streaming_backup_name'
# uncomment 'create_slot'
# uncomment 'streaming_archiver_name'
# uncomment 'streaming_archiver_batch_size'
# uncomment 'path_prefix'
```

# streamingSync
##### Configure Barman for streamingSync (Using streaming-server.conf-template)
```
# sudo rm -rf /etc/barman.d/pg_sync.conf
sudo cp /etc/barman.d/streaming-server.conf-template /etc/barman.d/pg_sync.conf
sudo vi /etc/barman.d/pg_sync.conf

# change '[streaming]' to 'pg_sync'
# change 'description = "Example of PostgreSQL Database (Streaming Sync)"'
# change 'conninfo = host=pg_sync' to 'conninfo = host=172.29.113.229'
# change 'streaming_conninfo = host=pg_sync' to 'conninfo = host=172.29.113.229'
# uncomment 'streaming_backup_name'
# uncomment 'create_slot'
# uncomment 'streaming_archiver_name'
# uncomment 'streaming_archiver_batch_size'
# uncomment 'path_prefix'
```

# viaSSH
##### Configure Barman for viaSSH (Using ssh-server.conf-template)
```
# sudo rm -rf /etc/barman.d/pg_vssh.conf
sudo cp /etc/barman.d/ssh-server.conf-template /etc/barman.d/pg_vssh.conf
sudo vi /etc/barman.d/pg_vssh.conf

# change '[ssh]' to 'pg_vssh'
# change 'description = "Example of PostgreSQL Database (via SSH)"'
# change 'ssh postgres@pg_vssh' to 'ssh postgres@172.29.112.36'
# change 'conninfo = host=pg_vssh' to 'conninfo = host=172.29.112.36'
# uncomment 'reuse_backup'
# uncomment 'path_prefix'

# WAL ARCHIVE OR RSYNC OPTION
# WAL ARCHIVE
# add streaming_conninfo below conninfo
streaming_conninfo  =host=172.29.112.36 user=streaming_barman
# RSYNC
#DO NOTHING
```
