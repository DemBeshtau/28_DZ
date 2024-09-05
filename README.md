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
### 1. Настройка hot-standby репликации с использованием слотов
   - Установка необходимого ПО postgresql на хостах node1 и node2:
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
   - Удаление на хосте node2 содержимого /var/lib/postgresql/14/main/ и копирование по этому пути данных с хоста node1 с использованием утилиты pg_basebackup (запуск репликации):
   ```shell
   root@node2:~# systemctl stop postgresql
   root@node2:~# rm -rf /var/lib/postgresql/14/main/*
   root@node2:~# pg_basebackup -h 192.168.56.11 -U  replicator -D  /var/lib/postgresql/14/main/ -R -P
   root@node2:~# systemctl start postgresql
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
### 2. Настройка резервного копирования
   - Установка для резервного копирования утилиты Barman:
   ```shell
   root@barman:~# apt install barman-cli barman postgresql
   ```
   - Генерирование на хостах node1 и barman ssh-ключей для пользователей postgres и barman соответственно с помощью команды ```ssh-keygen -t rsa -b 4096```.
   - Копирование публичных ключей указанных хостов в соответствующие файлы authorized_keys.
   - Создание на node1 в postgresql пользователя barman с соответствующими правами:
   ```shell
   CREATE USER barman WITH REPLICATION Encrypted PASSWORD 'Otus2024!';
   ```
   - Добавление изменений в конфигурационный файл pg_hba.conf хоста node1:
   ```shell
   ...
   host    all                 barman       192.168.56.13/32      scram-sha-256
   host    replication         barman       192.168.56.13/32      scram-sha-256
   ```
   - Подготовка на хосте barman необходимых конфигурационных файлов для подключения к БД:
   ```shell
   barman@barman:~$ cat .pgpass 
   192.168.56.11:5432:*:barman:Otus2024!

   root@barman:~# cat /etc/barman.conf 
   [barman]
   barman_home = /var/lib/barman
   configuration_files_directory = /etc/barman.d
   barman_user = barman
   log_file = /var/log/barman/barman.log
   compression = gzip
   backup_method = rsync
   archiver = on
   retention_policy = REDUNDANCY 3
   immediate_checkpoint = true
   last_backup_maximum_age = 4 DAYS
   minimum_redundancy = 1

   root@barman:~# cat /etc/barman.d/node1.conf 
   [node1]
   description = "backup node1"
   active = on
   ssh_command = ssh postgres@192.168.56.11
   conninfo = host=192.168.56.11 user=barman port=5432 dbname=postgres
   retention_policy_mode = auto
   retention_policy = RECOVERY WINDOW OF 7 days
   wal_retention_policy = main
   streaming_archiver=on
   path_prefix = /usr/pgsql-14/bin/
   create_slot = auto
   slot_name = node1
   streaming_conninfo = host=192.168.56.11 user=barman 
   backup_method = postgres
   archiver = off
   ```
   - Запуск резервного копирования на хосте barman:
   ```shell
   barman switch-wal node1
   barman cron
   barman backup node1
   ```
   - Проверка работоспособности конфигурации:
   ```shell
   (node1)
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
   postgres=# DROP DATABASE otus;
   DROP DATABASE
   postgres=# \l
                                 List of databases
      Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
   -----------+----------+----------+---------+---------+-----------------------
    postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
    template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
    template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
   (3 rows)

   (barman)
   root@barman:~# barman list-backup node1
   node1 20240905T152330 - Thu Sep  5 12:23:31 2024 - Size: 33.6 MiB - WAL Size: 0 B - WAITING_FOR_WALS
   
   root@barman:~# barman recover node1 20240905T152330 /var/lib/postgresql/14/main --remote-ssh-command "ssh postgres@192.168.56.11"
   The authenticity of host '192.168.56.11 (192.168.56.11)' can't be established.
   ED25519 key fingerprint is SHA256:YTm5V7Ir4kBME/Zvij3DSN9uCYcN8KRwhoKmdYCCGIw.
   This key is not known by any other names
   Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
   Starting remote restore for server node1 using backup 20240905T152330
   Destination directory: /var/lib/postgresql/14/main
   Remote command: ssh postgres@192.168.56.11
   Using safe horizon time for smart rsync copy: 2024-09-04 16:15:27.297790-03:00
   Copying the base backup.
   Copying required WAL segments.
   Generating archive status files
   Identify dangerous settings in destination directory.

   WARNING
   The following configuration files have not been saved during backup, hence they have not been restored.
   You need to manually restore them in order to start the recovered PostgreSQL instance:

       postgresql.conf
       pg_hba.conf
       pg_ident.conf

   Recovery completed (start time: 2024-09-05 15:28:10.869232, elapsed time: 10 seconds)

   Your PostgreSQL server has been successfully prepared for recovery!

   (node1)
   root@node1:~# systemctl restart postgresql
   root@node1:~# sudo -u postgres psql
   could not change directory to "/root": Permission denied
   psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
   Type "help" for help.

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
&ensp;&ensp;Для автоматического конфигурирования инфраструктуры с помощью Ansible, подготовлен плейбук playbook.yml.
