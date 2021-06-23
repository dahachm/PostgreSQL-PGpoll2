# Отказоустойчивый кластер на базе PostgreSQL 9.6 + PGpoll II

Установка и настройка производится в ОС Astra Linux SE 1.6

(Здесь рисунок с нодами и указанем их IP)

## Установка

```
$ sudo apt install ssh rsync -y
```
```
$ sudo apt install postgresql postgresql-contrib postgresql-client -y
```
```
$ sudo apt install pgadmin3 -y
```
```
$ sudo apt install pgpool2 -y
```

## Настройка сети

### 1. Обновление /etc/hosts

Добавить следующие записи в файл /etc/hosts:
  
  ```
  192.168.12.6    master
  192.168.12.7    slave
  192.168.12.4    balancer
  ```
   
И к записи с резолвом адреса 127.0.0.1 добавить новое имя соответвующего хоста, где 'hostname' принимает одно из значений {'master','slave','balancer'}:
  
  ```
  127.0.0.0    localhost <hostname>
  ```
    
### 2. Смена hostname

  ```
  $ sudo hostnamectl set-hostname <hostname>
  ```

## Настройка пользователя postgres

### 1. Изменение пароля
    
  В системе: 
  
    ```
    $ sudo passwd postgres
    ```
  
  В БД:
  
    ```
    $su postgres -c "psql"
    ```
    ```
    #: ALTER USER postgres WITH PASSWORD '<password>';
    ```

### 2. Настройка прав доступа к таблице мандатных привилегий

    ```
    usermod -a -G shadow postgres
    setfacl -d -m u:postgres:r /etc/parsec/macdb
    setfacl -R -m u:postgres:r /etc/parsec/macdb
    setfacl -m u:postgres:rx /etc/parsec/macdb
    setfacl -d -m u:postgres:r /etc/parsec/capdb
    setfacl -R -m u:postgres:r /etc/parsec/capdb
    setfacl -m u:postgres:rx /etc/parsec/capdb
    ```
    
### 3. Настройка доступа по SSH

    ```
    $ su postgres
    $ ssh-keygen -P '' -f /var/lib/postgresql/.ssh/id_rsa
    $ ssh-copy-id -i /var/lib/postgresql/.ssh/id_rsa remote_host
    ```
    
  Где 'remote_host' - имя или IP адрес удаленного хоста.
  
  Нужно обеспечить возможность входа с 'balancer' на 'master' и на 'slave', с 'master' на 'slave' и со 'slave' на 'master'.
  
  (Сюда тоже можно рисунок, кто куда должен входить по SSH без пароля)
  
## Открыть порты 

    ```
    $ sudo ufw allow in ssh
    $ sudo ufw allow out ssh
    $ sudo ufw allow in 5432/tcp
    $ sudo ufw allow out 5432/tcp
    $ sudo ufw allow in 5433/tcp
    $ sudo ufw allow out 5433/tcp
    $ sudo ufw allow in 9898/tcp
    $ sudo ufw allow out 9898/tcp
    ```
    
  (фото статуса настроек файервола)

## Настройка стриминговой (stream) репликации

1. Остановить кластер на ведомом сервере (**slave**)

  ```
  $ sudo pg_ctlcluster p.6 main stop
  ```

2. Отредактировать файлы конфигурации

  Файл /etc/postgresql/9.6/main/postgresql.conf на **master**:
  
  ```
  listen_address = 'localhost, 192.168.12.6'
  
  wal_level = replica
  
  max_wal_senders = 2
  wal_keep_segments = 32 
  ```
  
  Файл /etc/postgresql/9.6/main/pg_hba.conf на **master**:
  
  ```
  host     replication     postgres        192.168.12.7/24         trust
  ```
  
  Файл /etc/postgresql/9.6/main/postgresql.conf на **slave**:
  
  ```
  listen_address = 'localhost, 192.168.12.7
    
  hot_standby = on
  ```
  

3. Перенос резервной копии данных на ведомый сервер

Удалить  все  файлы  в  каталоге  ведомого  сервера  за  исключением
*pg_audit.conf*:

  ```
  $ cd /var/lib/postgresql/$VERSION/$SLAVE/
  $ sudo ls | grep -v pg_audit.conf | xargs rm -rf
  ```
  
От имени пользователя postgres на **master** сервере:

  ```
  $ su postgres
  $ psql -h master -p 5432 -U postgres -d template1 -c "SELECT
pg_start_backup(’initial’);"
  $ rsync -a /var/lib/postgresql/9.6/main/*
postgres@192.168.12.7:/var/lib/postgresql/9.6/main/
--exclude pg_log/* --exclude pg_xlog/* --exclude postmaster.* --exclude
pg_audit.conf >> /dev/null
  $ psql -h master -p 5432 -U postgres -d template1 -c "SELECT
pg_stop_backup();"
  ```
  
4. Запуск кластера на ведомом сервере (**slave**)
  
  ```
  $ sudo pg_ctlcluster 9.6 main start
  ```
  
  Среди активных процессов на машине **master** должен появится процесс **sender**:
  
  
  Среди активных процессов на машине **slave** должен появится процесс **receiver**:
  
5. Тестирование

  Создаем таблицу на **master**:
  
  ```
  $ su postgres -c "psql"
  ```
  ```
  =# CREATE TABLE VPN_proto (ID SERIAL PRIMARY KEY, name CHARACTER VARYING(30), tunnels_num INTEGER, crypts_num INTEGER, KZ_num INTEGER, key_start_date TIMESTAMP );
  =# INSERT INTO VPN_proto VALUES (1, 'OpenVPN', 2, 2, 5, '2021-06-14 00:01:03');
  =# INSERT INTO VPN_proto VALUES (2, 'SSTP', 1, 2, 1, '2021-05-04 04:01:00');
  =# INSERT INTO VPN_proto VALUES (3, 'IPSec', 2, 2, 6, '2021-09-28 10:15:21');
  ```
  
  (рисунок с новой таблицей)
  
  Пробуем вывести содержимое таблицы:
  
  (получилось)
  
  Причем при активном режиме "горячего резервирование" (**hot stanby**) запрещено вносить изменения в БД с запасной машины:
  
  (скрин с ошибкой при попытке удаления строки)

## Настройка механизма failover

**Failover** - это ...

### 1. Настройка pgpool2 на balancer
  
  На машине **balancer** настроить файл */etc/phpool2/pgpool.conf*:
  
  ```
  listen_addresses = '192.168.12.29'
  
  backend_hostname0 = '192.168.12.7'
  backend_port0 = 5432
  backend_weight0 = 1
  backend_data_directory0 = '/var/lib/postgresql/9.6/main'
  backend_flag0 = 'ALLOW_TO_FAILOVER'
  
  backend_hostname0 = '192.168.12.6'
  backend_port0 = 5432
  backend_weight0 = 1
  backend_data_directory0 = '/var/lib/postgresql/9.6/main'
  backend_flag0 = 'ALLOW_TO_FAILOVER'
  
  enable_pool_hba = on
  
  log_destination = 'syslog'
  
  load_balance_mode = on
  
  master_slave_mode = on 
  master_slave_sub_mode = 'stream'
  
  sr_check_period = 10
  sr_check_user = 'postgres'
  sr_check_password = '12345678'
  
  health_check_period = 10
  health_check_timeout = 20
  health_check_user = 'postgres'
  health_check_password = '12345678'
  health_check_database = 'postgres'

  failover_command = '/etc/pgpool2/failover.sh %d %H %M /tmp/trigger_main.5432'
  
  fail_over_on_backend_error = on
  ```
  
  В файле */etc/pgpool2/pool_hba.conf* на машине **balancer**:
  
  ```
  host    all         all         192.168.12.0/24       trust
  ```
  
### 2. Настройка pgpool2 на остальных машинах <---- вроде не нужно это настраивать, т.к. pgpool проверяет состояния самой СУБД postgresql на порту 5432

На машине **master** поправить */etc/pgpool2/pgpool.conf*:

  ```
  listen_addresses = 'localhost, 192.168.12.6'
  
  log_destination = 'syslog'
  ```
  
### 3. Настройка аутентификации в pgpool communication manager (pcp)

Для доступа к менеджеру ресурсов pcp необходимо настроить аутентификацию пользователя, от имени которого будет вызывать утилита. Это пользователь необязательно должен быть пользователем в СУБД. 
Файл /etc/pgpool2/pool_passwd содержит информацию о хэше паролей соответсвующих пользователей и именно по этому файлу происходит аутентификация пользователя при вызове pcp.

Создание записи о хэше пароля заданного пользователя (root):

```
$ sudo pg_md5 -U root -p --md5auth
```

После этого потребуется ввести пароль, задаваемый указанному пользователю. 

Новая запись появится в конце файла /etc/pgpool2/pool_passwd:

(рисунок)

Нужно также добавить пароль в конец файла /etc/pgpool2/pcp.conf.

Для этого сначала получить его в форме md5:

```
$ sudo pg_md5 -p 
```

И результат вывода добавить в /etc/pgpool2/pcp.conf вместе с именем пользователя в формает USER:MD5PASSWORD:

(рисунок)

Теперь можем получать информации о состоянии узлов (бэкенды, указанные в конфиге pgpool2 с соответсвующими id).

```
$ pcp_node_info -h 192.168.12.29 -n 0 -v
```

(рисунок)

```
$ pcp_node_info -h 192.168.12.29 -n 1 -v
```

(рисунок)

Расшифровка статусов:

```
0 - this state is only used during initialization. PCP will never display it.
1 - Node is up. No connections yet.
2 - Node is up. Connections are pooled.
3 - Node is down.
```

### 4. Failover

Создать скрипт */etc/pgpool2/failover.sh*Ж

```
#! /bin/sh
# Failover command for streaming replication.
# This script assumes that DB node 1 is primary, and 0 is standby.
#
# If primary goes down, create a
# trigger file so that standby takes over primary node.
#
# Arguments: $1: failed node id. $2: new primary node hostname. $3: path to
# trigger file.

failed_node=$1
new_primary=$2
trigger_file=$3

KEY_PATH="/var/lib/postgresql/.ssh/id_rsa"

if [ $failed_node = 1 ]; then
     if [ $UID -eq 0 ]
     then
         sudo -l postgres ssh -T -i $KEY_PATH postgres@$new_primary "touch $trigger_file"
        exit 0;
     fi
     ssh -T -i $KEY_PATH postgres@$new_primary "touch $trigger_file"
fi;

echo "Сервер $failed_node упал."
echo "Ведущий сервер - $new_primary."

exit 0;
```
