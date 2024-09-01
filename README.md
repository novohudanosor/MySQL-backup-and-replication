# MySQL-backup-and-replication
MySQL: бэкап и репликация
## Что нужно сделать?
* В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp
* Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:
```
| bookmaker |
| competition |
| market |
| odds |
| outcome |
```
* Настроить GTID репликацию

## Решение
1. Командой ``` vagrant up ``` поднимаем виртуальные машины.
2. Устанавливем mysql-server-8.0. на обоих серверах:
```
apt update
apt install mysql-server
```
* Копируем файлы конфигурации mysql
```
root@master:~# cp /vagrant/provisioning/files/conf.d/mysqld_master.cnf /etc/mysql/mysql.conf.d/mysqld.cnf

root@slave:~# cp /vagrant/provisioning/files/conf.d/mysqld_slave.cnf /etc/mysql/mysql.conf.d/mysqld.cnf
```
* Далее нужно перезапустить службу mysql
```
systemctl restart mysql
```
* **Устанавливаем пароль root в mysql**
```
mysql -e \"ALTER USER 'root'@'localhost' IDENTIFIED WITH 'caching_sha2_password' BY 'Otus2024!@#'\"
```
* **Следует обратить внимание**, что атрибут server-id на мастер-сервере должен обязательно отличаться от server-id слейв-сервера. Проверяем:
 ```
mysql> select @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)

mysql> select @@server_id;
+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
1 row in set (0.00 sec)
```
* Создаем базу данных, и загружаем в нее имеющийся дамп:
```
mysql> CREATE DATABASE bet;
Query OK, 1 row affected (0.00 sec)

root@master:~# mysql -uroot -p -D bet < /vagrant/bet.dmp

mysql> use bet;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |   
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
7 rows in set (0.00 sec)
```
* Далее создадим пользователя для репликации и дадим ему права на эту самую репликацию:
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2024';
mysql> SELECT user,host FROM mysql.user where user='repl';
+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0.00 sec)
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2024';
```
* Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию:
```
root@master:~# mysqldump --all-databases --triggers --routines --master-data
--ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```
* На этом настройка Mastera завершена. Файл дампа нужно залить на слейв. После этого проверяем корректность переноса БД:
```
mysql> SOURCE /tmp/master.sql
mysql> SHOW DATABASES LIKE 'bet';
+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
1 row in set (0.07 sec)

mysql> use bet;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> SHOW TABLES;
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0.01 sec)
```
* Раскомментируем в /etc/my.cnf.d/05-binlog.cnf строки:
```
#replicate-ignore-table=bet.events_on_demand
#replicate-ignore-table=bet.v_same_event
```
* Таким образом указываем таблицы которые будут игнорироваться при репликации
```
* В выводе нет таблиц v_same_event и events_on_demand на слэйв сервере. Далее пробуем подключить и запустить репликацию:
```
mysql> CHANGE REPLICATION SOURCE TO SOURCE_HOST = "192.168.11.150", SOURCE_PORT = 3306, SOURCE_USER = "repl", SOURCE_PASSWORD =  "!OtusLinux2024", SOURCE_AUTO_POSITION = 1;
mysql> START REPLICA;
mysql> SHOW REPLICA STATUS;
*************************** 1. row ***************************
          Replica_IO_State: Waiting for source to send event
               Source_Host: 192.168.56.150
               Source_User: repl
               Source_Port: 3306
             Connect_Retry: 60
           Source_Log_File: binlog.000004
       Read_Source_Log_Pos: 93619
            Relay_Log_File: slave-relay-bin.000003
             Relay_Log_Pos: 93829
     Relay_Source_Log_File: binlog.000004
        Replica_IO_Running: Yes
       Replica_SQL_Running: Yes
           Replicate_Do_DB: 
       Replicate_Ignore_DB: 
        Replicate_Do_Table: 
    Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      ....
    Retrieved_Gtid_Set: 2bc7b2cb-62dd-11ef-93b2-0255be2b59e4:1-38
         Executed_Gtid_Set: 18bbce3a-62dd-11ef-8fee-0255be2b59e4:1-174,2bc7b2cb-62dd-11ef-93b2-0255be2b59e4:1-38
             Auto_Position: 1
```
* На мастер сервере в базе 'bet' создаётся запись '1xbet' в таблице 'bookmaker'. Данные изменения происходят и на слэйв сервере:
* Master сервер
```
mysql> USE bet;
Database changed
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
Query OK, 1 row affected (0.05 sec)

mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.00 sec)
```
* Slave сервер
```
mysql> USE bet;
Database changed
mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |   
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.00 sec)   
```
