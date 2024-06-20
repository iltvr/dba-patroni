# Cтроим высокодоступный кластер на базе PostgreSQL с Patroni, etcd и HAProxy.
## Введение
Высокая доступность является жизненно важной частью современной архитектуры и современных систем.  В этом сообщении блога описывается возможный подход к созданию высокодоступного кластера PostgreSQL с использованием Patroni и HAProxy.  Высокая доступность позволяет узлу БД PostgreSQL автоматически перенаправлять работу на другой доступный узел БД PostgreSQL в случае сбоя.
## План действий
- Создать инфраструктуру из 5 виртуальных машин в облаке Yandex Cloud.

    | Hostname   | IP Address    | Used as              |
    |------------|---------------|----------------------|
    | postgres01 | 10.131.0.15  | PostgreSQL Server    |
    | postgres02 | 10.131.0.32  | PostgreSQL Server    |
    | postgres03 | 10.131.0.6  | PostgreSQL Server    |
    | etcd       | 10.131.0.5  | Cluster Data Store   |
    | haproxy    | 10.131.0.3  | Load Balancer        |

- Установить PostgreSQL на виртуальные машины postgres01, postgres02, postgres03
- Установить и настроить etcd на виртуальной машине etcd
- Настроить Patroni на виртуальных машинах postgres01, postgres02, postgres03
- Установить и настроить HAProxy на виртуальной машине haproxy
- Проверить работу кластера PostgreSQL

## Создание виртуальных машин PostgreSQL в облаке Yandex Cloud
Создадим виртуальные машины в облаке Yandex Cloud.  Для этого воспользуемся консолью управления облаком.
### Создание виртуальной машины postgres01 (postgres02, postgres03)
- Имя: postgres01
- ОС: Ubuntu 20.04
- Сеть: default
- Зона: ru-central1-d
```bash
yc compute instance create \
    --name haproxy \
    --zone ru-central1-d \
    --network-interface subnet-name=default-ru-central1-d,nat-ip-version=ipv4 \
    --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
    --memory=4 \
    --cores=2 \
    --ssh-key ~/.ssh/yc_key.pub
```
<details>
<summary>Вывод</summary>

```bash
id: fv4r6nh3l4so2av4djd4
folder_id: b1g9moofodgrt0vclis4
created_at: "2024-06-19T14:09:06Z"
name: postgres01
zone_id: ru-central1-d
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fv4kdoe7bmenk14eqntc
  auto_delete: true
  disk_id: fv4kdoe7bmenk14eqntc
network_interfaces:
  - index: "0"
    mac_address: d0:0d:1b:35:e2:3a
    subnet_id: fl8pekfcrllb751nqckk
    primary_v4_address:
      address: 10.131.0.15
      one_to_one_nat:
        address: 158.160.149.207
        ip_version: IPV4
gpu_settings: {}
fqdn: fv4r6nh3l4so2av4djd4.auto.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
</details>

### Установка необходимого ПО
- postgresql-15 – PostgreSQL 15
- patroni – Patroni, фреймворк для создания высокодоступных кластеров PostgreSQL.
- python3-etcd – Библиотека Python 3 для работы с etcd.
- python3-psycopg2 – Адаптер PostgreSQL для Python 3.

```bash
sudo apt update
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install postgresql-15 postgresql-server-dev-15 patroni python3-etcd python3-psycopg2

```
### Настройка PostgreSQL
Настроим PostgreSQL на виртуальной машине postgres01 (postgres02, postgres03).
- Остановим PostgreSQL
- Создадим симлинк на бинарники PostgreSQL для того, чтобы patroni смог запускать бинарники PostgreSQL.
```bash
sudo systemctl stop postgresql patroni
sudo ln -s /usr/lib/postgresql/15/bin/* /usr/sbin/
```
### Настройка Patroni
- Проверим, что patroni установлен.
- Проверим что psql доступен.
- Проверим версию patroni.
```bash
which patroni
which psql
patroni --version
```
## Установка etcd
Установим etcd на виртуальной машине etcd.
- Установим etcd
- Создадим конфигурационный файл etcd
```bash
sudo apt update
sudo apt install etcd-server etcd-client
```
### Создание конфигурационного файла etcd
- ETCD_LISTEN_PEER_URLS – URL, по которому etcd будет принимать запросы от других узлов кластера.
- ETCD_LISTEN_CLIENT_URLS – URL, по которому etcd будет принимать запросы от клиентов.
- ETCD_INITIAL_ADVERTISE_PEER_URLS – URL, по которому etcd будет коммуницировать с другими узлами кластера.
- ETCD_INITIAL_CLUSTER – Начальная настройка кластера для начальной загрузки.
- ETCD_ADVERTISE_CLIENT_URLS – Список URL-адресов клиентов этого участника для коммуникации с остальной частью кластера.
- ETCD_INITIAL_CLUSTER_TOKEN – Токен кластера.
- ETCD_INITIAL_CLUSTER_STATE – Состояние кластера.

```bash
sudo tee /etc/default/etcd <<EOF
ETCD_LISTEN_PEER_URLS="http://10.131.0.5:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.131.0.5:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.131.0.5:2380"
ETCD_INITIAL_CLUSTER="default=http://10.131.0.5:2380,"
ETCD_ADVERTISE_CLIENT_URLS="http://10.131.0.5:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```
### Запуск etcd
```bash
sudo systemctl restart etcd
sudo systemctl status etcd
```
<detais>
<summary>Вывод</summary>

```bash
● etcd.service - etcd - highly-available key value store
     Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-06-19 14:56:33 UTC; 35s ago
       Docs: https://github.com/coreos/etcd
             man:etcd
   Main PID: 1533 (etcd)
      Tasks: 12 (limit: 4630)
     Memory: 19.9M
     CGroup: /system.slice/etcd.service
             └─1533 /usr/bin/etcd
```
</details>

### Проверим список участников кластера
```bash
etcdctl member list
8e9e05c52164694d: name=fv45t14c032pffgkl6rr peerURLs=http://localhost:2380 clientURLs=http://10.131.0.5:2379 isLeader=true
```
### Проверим подключение к etcd
```bash
nc -vz 10.131.0.5 2379
Connection to 10.131.0.5 2379 port [tcp/*] succeeded!
```

## Развертывание PostgreSQL кластера с Patroni на виртульных машинах postgres01, postgres02, postgres03
### Создание конфигурационного файла patroni
- Заполним конфигурационный файл patroni

```bash
sudo cp /etc/patroni/config.yml.in /etc/patroni/config.yml
sudo tee /etc/patroni/config.yml <<EOF
scope: postgres
namespace: /db/
name: postgres01

restapi:
  listen: 10.131.0.15:8008
  connect_address: 10.131.0.15:8008

etcd3:
  host: 10.131.0.5:2379

# The bootstrap configuration. Works only when the cluster is not yet initialized.
# If the cluster is already initialized, all changes in the `bootstrap` section are ignored!
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      pg_hba:
      - host replication replicator 127.0.0.1/32 scram-sha-256
      - host replication replicator 10.131.0.15/0 scram-sha-256
      - host replication replicator 10.131.0.32/0 scram-sha-256
      - host replication replicator 10.131.0.6/0 scram-sha-256
      - host all all 0.0.0.0/0 scram-sha-256
      users:
        admin:
            password: admin
            options:
                - createrole
                - createdb
      parameters:
#        wal_level: hot_standby
#        hot_standby: "on"
#        max_connections: 100
#        max_worker_processes: 8
#        wal_keep_segments: 8
#        max_wal_senders: 10
#        max_replication_slots: 10
#        max_prepared_transactions: 0
#        max_locks_per_transaction: 64
#        wal_log_hints: "on"
#        track_commit_timestamp: "off"
#        archive_mode: "on"
#        archive_timeout: 1800s
#        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
#      recovery_conf:
#        restore_command: cp ../wal_archive/%f %p

  initdb:
  - encoding: UTF8
  - data-checksums
  - auth: scram-sha-256

postgresql:
  listen: 10.131.0.15:5432
  connect_address: 10.131.0.15:5432

  data_dir: /var/lib/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: rep-pass
    superuser:
      username: postgres
      password: secretpassword

  parameters:
    unix_socket_directories: '/var/run/postgresql/'  # parent directory of data_dir

tags:
    # failover_priority: 1
    noloadbalance: false
    clonefrom: false
    nosync: false
    nostream: false
EOF
```
### Подготовим директории для Patroni
```bash
sudo mkdir -p /var/lib/patroni
sudo chown -R postgres:postgres /var/lib/patroni
sudo chmod 700 /var/lib/patroni
```
### Запустим Patroni
```bash
sudo systemctl start patroni
sudo systemctl status patroni
```
<details>
<summary>Вывод</summary>

```bash
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
     Loaded: loaded (/lib/systemd/system/patroni.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-06-19 17:49:51 UTC; 23s ago
   Main PID: 11307 (patroni)
      Tasks: 13 (limit: 4630)
     Memory: 89.2M
     CGroup: /system.slice/patroni.service
             ├─11307 /usr/bin/python3 /usr/bin/patroni /etc/patroni/config.yml
             ├─11345 postgres -D /var/lib/patroni --config-file=/var/lib/patroni/postgresql.conf --listen_addresses=10.131.0.15 --port=5432 --cluster_name=postgres --wal_level=replica --hot_standby=on --max_connec>
             ├─11347 postgres: postgres: checkpointer
             ├─11348 postgres: postgres: background writer
             ├─11353 postgres: postgres: walwriter
             ├─11354 postgres: postgres: autovacuum launcher
             ├─11355 postgres: postgres: logical replication launcher
             └─11358 postgres: postgres: postgres postgres 10.131.0.15(37226) idle
```
</details>

### Проверим работу кластера
```bash
sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: postgres (7382273075804097608) ----+----+-----------+
| Member     | Host        | Role   | State   | TL | Lag in MB |
+------------+-------------+--------+---------+----+-----------+
| postgres01 | 10.131.0.15 | Leader | running |  1 |           |
+------------+-------------+--------+---------+----+-----------+
```
### Настроим остальные узлы кластера и проверим работу клаустреа
```bash
yc-user@fv4r6nh3l4so2av4djd4:~$ sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: postgres (7382273075804097608) -------+----+-----------+
| Member     | Host        | Role    | State     | TL | Lag in MB |
+------------+-------------+---------+-----------+----+-----------+
| postgres01 | 10.131.0.15 | Leader  | running   |  2 |           |
| postgres02 | 10.131.0.32 | Replica | streaming |  2 |         0 |
| postgres03 | 10.131.0.6  | Replica | streaming |  2 |         0 |
+------------+-------------+---------+-----------+----+-----------+
```

### Отключим автоматический запуск PostgreSQL при загрузке системы, поскольку Patroni управляет новым сервером PostgreSQL.
```bash
sudo systemctl disable --now postgresql
Synchronizing state of postgresql.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable postgresql
Removed /etc/systemd/system/multi-user.target.wants/postgresql.service.
```

## Установка HAProxy в качестве балансировщика нагрузки
### Настроим hosts
```bash
sudo tee -a /etc/hosts <<EOF
10.131.0.15 postgres01
10.131.0.32 postgres02
10.131.0.6 postgres03
EOF
```
### Установим HAProxy
```bash
sudo apt update
sudo apt install haproxy
```
### Создадим конфигурационный файл HAProxy
```bash
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
sudo tee /etc/haproxy/haproxy.cfg <<EOF
global
    maxconn 100
    log 127.0.0.1 local2

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:8080
    stats enable
    stats uri /

listen postgres
    bind *:5432
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgres01 10.131.0.15:5432 maxconn 100 check port 8008
    server postgres02 10.131.0.32:5432 maxconn 100 check port 8008
    server postgres03 10.131.0.6:5432 maxconn 100 check port 8008
EOF
```
### Запустим HAProxy
```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy
```
<details>
<summary>Вывод</summary>

```bash
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-06-19 19:14:46 UTC; 1min 6s ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
    Process: 1663 ExecStartPre=/usr/sbin/haproxy -Ws -f $CONFIG -c -q $EXTRAOPTS (code=exited, status=0/SUCCESS)
   Main PID: 1679 (haproxy)
      Tasks: 3 (limit: 4630)
     Memory: 2.4M
     CGroup: /system.slice/haproxy.service
             ├─1679 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
             └─1681 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Jun 19 19:14:46 fv4al8i00epi5gtgd6s6 systemd[1]: Starting HAProxy Load Balancer...
Jun 19 19:14:46 fv4al8i00epi5gtgd6s6 haproxy[1679]: [NOTICE] 170/191446 (1679) : New worker #1 (1681) forked
Jun 19 19:14:46 fv4al8i00epi5gtgd6s6 systemd[1]: Started HAProxy Load Balancer.
Jun 19 19:14:47 fv4al8i00epi5gtgd6s6 haproxy[1681]: [WARNING] 170/191447 (1681) : Server postgres/postgres02 is DOWN, reason: Layer7 wrong status>
Jun 19 19:14:48 fv4al8i00epi5gtgd6s6 haproxy[1681]: [WARNING] 170/191448 (1681) : Server postgres/postgres03 is DOWN, reason: Layer7 wrong status>
```
</details>

### Посетим страницу статистики HAProxy
![haproxy.png](/public/haproxy.png)

- на странице статистики HAProxy можно увидеть, что сервер postgres02 и postgres03 не доступны. Это связано с тем, что Patroni настроен на работу с сервером postgres01 в качестве лидера, а серверы postgres02 и postgres03 находятся в режиме реплики.

## Проверим работу кластера.
### Подключимся к кластеру PostgreSQL извне
```bash
➜  dba-patroni git:(main) ✗ psql -h 158.160.159.39 -U postgres -d postgres
Password for user postgres:
psql (15.5 (Homebrew), server 15.7 (Ubuntu 15.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(3 rows)

postgres=#
```

## Проверим работу кластера при сбое одного из узлов
### Cписок участников кластера
```bash
sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: postgres (7382273075804097608) -------+----+-----------+
| Member     | Host        | Role    | State     | TL | Lag in MB |
+------------+-------------+---------+-----------+----+-----------+
| postgres01 | 10.131.0.15 | Leader  | running   |  2 |           |
| postgres02 | 10.131.0.32 | Replica | streaming |  2 |         0 |
| postgres03 | 10.131.0.6  | Replica | streaming |  2 |         0 |
+------------+-------------+---------+-----------+----+-----------+
```

### Имитация сбоя сервера postgres01
```bash
Password for user postgres:
psql (15.7 (Ubuntu 15.7-1.pgdg20.04+1))
Type "help" for help.

postgres=#
SELECT inet_server_addr() AS hostname;
  hostname
-------------
 10.131.0.15
(1 row)

postgres=# \q
```
```bash
sudo systemctl stop patroni
sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: postgres (7382273075804097608) -----+----+-----------+
| Member     | Host        | Role    | State   | TL | Lag in MB |
+------------+-------------+---------+---------+----+-----------+
| postgres01 | 10.131.0.15 | Replica | stopped |    |   unknown |
| postgres02 | 10.131.0.32 | Replica | running |  2 |         0 |
| postgres03 | 10.131.0.6  | Leader  | running |  3 |           |
+------------+-------------+---------+---------+----+-----------+
```
- сервер postgres01 перешел в режим stopped, сервер postgres03 стал лидером.

### Посетим страницу статистики HAProxy
![haproxy.png](/public/haproxy_failover_1.png)

- на странице статистики HAProxy можно увидеть, что статус сервера postgres01 "DOWN", а сервер postgres03 имееет статус "UP".

### Подключимся к кластеру PostgreSQL извне
```bash
psql -h 158.160.159.39 -U postgres -d postgres
Password for user postgres:
psql (15.5 (Homebrew), server 15.7 (Ubuntu 15.7-1.pgdg20.04+1))
Type "help" for help.
➜  dba-patroni git:(main) ✗ psql -h 158.160.159.39 -U postgres -d postgres
Password for user postgres:
psql (15.5 (Homebrew), server 15.7 (Ubuntu 15.7-1.pgdg20.04+1))
Type "help" for help.
postgres=# SELECT inet_server_addr() AS hostname;
  hostname
------------
 10.131.0.6
(1 row)

postgres=# \q
```
- подключились к серверу postgres03.

### Восстановим сервер postgres01
```bash
sudo systemctl start patroni
sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: postgres (7382273075804097608) -------+----+-----------+
| Member     | Host        | Role    | State     | TL | Lag in MB |
+------------+-------------+---------+-----------+----+-----------+
| postgres01 | 10.131.0.15 | Replica | running   |  2 |         0 |
| postgres02 | 10.131.0.32 | Replica | streaming |  3 |         0 |
| postgres03 | 10.131.0.6  | Leader  | running   |  3 |           |
+------------+-------------+---------+-----------+----+-----------+
```
- сервер postgres01 восстановлен и перешел в режим running.

## Выводы
В данной работе описан возможный подход к созданию высокодоступного кластера PostgreSQL с использованием Patroni и HAProxy; протестирована работа кластера PostgreSQL в случае сбоя одного из узлов.
- Patroni позволяет создать высокодоступный кластер PostgreSQL, который автоматически перенаправляет работу на другой доступный узел БД PostgreSQL в случае сбоя.
- HAProxy позволяет балансировать нагрузку между узлами кластера PostgreSQL.
- etcd используется для хранения конфигурации кластера Patroni.





