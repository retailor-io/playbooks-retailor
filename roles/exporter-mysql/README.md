# mysqld exporter

(https://github.com/prometheus/mysqld_exporter)[https://github.com/prometheus/mysqld_exporter]

## Required Grants

```mysql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
```

