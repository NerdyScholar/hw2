# Homework 6 (Физический уровень PostgreSQL)


Поднимаем ВМ и ставим на нее PostgreSQL, используя информаицю из уроков 3 и 6.
Проверим:
```
 yc-user@otus-vm:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

Зайдем  из под пользователя postgres в psql и сделаем произвольную таблицу с произвольным содержимым

```
  postgres=# create table test(c1 text);
CREATE TABLE
postgres=# INSERT INTO test (c1)
postgres-# VALUES
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text)),
postgres-#   (md5(random()::text));
INSERT 0 10
postgres=# 
postgres=# select * from test
postgres-# ;
                c1                
----------------------------------
 7ae00286ca78c7c7e4a3a0d32b8b707a
 08136bf3d0c02c1f5c255225367cdf56
 ab4e4043ae0553d85c48614a00a9b18f
 e325e2170c5ff3a98638ea2b8b503ba9
 da287f1bc39515decf96f3e9e8e03ccb
 d6e8b047cad1e23de561c5e2d8a9c305
 c862bb87a6acd25c20220832b8e048d3
 0a486a56700360319ada606f94319600
 fbd39845343a0cd505b1f15d027eb065
 07e8ab60af9c285980c712cabf492af3
(10 rows)

```


Остановим PostgreSQL:

```
  yc-user@otus-vm:~$ sudo -u postgres pg_ctlcluster 14 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@14-main
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

Создадим новый диск, разметим и примонтируем его к /mnt/data/:



```
  syc compute disk create \
    --name new-disk \
    --type network-hdd \
    --size 10 \
    --description "second disk for otus-vm"
```


```
 yc-user@otus-vm:~$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE SIZE MOUNTPOINT LABEL
vda            15G            
├─vda1          1M            
└─vda2 ext4    15G /          
vdb            10G            
```


```
  yc-user@otus-vm:~$ sudo mount /dev/vdb1 /mnt/data
```



```
 yc-user@otus-vm:~$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE SIZE MOUNTPOINT LABEL
vda            15G            
├─vda1          1M            
└─vda2 ext4    15G /          
vdb    ext4    10G /mnt/data 
```

После рестарта ВМ монтирование не сохранилось, поэтому необходимо внести корректировки через fstab:


```
 yc-user@otus-vm:~$ sudo nano /etc/fstab
```
Внесем такую строчку:

```
UUID=be2c7c06-cc2b-4d4b-96c6-e3700932b129 /mnt/data ext4 defaults 0 2

```
Пояснения:
```
UUID=<UUID_диска> <точка_монтирования> <тип_файловой_системы> <опции_монтирования> <флаги_дампа> <флаги_проверки>

```
Перенесем содержимое /var/lib/postgres/15 в /mnt/data:



```
 yc-user@otus-vm:~$ sudo mv /var/lib/postgresql/14/* /mnt/data/
```

Сменим владельца /mnt/data с root на postgres:


```
  yc-user@otus-vm:~$ sudo chown -R postgres:postgres /mnt/data
```
Попытаемся поднять Postgres:
```
  yc-user@otus-vm:~$  sudo -u postgres pg_ctlcluster 14  main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist
```
Не получилось, потому что Postgres обращается к директории, где уже нет файлов для запуска, так как мы их перенесли.

Зайдем в конфигурационный файл:
```
sudo nano /etc/postgresql/14/main/postgresql.conf
```
Сменим data_directory c /var/lib/postgresql/14/main на /mnt/data/main:

```
data_directory = '/mnt/data/main'

```
Попробуем снова запуститься:

```
yc-user@otus-vm:~$ sudo systemctl restart postgresql@14-main
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
14  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-14-main.log


```
Все успешно, проверим данные в таблице:


```
postgres=# select * from test;
                c1                
----------------------------------
 7ae00286ca78c7c7e4a3a0d32b8b707a
 08136bf3d0c02c1f5c255225367cdf56
 ab4e4043ae0553d85c48614a00a9b18f
 e325e2170c5ff3a98638ea2b8b503ba9
 da287f1bc39515decf96f3e9e8e03ccb
 d6e8b047cad1e23de561c5e2d8a9c305
 c862bb87a6acd25c20220832b8e048d3
 0a486a56700360319ada606f94319600
 fbd39845343a0cd505b1f15d027eb065
 07e8ab60af9c285980c712cabf492af3
(10 rows)
```

Все получилось, данные перенеслись успешно.


