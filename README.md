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
* ** Устанавливаем пароль root в mysql **
```
mysql -e \"ALTER USER 'root'@'localhost' IDENTIFIED WITH 'caching_sha2_password' BY 'Otus2024!@#'\"
```
* ** Следует обратить внимание **, что атрибут server-id на мастер-сервере должен обязательно отличаться от server-id слейв-сервера. Проверяем:
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
