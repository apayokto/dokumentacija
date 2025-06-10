## Docker
Sve sto treba za docker:
[Docker cheat sheet](https://it-cheat-sheets-21aa0a.gitlab.io/docker-cheat-sheet.html)


## SQL 
Veoma dobar [cheat sheet](https://it-cheat-sheets-21aa0a.gitlab.io/sql-cheat-sheet.html)

Knjiga za procitati [High perfomance](https://learning.oreilly.com/library/viwe/high-perfomance-mysql/9781492080503/)

## Replication in MySQL [tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql)

![Kako radi replikacija Master Slave](./images/1_Qei-oIdsrTspUAcu8QoUIA.png)

1. Pravimo [Docker Compose YAML](./images/MasterSlaveyaml.pngMasterSlaveyaml.png)  file(tu sam skontao da je dobra praksa da ime servisa i kontenjera bude isto)
2. Pravimo konfiguracijske fileove [master.conf](./images/masterconf.png) [slave.conf](./images/slaveconf.png)
3. Pokrenemo pomocu docker-composea oba kontenjera
4. Udjemo u master kontenjer i u mysql te kreiramo usera za replikaciju:

        CREATE USER 'replica_user'@'%' IDENTIFIED WITH mysql_native_password BY '123';

        GRANT REPLICATION SLAVE ON *.* TO        'replica_user'@'%';

        FLUSH PRIVILEGES;

5. Sada nadjemo zadnji unos u binary logu, i odatle ce poceti replikacija, ako je baza ziva radimo ove korake, sve to i dalje na masteru:

        FLUSH TABLES WITH READ LOCK;

        SHOW MASTER STATUS;

6. Vidimo koji je zadnji unos u binlogu i zapisemo.

7. Spojimo se na slave kontenjer i tamo uradimo:



        CHANGE REPLICATION SOURCE TO

        SOURCE_HOST='master',

        SOURCE_USER='replica_user',

        SOURCE_PASSWORD='123',

        SOURCE_LOG_FILE='mysql-bin.000001',

        SOURCE_LOG_POS=899;

        SOURCE_DELAY = 3600;


7.1  

         START REPLICA;

         SHOW REPLICA STATUS\G;

8.  Ako neki binlog zapis pravi problem mozemo ga pogurati sa:

        SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; 

9. Za zaustavljane replikacije koristimo

        STOP REPLICA;



