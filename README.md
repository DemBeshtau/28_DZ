# Репликация postgres
1. Настроить hot-standby репликацию с использованием слотов;
2. Настроить правильное резервное копирование.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.18, Ansible 9.4.0, образа ubuntu/jammy64 версии 20240301.0.0.<br/>
### Ход решения ###
1. Установка и настройка postgresql на хостах node1 и node2:
   - Установка необходимого ПО postgresql на обоих хостах:
   ```shell
   apt install postgresql postgresql-contrib
   systemctl enable --now postgresql
   ```
   - Создание пользователя для репликации на хосте node1:
   ```shell
   root@node1:~# sudo -u postgres psql
   could not change directory to "/root": Permission denied
   psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
   Type "help" for help.

   postgres=# CREATE USER replicator WITH REPLICATION Encrypted PASSWORD 'Otus2024!';
   ...
   ```
   - Подготовка на обоих хостах конфигурационного файла /etc/postgresql/14/main/postgresql.conf:
   ```shell
   data_directory = '/var/lib/postgresql/14/main'          # use data in another directory
   hba_file = '/etc/postgresql/14/main/pg_hba.conf'        # host-based authentication file
   ident_file = '/etc/postgresql/14/main/pg_ident.conf'    # ident configuration file
   external_pid_file = '/var/run/postgresql/14-main.pid'   # write an extra PID file
   #Указываем ip-адреса, на которых postgres будет слушать трафик на порту 5432 (параметр port)
   listen_addresses = 'localhost, 192.168.56.11'
   #Указываем порт порт postgres
   port = 5432
   #Устанавливаем максимально 100 одновременных подключений
   max_connections = 100
   log_directory = 'log'
   log_filename = 'postgresql-%a.log'
   log_rotation_age = 1d
   log_rotation_size = 0
   log_truncate_on_rotation = on
   max_wal_size = 1GB
   min_wal_size = 80MB
   log_line_prefix = '%m [%p] '
   #Указываем часовой пояс для Москвы
   log_timezone = 'UTC+3'
   timezone = 'UTC+3'
   datestyle = 'iso, mdy'
   lc_messages = 'en_US.UTF-8'
   lc_monetary = 'en_US.UTF-8'
   lc_numeric = 'en_US.UTF-8'
   lc_time = 'en_US.UTF-8'
   default_text_search_config = 'pg_catalog.english'
   #можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления;
   hot_standby = on
   #Включаем репликацию
   wal_level = replica
   #Количество планируемых слейвов
   max_wal_senders = 3
   #Максимальное количество слотов репликации
   max_replication_slots = 3
   #будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.
   hot_standby_feedback = on
   #Включаем использование зашифрованных паролей
   password_encryption = scram-sha-256
   include_dir = 'conf.d'
   ```
   - Подготовка на обоих хостах конфигурационного файла /etc/postgresql/14/main/pg_hba.conf:
   ```shell
   # Database administrative login by Unix domain socket
   local   all             postgres                                peer
   # TYPE  DATABASE        USER            ADDRESS                 METHOD
   # "local" is for Unix domain socket connections only
   local   all             all                                     peer
   # IPv4 local connections:
   host    all             all             127.0.0.1/32            scram-sha-256
   # IPv6 local connections:
   host    all             all             ::1/128                 scram-sha-256
   # Allow replication connections from localhost, by a user with the
   # replication privilege.
   local   replication     all                                     peer
   host    replication     all             127.0.0.1/32            scram-sha-256
   host    replication     all             ::1/128                 scram-sha-256
   host    replication     replicator      192.168.56.11/32        scram-sha-256
   host    replication     replicator      192.168.56.12/32        scram-sha-256
   host    all             barman          192.168.56.13/32        scram-sha-256
   host    replication     barman          192.168.56.13/32        scram-sha-256
   ```
   - Перезапуск сервиса postgresql на обоих хостах:
   ```shell
   systemctl restart postgresql
   ```
   - Удаление на хосте node2 содержимого /var/lib/postgresql/14/main/ и копирование по этому пути данных с хоста node1 с использованием утилиты pg_basebackup (настройка репликации):
   ```shell
   root@node2:~ systemctl stop postgresql
   root@node2:~ rm -rf /var/lib/postgresql/14/main/*
   root@node2:~ pg_basebackup -h 192.168.56.11 -U  replicator -D  /var/lib/postgresql/14/main/ -R -P
   root@node2:~ systemctl start postgresql
   ```
   - Для проверки работы репликации, на хосте node1 создаётся таблица, которая, в случае правильной настройки, появляется на хосте node2:
   ```shell
   (node1)
   postgres=# CREATE DATABASE otus;
   CREATE DATABASE
   postgres=# \l
                                 List of databases
      Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
   -----------+----------+----------+---------+---------+-----------------------
    otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
    postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
    template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
    template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
   (4 rows)

   (node2)
   postgres=# \l
                                 List of databases
      Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges         
   -----------+----------+----------+---------+---------+-----------------------
    otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
    postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
    template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
    template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
   (4 rows)

   ```

