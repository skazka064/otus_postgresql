
# Тестирование

### etcd
```
etcdctl member list
etcdctl cluster-health
etcdctl ls /service/patroni
etcdctl get /service/patroni/members/pgsql2
```

```bash
root@etcd1:/home/yc-user# etcdctl ls /service/patroni
/service/patroni/leader
/service/patroni/status
/service/patroni/history
/service/patroni/members
/service/patroni/initialize
/service/patroni/config
```





### bouncer
```
systemctl restart pgbouncer
systemctl status pgbouncer.service
psql -p 6432 pgbouncer
SHOW STATS;
SHOW TOTALS;
SHOW DATABASES;
```


### patroni
```
patronictl -c /etc/patroni/patroni.yml list 
patronictl -c /etc/patroni/patroni.yml edit-config

-- Рестарт одной ноды:
patronictl -c /etc/patroni/patroni.yml restart patroni pgsql2

patronictl -c /etc/patroni/patroni.yml pause patroni
patronictl -c /etc/patroni.yml resume patroni

-- Выключение одной ноды:
sudo systemctl stop patroni 

-- Рестарт всего кластера:
patronictl -c /etc/patroni.yml restart patroni

-- Рестарт reload кластера:
patronictl -c /etc/patroni/patroni.yml reload patroni

-- Плановое переключение:
patronictl -c /etc/patroni/patroni.yml switchover patroni

-- Реинициализации ноды:
patronictl -c /etc/patroni/patroni.yml reinit patroni pgsql2
```
