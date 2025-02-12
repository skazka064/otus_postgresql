# Начало
## Установка Postgresql
### Создал ВМ 
### В ней установил докер
### Создал докер контейнер с postgresql с монтированием в созданную директорию на хосте, с указанием созданной подсети, с пробросом порта 5432 на хост
```bash
apt-get update
apt-get install docker-engine
systemctl start docker
systemctl enable docker --now
docker network create pg-net
docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgresql:/var/lib/postgresql/data postgres:15  
8a61c70d05f39850ed14d3406011a01460934bc1229ea3257d012d145798fd1b
```
### Зашел в postgresql под клиентом, запущенным в докере

```bash
docker run -it --rm --name pg-client --network pg-net -v /var/lib/postgresql:/var/lib/postgresql/data postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.10 (Debian 15.10-1.pgdg120+1))
Type "help" for help.
```
```sql
postgres=# \l


                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(3 rows)
```

### Убедился что таблиц нет

```sql
postgres=# \dt
Did not find any relations.
postgres=#
```


### Создал тестовую таблицу и залил одну строчку

```sql
postgres=#  create table t (s text);
CREATE TABLE
postgres=# insert into t values ('Hello World!');
INSERT 0 1
```

### Затем, исправил pg_hba.conf и postgesql.conf, что бы подключиться извне. Перезапустил postgres.

```bash
host    all             all             0.0.0.0/0            trust
listen_addresses = '*'

```

### Посмотрел работу докера
```bash
root@user-VirtualBox:~# docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS             PORTS                                       NAMES
b14db7fb1ac0   postgres:15   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```
### Подключился с ВМ к докеру с postgresql 
```sql
postgres@user-VirtualBox:~$ psql -h 172.29.2.215
Password for user postgres:
psql (12.20 (Ubuntu 12.20-0ubuntu0.20.04.1), server 15.10 (Debian 15.10-1.pgdg120+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

```

### *Убедился, что все работает*

### Затем, подключился к контейнеру с postgresql извне 
### И запустил генерацию данных с другого сервера, где преварительно был установлен генератор данных.

```javascript
// Подключение к PostgreSQL
const sql = postgres('postgres://username:password@host:5432/database', {
  host                 : '172.29.2.215',            // 
  port                 : 5432,                    // Postgres server port[s]
  database             : 'postgres',             // Name of database to connect to
  username             : 'postgres',            // Username of database user
  password             : 'postgres',           // Password of database user
 
})
await sql `CREATE TABLE IF NOT EXISTS  users (userid text, username text, email text, avatar text, password text, birthdate text, registeredAt text);`;
await sql `CREATE TABLE IF NOT EXISTS company (department text, productname text, price text, productadjective text, productmaterial text, product text, productdescription text, isbn text)`
await sql `CREATE TABLE IF NOT EXISTS location (zipcode text, state text, city text, streetaddr text, longitude text)`
await sql `INSERT INTO users  VALUES (${userid[0]},${username[0]},${email[0]},${avatar[0]},${password[0]},${birthdate[0]},${registeredat[0]}),(${userid[1]},${username[1]},${email[1]},${avatar[1]},${password[1]},${birthdate[1]},${registeredat[1]}),(${userid[2]},${username[2]},${email[2]},${avatar[2]},${password[2]},${birthdate[2]},${registeredat[2]}),(${userid[3]},${username[3]},${email[3]},${avatar[3]},${password[3]},${birthdate[3]},${registeredat[3]}),(${userid[4]},${username[4]},${email[4]},${avatar[4]},${password[4]},${birthdate[4]},${registeredat[4]}),(${userid[5]},${username[5]},${email[5]},${avatar[5]},${password[5]},${birthdate[5]},${registeredat[5]}),(${userid[6]},${username[6]},${email[6]},${avatar[6]},${password[6]},${birthdate[6]},${registeredat[6]}),(${userid[7]},${username[7]},${email[7]},${avatar[7]},${password[7]},${birthdate[7]},${registeredat[7]}),(${userid[8]},${username[8]},${email[8]},${avatar[8]},${password[8]},${birthdate[8]},${registeredat[8]}),(${userid[9]},${username[9]},${email[9]},${avatar[9]},${password[9]},${birthdate[9]},${registeredat[9]})`
await sql `INSERT INTO location ("zipcode","state","city","streetaddr","longitude") VALUES (${zipcode[0]},${state[0]},${city[0]},${streetaddr[0]},${longitude[0]}),(${zipcode[1]},${state[1]},${city[1]},${streetaddr[1]},${longitude[1]}),(${zipcode[3]},${state[3]},${city[3]},${streetaddr[3]},${longitude[3]}),(${zipcode[4]},${state[4]},${city[4]},${streetaddr[4]},${longitude[4]}),(${zipcode[2]},${state[2]},${city[2]},${streetaddr[2]},${longitude[2]}),(${zipcode[6]},${state[6]},${city[6]},${streetaddr[6]},${longitude[6]}),(${zipcode[7]},${state[7]},${city[7]},${streetaddr[7]},${longitude[7]}),(${zipcode[8]},${state[8]},${city[8]},${streetaddr[8]},${longitude[8]}),(${zipcode[9]},${state[9]},${city[9]},${streetaddr[9]},${longitude[9]}),(${zipcode[5]},${state[5]},${city[5]},${streetaddr[5]},${longitude[5]})`;
await sql `INSERT INTO company ("department","productname","price","productadjective","productmaterial","product","productdescription","isbn") VALUES (${department[0]},${productname[0]},${price[0]},${productadjective[0]},${productmaterial[0]},${product[0]},${productdescription[0]},${isbn[0]}),(${department[1]},${productname[1]},${price[1]},${productadjective[1]},${productmaterial[1]},${product[1]},${productdescription[1]},${isbn[1]}),(${department[2]},${productname[2]},${price[2]},${productadjective[2]},${productmaterial[2]},${product[2]},${productdescription[2]},${isbn[2]}),(${department[3]},${productname[3]},${price[3]},${productadjective[3]},${productmaterial[3]},${product[3]},${productdescription[3]},${isbn[3]}),(${department[4]},${productname[4]},${price[4]},${productadjective[4]},${productmaterial[4]},${product[4]},${productdescription[4]},${isbn[4]}),(${department[5]},${productname[5]},${price[5]},${productadjective[5]},${productmaterial[5]},${product[5]},${productdescription[5]},${isbn[5]}),(${department[6]},${productname[6]},${price[6]},${productadjective[6]},${productmaterial[6]},${product[6]},${productdescription[6]},${isbn[6]}),(${department[7]},${productname[7]},${price[7]},${productadjective[7]},${productmaterial[7]},${product[7]},${productdescription[7]},${isbn[7]}),(${department[8]},${productname[8]},${price[8]},${productadjective[8]},${productmaterial[8]},${product[8]},${productdescription[8]},${isbn[8]}),(${department[9]},${productname[9]},${price[9]},${productadjective[9]},${productmaterial[9]},${product[9]},${productdescription[9]},${isbn[9]})`
```


### Зашел в postgresql из контейнера с клиентом
```bash
 docker run -it --rm --name pg-client --network pg-net -v /var/lib/postgresql:/var/lib/postgresql/data postgres:15 psql -h pg-server -U postgres
```

### Посмотрел размер таблиц и БД
```sql
postgres=# \dt+
                                    List of relations
 Schema |   Name   | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+----------+-------+----------+-------------+---------------+-------+-------------
 public | company  | table | postgres | permanent   | heap          | 60 MB |
 public | location | table | postgres | permanent   | heap          | 24 MB |
 public | t        | table | postgres | permanent   | heap          | 16 kB |
 public | users    | table | postgres | permanent   | heap          | 67 MB |

postgres=# select * from location limit 5 ;
  zipcode   |    state     |       city       |        streetaddr         | longitude
------------+--------------+------------------+---------------------------+-----------
 48483-1209 | Vermont      | North Faustino   | 6537 Prospect Avenue      | -92.0066
 18867-8673 | Louisiana    | South Justice    | 578 S 7th Street          | -98.2856
 78701      | Utah         | McClureboro      | 1863 MacGyver Lake        | 3.7909
 13625-6808 | North Dakota | East Spencerberg | 99315 The Beeches         | -163.7633
 69099-4653 | Connecticut  | Bednarworth      | 72389 Destinee Extensions | -140.4979
(5 rows)

postgres= select * from t;
      s
--------------
 Hello World!
(1 row)
postgres=#
```

# Затем я удалил контейнер с postgresql
```bash
docker stop pg-server
docker rm pg-server
```

### Заново развернул контейнер примонтировав директорию,с которой изначально создавал контейнер т.к. она сохранилась на хосте и в ней остались(ли) данные
```bash
docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgresql:/var/lib/postgresql/data postgres:15
```


### По ip удалось зайти в postgresql.
```sql
$ psql -h 172.29.2.215
psql (12.22 (Ubuntu 12.22-0ubuntu0.20.04.1), server 15.10 (Debian 15.10-1.pgdg120+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
Type "help" for help.
```

### Затем проверил все ли месте
```sql
postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

postgres=# \dt
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | company  | table | postgres
 public | location | table | postgres
 public | t        | table | postgres
 public | users    | table | postgres
(4 rows)

postgres=# \dt+
                     List of relations
 Schema |   Name   | Type  |  Owner   | Size  | Description
--------+----------+-------+----------+-------+-------------
 public | company  | table | postgres | 60 MB |
 public | location | table | postgres | 24 MB |
 public | t        | table | postgres | 16 kB |
 public | users    | table | postgres | 67 MB |
(4 rows)

postgres=#

postgres= select * from t;
      s
--------------
 Hello World!
(1 row)

postgres=# select * from location limit 5;
  zipcode   |    state     |       city       |        streetaddr         | longitude
------------+--------------+------------------+---------------------------+-----------
 48483-1209 | Vermont      | North Faustino   | 6537 Prospect Avenue      | -92.0066
 18867-8673 | Louisiana    | South Justice    | 578 S 7th Street          | -98.2856
 78701      | Utah         | McClureboro      | 1863 MacGyver Lake        | 3.7909
 13625-6808 | North Dakota | East Spencerberg | 99315 The Beeches         | -163.7633
 69099-4653 | Connecticut  | Bednarworth      | 72389 Destinee Extensions | -140.4979
(5 rows)
```

# ДАННЫЕ ОСТАЛИСЬ НА МЕСТЕ

