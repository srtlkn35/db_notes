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
- db-async: 172.24.138.9
- db-sync: 172.24.136.12
- db-vssh: 172.24.130.192
- bms: 172.24.139.61

##### MULTIPASS VM Deletion
```
multipass delete bms
multipass purge
```


# BARMAN SERVER INITIALIZATION (IP: 172.24.139.61)
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

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC58O1t58kNll+AzzC64zyFjBHm7cLsGL0uax6ZdqWwFrgyjIFkAYiKCzFLP+KD1BH+G2xxJCm5Yj7HP/2lRRqNlrl4ZuzppSNn0/WDXz+ozxd5NBHwtDc0tetzkbHWTX0Ux8asyTPhDotlOzxYG+YSeLvWvK5K9zuU6DjRvK4nGmaGEc6nSRKe6Pz03U8b3IATMcArJqJmIT70D9w/vAUmVzX9VRH5qNnBcIOCFfSpqGJanl2902w8e4z+cyQdClbPb96KCO3AwxgqXp+Qma8MCjthJzwpimRLYKqGnl6ZHIXNn+wOP3nwK6G71HZaEZ3gp1SqyNz1142015LPiDeT4VOfc2kcmWq307kXnqJfsdNuujK72lWPThOR5q5LTizN3cl7ofTzctjB0M6xK8etjYWIaQzvbXdinFiVjCXlkUtJsGnMsUX/f+vqPnNIZhI9PFEQIDtYEf10N+6Z98P5VehezfXRo6VsCRkJZUvJPQTKCOmzD7ZVQ638NOI7LmU= barman@bms
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
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDgQbU0psPS9FUMLa/3X0LBRJPzJlembx+AvZf+BzKRyKJ+lMp6lo/pM1IwsKt9gL8yo73rwl6SCqYh2B+eMd8q+CrIP06TWLpVO6UkbQHTLLEk2725ioxhn5i4f1yjL7aWC4H+Mn7cdKgiaL4v0El/2rniZrQivpXSWfqAjfxr7vAJoVvpeXHUvImWxMehlldw1eraG6eKQBYi5QJeivVxHPZePhvjZFm3zJ5zuBaDf5Nkq+jSo6dpnh+ijNFiGDwPqClz/iujTP5/uieoCbd7mbj6/H9ItlSu1YS9u176vUlMkKqsDH8Zes9ToZJeo38QKM7JrSf3InCGyOvZO4uQkNLQ6Hwnmzv+1HnSWea7SCOggSTPi5hCleq0FmisffwzNpun7qzioKMBpIYa0+QthMF1Bcsc7U64IyjqYbenSIsuC3wp/m02uNWWDs+FAnVw/BjFpZ6MLEKpr3TZchLHXPjgYYLhGFeUN31iOC+IqaEENStQomV5DJEDoHjPKZE= postgres@db-async
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDPl/5yXv383M0gIlhy5/CwNp0ZvXJhncvn0BYBOJP0ZZ2u9GvGt6jv/VuazCWndN5u74M4b24FB7qfwGqPxJ1vek/td+/sIkOnSjK8xqDJg3kCjcvDj9dhgNGulx39HvixntmK+F2H09/mlYvm9GhDh+jBbgLCIsubNLL4TcTroPnHv+OKC5+p1nop1fgI0Bk+Hlg7U2BWXVrd0ebcNjbbH7kjxkxbI0Yt0G8QjczjOdbJoOiR64UMS40FeywXWd09v1Zm9Jr8lM/3O/nccBl+ewzdcAqCMfXHk3SO23BWcUjI1Efx24bKgCckKCS+87duAtUKbIGpkdWc06HpPdnm4Ek0/DlRmD7FM9TgAZCBA1UJXJAMto/vAgpGZnkq10zbV+/bPiVdyYZ6zqc+wuws02bCxwswazNtscPVrEa9zDpI8sKY+wyJ9pE2XpHJTMP85UkQFbaNw+QhRo/PgI5BujHlITUSaTIl/mMTFz+frt4gPA3xn102N+J/t5jnFO0= postgres@db-sync
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDRtb+9YG7LMIfKDIlRD1nZW2N1hS9boAUgMN79WMpo/xRFoWcGwwoK6NmuNSjMV8JFE/vfZQt3ha5jLiRF09hKN+0M4G/NnLBnnnNLW6tSimpVrehJ5isXBxRDGfxhuL3pzkASidtwGXHieMUZqa7JuOg1/seqIPYImh00nfz1nYPkrXO4qj6vRWnihRFjbgUwpGqNfQX1BCXY9+L3Xg5UGMY+iTSUV1wFtvxonM37qMNAwglvlPleSLcHIObIt1qmfCjHYEdrNihkiRa2LEck2O9PIUKw/QNVSNmxqKuX9gLLVdUF1YouxjlvhXMX4gp0bxO5XRbV98L9MQ6uGPssayG1sLljSWMHZ5ou6liiredTHty/QJyB9/p+5VD9X8zmZN+SlIB09QCLcD4580upC/obcDvhRfYih+vs6f9sWAs8uiUT37ZFxYidzSfuF5bamcxdoUSm8GKtN5+aWNrel4/ro9IorOE7GQrsm2513364mIhSJY54BKenoKneBrU= postgres@db-vssh
#

chmod 700 /var/lib/barman/.ssh
chmod 600 /var/lib/barman/.ssh/authorized_keys
restorecon -r -vv /var/lib/barman/.ssh/authorized_keys
```

##### Test SSH TO BARMAN SERVER VM
```
# postgres@db-async
ssh postgres@172.24.138.9
exit

# postgres@db-sync
ssh postgres@172.24.136.12
exit

# postgres@db-vssh
ssh postgres@172.24.130.192
exit
```

##### Set Postgress User's Password
```
cat <<EOF | tee /var/lib/barman/.pgpass
172.24.138.9:5432:postgres:barman:welcome
172.24.138.9:5432:postgres:streaming_barman:welcome
172.24.136.12:5432:postgres:barman:welcome
172.24.136.12:5432:postgres:streaming_barman:welcome
172.24.130.192:5432:postgres:barman:welcome
172.24.130.192:5432:postgres:streaming_barman:welcome
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
# change 'conninfo = host=pg_async' to 'conninfo = host=172.24.138.9'
# change 'streaming_conninfo = host=pg_async' to 'conninfo = host=172.24.138.9'
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
# change 'conninfo = host=pg_sync' to 'conninfo = host=172.24.136.12'
# change 'streaming_conninfo = host=pg_sync' to 'conninfo = host=172.24.136.12'
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
# change 'ssh postgres@pg_vssh' to 'ssh postgres@172.24.130.192'
# change 'conninfo = host=pg_vssh' to 'conninfo = host=172.24.130.192'
# uncomment 'reuse_backup'
# uncomment 'path_prefix'

# WAL ARCHIVE OR RSYNC OPTION
# WAL ARCHIVE
# add streaming_conninfo below conninfo
streaming_conninfo  =host=172.24.130.192 user=streaming_barman
# RSYNC
#DO NOTHING
```

# Test Connections
```
sudo su - barman

psql -c 'SELECT version()' -U barman -d postgres -h 172.24.138.9 -p 5432
psql -c 'SELECT version()' -U barman -d postgres -h 172.24.136.12 -p 5432
psql -c 'SELECT version()' -U barman -d postgres -h 172.24.130.192 -p 5432
```
