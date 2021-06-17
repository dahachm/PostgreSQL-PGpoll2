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
    #: ALTER USER postgres WITH PASSWORD '<password>'
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
    ```
    
  (фото статуса настроек файервола)
