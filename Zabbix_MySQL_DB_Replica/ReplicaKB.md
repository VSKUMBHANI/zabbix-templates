# MySQL Database Replication on AlmaLinux 9 for Database `zabbix`

## Master: `192.168.1.10`

### 1. Configure MySQL on Master

Edit the file: `/etc/my.cnf.d/mysql-server.cnf`

```ini
[mysqld]
server-id=1
log_bin=mysql-bin
binlog_do_db=zabbix
```

### 2. Create Replication User

Login to MySQL:

```sql
CREATE USER 'replica'@'192.168.1.20' IDENTIFIED BY 'YourSecurePassword';
```

> 🔒 **Note:** By default, the user uses `caching_sha2_password`. If the replica fails with an authentication error, switch to `mysql_native_password`:

```sql
ALTER USER 'replica'@'192.168.1.20' IDENTIFIED WITH mysql_native_password BY 'YourSecurePassword';
```

### 3. Grant Replication Privileges

```sql
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'192.168.1.20';
FLUSH PRIVILEGES;
```

### 4. Verify Authentication Plugin (Optional)

```sql
SELECT user, host, plugin FROM mysql.user WHERE user = 'replica';
```

### 5. Lock Tables and Note Log File Position

```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```
> ✍️ **Record the output values:**
>
> - **File:** `1.003054`  
> - **Position:** `3743896`

### 6. Login to another ssh session of Master and dump the 'zabbix' database.
```sql
mysqldump -u root -p zabbix > zabbix.sql
```
### 7. Copy dump file to Slave using scp command.
```sql
scp zabbix.sql root@192.168.170.103:/root/
```
### 8. Return back to first session and unlock the table
```sql
 UNLOCK TABLES;
```
---

## Slave: `192.168.1.20`

### 1. Configure MySQL on Slave

Edit the file: `/etc/my.cnf.d/mysql-server.cnf`

```ini
[mysqld]
server-id=2
relay_log=relay-log
```
### 2. Create 'zabbix' database in Slave

```sql
mysql -u root -p -e "CREATE DATABASE zabbix;"
```
### 3. Dump data from zabbix.sql to 'zabbix' database in Slave

```sql
mysql -u root -p zabbix < /root/zabbix.sql;
```
### 4. Create Replica User (Optional if already created on Master)

```sql
CREATE USER 'replica'@'192.168.1.20' IDENTIFIED WITH mysql_native_password BY 'YourSecurePassword';
```

### 5. Configure the Replication

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.1.10',
  MASTER_USER='replica',
  MASTER_PASSWORD='YourSecurePassword',
  MASTER_LOG_FILE='1.003054',
  MASTER_LOG_POS=3743896;
```

### 6. Start Replication

```sql
START SLAVE;
```

### 7. Verify Replication Status

```sql
SHOW SLAVE STATUS\G
```

> ✅ **Expected output:**
>
> - `Slave_IO_Running: Yes`  
> - `Slave_SQL_Running: Yes`  
>
> If both values are `Yes`, replication is working correctly.

---

### 🔁 Final Notes

- Always use a **strong password** for the replication user.
- Ensure **firewall rules** and **SELinux settings** permit MySQL traffic between the master and slave.
- To stop replication during maintenance, use:
  
  ```sql
  STOP SLAVE;
  ```

- To resume:
  
  ```sql
  START SLAVE;
  ```
