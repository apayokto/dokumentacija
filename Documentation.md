# LINUX

Dobar pocetak ako si [extrovert](https://www.youtube.com/watch?v=VbEx7B_PTOE&list=PLIhvC56v63IJIujb5cyE13oLuyORZpdkL)

Dobar pocetak ako si [introvert](https://www.youtube.com/@LearnLinuxTV)

# Docker

Uvijek [sluzbena dokumentacija](https://docs.docker.com/)

Sve sto treba za docker:
[Docker cheat sheet](https://it-cheat-sheets-21aa0a.gitlab.io/docker-cheat-sheet.html)

Napraviti account na [Docker hubu](https://hub.docker.com)


# MySQL

Uvijek [suzbena dokumentacija](https://dev.mysql.com/doc/)

Veoma dobar [cheat sheet](https://it-cheat-sheets-21aa0a.gitlab.io/sql-cheat-sheet.html)

Knjiga za procitati [High perfomance](https://learning.oreilly.com/library/viwe/high-perfomance-mysql/9781492080503/)

Jedon od boljih AI ([Amazing Indian](https://www.youtube.com/@hercules7sakthi/videos)) za ucenje Mysql

Rick RoTs [Rules of Thumb](https://mysql.rjweb.org/doc.php/ricksrots) ilitiga nauceno iz iskustva(drugih ljudi ne mene)


### Vjezba: Replication in MySQL [tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql)

![Kako radi replikacija Master Slave](./images/1_Qei-oIdsrTspUAcu8QoUIA.png)

1. Pravimo [Docker Compose YAML](./images/MasterSlaveyaml.pngMasterSlaveyaml.png)  file(tu sam skontao da je dobra praksa da ime servisa i kontenjera bude isto)
2. Pravimo konfiguracijske fileove [master.conf](./images/masterconf.png) [slave.conf](./images/slaveconf.png)
3. Pokrenemo pomocu docker-composea oba kontenjera
4. Udjemo u master kontenjer i u mysql te kreiramo usera za replikaciju:

        CREATE USER 'replica_user'@'%' IDENTIFIED WITH mysql_native_password BY '123';

        GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';

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

8.  Ako neki binlog zapis se ne applya na slavea i tu zapne replikacija mozemo je preskociti sa dole navedenom komandom:

        STOP REPLICA;

        SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; 

        START REPLICA;

9. Za zaustavljane replikacije koristimo

        STOP REPLICA;


# MONGODB

Uvijek [sluzbena dokumentacija](https://www.mongodb.com/docs/)

Offical [self managed path](https://learn.mongodb.com/learn/learning-path/mongodb-database-admin-self-managed-path) to scratch the surface

Unoffical [path](https://www.youtube.com/watch?v=cLsawKBUdTE&list=PLSmSa8KVdfSu-XFvjdWoN7z9WRoLly4my)


### Vjezba: REPLICA SET S 3 MONGODB NODA 

1. Prvo pisemo [docker-compose.yaml](./images/mongo.png) file
2. Pravimo foldere za conf fileove koje cemo mountati kroz yaml i dodajemo u njih sta nam treba  -->  [mongo1.cnf](./images/mongoconf.png) mongo2.cnf mongo3.cnf (2 i 3 su isto kao 1)
3. Pravimo foldere za datu koju cemo mountati kroz yaml  --> data1_mongo  data2_mongo data3_mongo  (mkdir) // on ce i sam napraiviti foldere al nece imati permisije zato ih ja odmah napravim
4. Pravimo foldere za logove koje cemo mountati kroz yaml --> log1 log2 log3  (mkdir)  
5. Dajemo im permisije za pisanje i vlasnistvo nad data i log folderima da to mongo instanca moze i raditi iz dockera  (chown, chmod) \\ davao sam za chmod 777 da mi sigurno radi al u praksi ne treba tako
6. Posto smo u conf enable security moramo sada napraviti key file i mountati ga premo yaml file u kontenjer za svaki node  /etc/mongodb-keyfile

        openssl rand -base64 756 > ./mongodb-keyfile  \\ ne znam sada gdje ga drzimo ja sam ga ostavio u ovom folderu
        chmod 0400 /etc/mongo-keyfile   \\ moramo mu dati permisije samo za citanje inace ga mongo instanca nece prihvatiti
        chown -R 999:999 /etc/mongo-keyfile

7. Sve dobro provjeriti i vidjetli u yaml file jesu li imena i fileovi dobro mountani \\ posebno zbog cestih typo
8. Pomocu docker-compose dici sve 3 instance(!obavezno jednu po jednu!)
9. Spojiti se na jednu i inicializirati replica set:


        rs.initiate(
        {
        _id: "rs0",  // id replica seta
        version: 1,
        members: [
                 { _id: 0, host : "mongors1:27017" },
                 { _id: 1, host : "mongors2:27017" },
                 { _id: 2, host : "mongors3:27017" }         ] } )


10. Upotrijebiti admin database i dodati usera:

        use admin
        db.createUser( { user: "admin", pwd: "ass", roles: [{ role: "root", db: "admin" }] })

11. Opet se logirati i provjeriti status replica seta:

        rs.status()

12. Ako je sve uredu mozemo odraditi mini vjezbu manualnog mijenjaja primarya,na primarnom nodu upisemo:

        cfg = rs.conf()   // Trenutna konfiguracija replica seta
        cfg.members[2].priority = 2  //stavimo mu veci prioriti tako znamo da ce on biti izabran
        rs.reconfig(cfg)  // primjenjuje novu konfiguraciju
        rs.stepDown(60)  // automatski mu je 60 sekundi ali stavimo svakako

13. Sada ce replica izabrati node s _id 2 da bude primarni posto smo njemu dali veci prioritet.