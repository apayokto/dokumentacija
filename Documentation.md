[PERCONA Vjezba01: Dodavanje asinkronog noda](#vjezba01-dodavanje-asinkronog-noda)

[PERCONA Vjezba02: Testiranje baze pomocu sysbencha](#vjezba02-testiranje-baze-pomocu-sysbencha)

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
2. Pravimo konfiguracijske fileove [master.conf](./images/master2.png) [slave.conf](./images/slave2.png)
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

# PERCONA XTRADB


### Vjezba00: Percona Cluster s 3 noda [link](https://docs.percona.com/percona-server/8.0/quickstart-docker-compose.html#insert-operations)

1. Prvo pravimo [docker-compose.yaml](./images/perconayaml1.png) file u folderu DockerPercona, pravimo isto i za node2 i node3

2. Pravimo folder za .cnf fileove i tamo pravimo [node1.cnf](./images/perconacnf1.png). Za druge dvije conf samo mijenjamo da je to node2 ili 3 i cluster adresu upisivamo sva tri noda. (innodb_io_capacity sam istrazivao i nisam  bio siguran da li ga staviti u cnf file, pa cemo se jos konzultovati)

*update(sa binlog formatom row, u binlog pise promjene po redcima a ne po izjavama i zato autoinc lock mode 2 odgovara perfomansama jer ne zakljucava te redke i on ih brze moze pisati)

3. Za svaki node pravimo foldere za datu i dajemo im permisije da ih instanca iz kontenjera moze koristiti, isto pravimo foldere i za logove.

4. Pokrenemo docker compose jedan po jedan node i vidimo da sve radi.


### Vjezba01: Dodavanje asinkronog noda 

1. U [docker-compose.yaml](./images/perconaarep.png) dodajemo servis za asinkronog 

2. Pravimo i njemu config file gdje mozemo izbaciti wsrep opcije a treba svima dodati gtid_mode=ON, enforce_gtid_consistency=ON da bi asinkrona replikacija mogla raditi pa to izgleda ovako [areplica.cnf](/images/perconaarepconf.png)

GRESKA: Pri restoru backupa zbog putanje bin logova, xtrabackup ih je restorao u folder s logovima a vamo su po defaultu bili u mysql folderu pa ih asinkrona replica nije citala, znaci samo pobrisite putanju za bin logove.

3. Slave node mora dobiti pocetnu kopiju podataka od pxc master noda.Tako da radimo backup na jednom nodu i restoramo na areplici

4. Na node3 radimo backup i prepare: 

        /usr/bin/pxc_extra/pxb-8.0/bin/xtrabackup --backup --target-dir=/tmp/new_replication_backup --slave-info -u root --password 

        /usr/bin/pxc_extra/pxb-8.0/bin/xtrabackup --prepare --target-dir=/tmp/new_replication_backup

5. Kopiramo ga na hosta u folder backups:

        sudo docker cp node3:/tmp/new_replication_backup ~/DockerPercona/backups/

6. Dajemo backupu permisije i vlasnistva:

        sudo chown -R 1001:1001 new_replication_backup/


7. Prebacivamo taj prepared backup na kontenjer gdje areplica:


       sudo docker cp ~/DockerPercona/backups/new_replication_backup areplica:/tmp/

8. Uradimo restore na areplici al prvo pobrisemo podatke iz date i logova:

        rm -rf areplica_data/*
        rm -rf logar/*
        

        /usr/bin/pxc_extra/pxb-8.0/bin/xtrabackup --copy-back --target-dir=/tmp/new_replication_backup



9. Odaberemo jedan node koji ce biti master u ovoj replici(to bude node koji ima najmanje posla) i na njemu pravimo usera za asinkronu replicu

        CREATE USER 'replica2'@'%' IDENTIFIED BY 'NovaLozinkaZaRepliku';

        GRANT REPLICATION SLAVE ON *.* TO 'replica2'@'%';

        FLUSH PRIVILEGES;

10. Na areplici kucamo:

        CHANGE REPLICATION SOURCE TO
        MASTER_HOST='node3',
        MASTER_USER='replica2',
        MASTER_PASSWORD='NovaLozinkaZaRepliku',
        MASTER_PORT=3306,
        MASTER_AUTO_POSITION=1,
        SOURCE_DELAY=60;

11.             Slave_IO_Running: Yes
                Slave_SQL_Running: Yes

### Vjezba02: Testiranje baze pomocu sysbencha 
[link](https://www.liberoscarcelli.com/blog/benchmarking-mysql-databases-with-sysbench)

1. U [docker-compose.yaml](./images/toolsyaml.png) dodajemo debian kontenjer i odmah mu dajemo komandu da instalira sysbench (mogao sam i napraviti docker sliku sa sysbenchom al docemo i do toga).

2. Konektiramo se na jedan node i napravimo usera preko kojega ce sysbench slati podatke.

                CREATE USER 'sbtest'@'%' IDENTIFIED BY 'mbhND5CPm5yIIv';
                CREATE SCHEMA sbtest;
                GRANT ALL ON sbtest.* TO 'sbtest'@'%';

3. Onda ulazimo u tools kontenjer i dajemo sysbenchu komandu da pripremi sve u bazi 

        sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=node1 --mysql-port=3306 --mysql-user=sbtest --mysql-password=mbhND5CPm5yIIv --mysql-db=sbtest --table_size=100000 --tables=10 --threads=10 prepare

4. Pomocu upita na bazi noda1 mozemo vidjeti koliko je podataka upisano u koju tablicu.

        SELECT     table_name,     table_rows 
        FROM     information_schema.tables 
        WHERE     table_schema = 'sbtest';

Meni je bilo zanimljivo da ne prodju svi podaci a log pokazuje da ne prodju zbog MDL (metadata locks). A sta je to nauciti ces kad procitas knjigu sto je linkana na pocetku.

5. [Ovdje](https://manpages.debian.org/testing/sysbench/sysbench.1.en.html) mozes vidjeti sve opcije za sysbench a mi koristimo:

        sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=node1 --mysql-port=3306 --mysql-user=sbtest --mysql-password=mbhND5CPm5yIIv --mysql-db=sbtest --table_size=100000 --tables=10 --threads=10 --report-interval=2 --time=60 --histogram=on run 

6. Onda mozemo poceti istrazivati galera cluster.

### Galera cluster

[Documentation](https://galeracluster.com/library/documentation/index.html)

 [link](https://severalnines.com/resources/whitepapers/galera-cluster-mysql-tutorial/)

[link](https://www.youtube.com/watch?v=P523ftzjijk)

 
Failure:
- SINGLE NODE FAILURE
- TOTAL CLUSTER CRASH

Monitoring:

-Clustera: wsrep_cluster_state_uuid, wsrep_cluster_size, wsrep_cluster_conf_id, wsrep_cluster_status

-Noda: wsrep_ready, wsrep_connected, wsrep_local_state_comment

-Replikacije: wsrep_local_rcv_queue_avg, wsrep_flow_control_paused,wsrep_cert_deps_distance


## MARIADB

### Vjezba00: Dignuti Mariadb cluster s 3 noda

1. Pravimo [docker-compose.yaml](./images/mariayaml.png), i ovako napravimo za jos dva noda

2. Pravimo im foldere za datu i logove i dajemo permisije da mogu pisati u njih

3. Pravimo folder config i u njemu za svaki node njegov conf file [maria1.cnf](./images/mariaconfig.png), tako i za 2 i 3.

4. Sada pokrenemo node jedan po jedan ali ne radi jer smo zaboravili u docker compose staviti

                command: --wsrep-new-cluster
da mozemo bootstrapati neki node.

### Vjezba01: Dodati maxscale ispred clustera
[link](https://mariadb.com/docs/maxscale/mariadb-maxscale-tutorials/mariadb-maxscale-2501-maxscale-2501-setting-up-mariadb-maxscale) 

1. U [docker-compose.yaml](./images/maxscaleyaml.png) dodajemo servis maxscale

2. U config folderu pravimo [maxscale.cnf](./images/maxscalecnf.png) file za maxscale, tu postavljamo dosta postavki [link](https://github.com/mariadb-corporation/MaxScale/blob/24.02/Documentation/Getting-Started/Configuration-Guide.md#configuration) [link](https://www.youtube.com/watch?v=hJd_diI7lBo)

3. Na svakome nodu napravimo maxscale usera kojeg cemo koristiti i za monitoring i za servise (znaci nismo koristili pravilo least privileges):

        CREATE USER 'maxscale'@'%' IDENTIFIED BY 'ass';
        GRANT SELECT ON mysql.user TO 'maxscale'@'%';
        GRANT SELECT ON mysql.db TO 'maxscale'@'%';
        GRANT SELECT ON mysql.tables_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.columns_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.procs_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.proxies_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.roles_mapping TO 'maxscale'@'%';
        GRANT SHOW DATABASES ON *.* TO 'maxscale'@'%';
        GRANT RELOAD ON *.* TO 'maxscale'@'%';
        GRANT PROCESS ON *.* TO 'maxscale'@'%';
        GRANT SLAVE MONITOR ON *.* TO 'maxscale'@'%';
        GRANT REPLICATION CLIENT ON *.* TO 'maxscale'@'%';
        GRANT REPLICATION SLAVE ON *.* TO 'maxscale'@'%';
        FLUSH PRIVILEGES;


4. Pokrenemo maxscale kontenjer  i sada se na baze mozemo spajati preko njega, ja sam to uradio kroz jos jedan kontenjer i [sysbench](#vjezba02-testiranje-baze-pomocu-sysbencha).

        sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=maxscale --mysql-port=3306 --mysql-user=sbtest --mysql-password=ass --mysql-db=sbtest --table_size=100000 --tables=20 --threads=10 --report-interval=2 --time=60 --histogram=on prepare/run

a za citanje ide port 4010.



### Vjezba02: Full Fizicki backup s mariabackupom i restore

1. Pravimo u [dcoker-compose.yaml](./images/mariabackupyaml.png) kontenjer za backup gdje mu mountamo volume s hosta kojemu cemo raditi backup i takodjer mountamo folder gdje ce se cuvati backupi.

2. Napravimo usera za backup:

        CREATE USER 'mariabackup'@'%'
        IDENTIFIED BY 'mbu_passwd';

        GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT
        ON *.* TO 'mariabackup'@'%';


3. Pokrenemo kontenjer za backup i kucamo(uvijek se radi u prazan folder):

        mariadb-backup --backup --host=maria2 --port=3306 --target-dir=/var/backups/novi --user=mariabackup --password=mbu_passwd

4. Spustamo sva 3 noda i brisemo im datu iz njihovih volumena.

5. U backup kontenjeru radimo prepare:

        mariadb-backup --prepare --target-dir=/var/backups/novi 

6. Pa radimo restore ilitiga copy back:

        mariadb-backup --copy-back --target-dir=/var/backups/novi

7. Sada boostrapamo  node2 jer smo preko njega radili restore i spajamo druga dva koja prihvacaju njegov izvor istine.


### Vjezba03: Incremental backup and restore

1. Incremental backup ne mozemo raditi bez full backupa a imamo ga iz prijasnje vjezbe pa sad unesemo nesto novih podataka da imamo sta backupirati s incremental backupom.

2. Incremental backup opet radimo iz backup kontenjera:

        mariadb-backup --backup --target-dir=/var/backups/noviinc --incremental-basedir=/var/backups/novi --host=maria2 --port=3306 --user=mariabackup --password=mbu_passwd

3. Sada opet spustamo sva 3 noda i brisemo im datu iz volumena.

4. Onda radimo prepare backupa, prvo full pa incremental:

        mariadb-backup --prepare --target-dir=/var/backups/novi 

        mariadb-backup --prepare --target-dir=/var/backups/novi --incremental-dir=/var/backups/noviinc

5. Pa radimo restore(ako je vise incremental backupa, prepare ih jedan po jedan na full i na kraju samo se restora iz fulla):

        mariadb-backup --copy-back --target-dir=/var/backups/novi

6. Dizemo node2 s novim izvorom istine i spajamo druga dva noda.

### Vjezba04: PITR restore

1. Unosili smo podatke u bazu i netko je slucajno pobrisao tablicu, a kasnije su se podaci nastavili unositi, pa cemo pomocu pitr i binlogova pokusati uraditi PITR te tablice.

2. Imamo jedan full backup i incremental backup, pa cemo njih iskoristiti, prvo moramo sacuvati binlogove koji nam trebaju.

3. Iz binloga izvlacimo sto nam treba, u segmentima jer samo zelimo izbjeci kada se desio drop tablice:

        mariadb-binlog  --stop-position=5112 /var/lib/mysql/mysql-bin.000001 > /var/backups/binlog01.sql

        mariadb-binlog  --start-position=5242 /var/lib/mysql/mysql-bin.000001 > /var/backups/binlog02.sql

4. Sada radimo full i incremental backup koje imamo,dizemo nodove pa vracamo binlog:

        mariadb -u root -p < /var/backups/binlog01.sql

        mariadb -u root -p < /var/backups/binlog02.sql

5. I sada bi tu trebala biti i slucajno dropana tablica.

### Vjezba05: Pravljenje skripte za full backup

1. Prvo treba razraditi taktiku za backupa (RPO i RTO). Mi smo odlucili full backup raditi svaki dan jednom u ponoc a incremental backup svaki sat.

2. Pravimo skriptu za full backup, u novom folderu kojega cemo nazvati cronbackup, skriptu nazivamo fullscript.sh

3. U prvom dijelu [skritpe](./images/skriptafull.png) definiramo varijable , stavljamo set -Eeuo a to su pravila za prekidanja skripte ako trap uhvati signale koje smo definirali u nastavku. I kazemo mu da kreira direktorij za logove i pomocu dirname izvuce ime loga iz putanje, i direktorij za backup ako ne postoje.

4. Prva polovina drugog dijela [skrite](./images/skriptafull2.png) odnosi se na definiranje backup_succes=0, tu je mala greska jel u unixu 0 znaci succes a u nasem slucaju neuspjeh. Tu definiramo i cleanup funkciju i sta ona radi u slucaju da trap povuce neki od signala INT TERM ERR sto znaci da se cleanup aktivira i brise ostatke fileova iza backupa i sve to appenda(-a) u log($LOG_FILE) i daje izlaz u terminal(tee).Zavrsavamo je sa exit 1, gdje je 1 razlicito od 0 sto znaci da skripta nije uspjela.

5. U drugoj polovini drugog dijela [skripte](./images/skriptafull3.png) definiramo finish fuknciju, ona se aktivira kad trap uhvati EXIT signal.Prvi if kaze ako je backup_succes jednak 1 onda je backup uspjesno zavrsen,u njoj definiramo putanju za kompresirani file(TAR_PATH) i dajemo backupu unique ime pomocu SOURCE_FOLDER.U podfunkciji ako postoji SOURCE_FOLDER onda ga kompresiramo u TAR_PATH i stavljamo u BACKUP_DIR govoreci mu sa basename da uzme samo ime filea na kraju putanje i tako ga skladisti.U podpodfunkciji mu kazemo, ako je sve to dobro proslo(=0) onda reci Kompresija zavrsena u ostalim slucajevima da javi gresku. U podfunkciji ako nije nasao SOURCE_FOLDER da to javi.U glavnoj funkciji mu jos kazemo da brise kompresirane backup fajlove starije od 3 dana i da brise nekompresirane backup foldere starije od 1 dana.Ako sta u glavnoj funkciji nije uspjelo javi da backup nije uspjesan.Ako je sve dobro proslo javi da je finish funkcija zavrsena.

6. U zadnjem dijelu [skripte](./images/skriptafull4.png) lovimo trap signale koji pokrecu cleanup i finish funkcije. Onda mu kazemo da nam javi da pokrece full backup.Prvo pravi parent direktorij gdje smo mu putanju definirali u varijablama.Onda postavljamo u uvjetu da radi backup i ako je uspjesan definiramo ga sa backup_succes=1 i javlja nam da je uspjesno zavrsen u else slucaju kazemo da je backup_succes=0 neuspjesan i izlazimo sa exit 1.

7. Pravimo [cronjob](./images/skriptacronjob.png) koji ce izvrsavati se svaki dan tocno u ponoc. Izguglao sam da moram dati koji shell da koristi i putanju za komande jer mi bez toga cronjob nije radio.

8. Pravimo [Dockerfile](./images/skriptadockerfile.png)

9. Sve to stavimo u [docker-compose.yaml](./images/skriptayaml.png)

10. Buildamo sliku i pokrenemo kontenjer. On ce automatski raditi fullbackup svaki dan u ponoc.A mozemo testirati pokrecuci skriptu van kontenjera:

        docker exec mariacron /usr/local/bin/fullscript.sh


### Vjezba06: Pravljenje skripte za incremental backup



### Vjezba07: MariaDB galera cluster + asinkroni node(GTID)+ maxscale

1. Pravimo novi folder i u njemu novi [docker-compose.yaml](./images/asmaxyaml.png) koji ce sadrzavati 3 mariadb servisa povezana u galera cluster i jedan asinkroni mariadb servis, te maxscale i jedan kontenjer za sysbench da mozemo 'gurati' podatke.

2. Pravimo konfiguracijske fileove za [nodove](./images/asmaxcnf1.png
) u galeri, [asinkronu](./images/asmaxacnf1.png) i [maxscale.cnf](./images/amaxcnf.png) 


3. Dizemo prvo koji cemo boostrapati u nasem slucaju mariam2.Spojimo se na njega i napravimo maxscale usera, kojem dadnemo sve ove privilegije posto cemo preko njega raditi i monitoring i replikaciju...

        CREATE USER 'maxscale'@'%' IDENTIFIED BY 'rNtOdlAg30SkaF';
        GRANT SELECT ON mysql.user TO 'maxscale'@'%';
        GRANT SELECT ON mysql.db TO 'maxscale'@'%';
        GRANT SELECT ON mysql.tables_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.columns_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.procs_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.proxies_priv TO 'maxscale'@'%';
        GRANT SELECT ON mysql.roles_mapping TO 'maxscale'@'%';
        GRANT SHOW DATABASES ON *.* TO 'maxscale'@'%';
        GRANT RELOAD ON *.* TO 'maxscale'@'%';
        GRANT PROCESS ON *.* TO 'maxscale'@'%';
        GRANT SLAVE MONITOR ON *.* TO 'maxscale'@'%';
        GRANT REPLICATION CLIENT ON *.* TO 'maxscale'@'%';
        GRANT REPLICATION SLAVE ON *.* TO 'maxscale'@'%';
        FLUSH PRIVILEGES;

4. Sada dizemo asinkronu repliku,prebacivamo joj restore koji smo uradili na jednom od nodova s mariabackupom i njoj radimo obicnu master slave GTID replikaciju preko nodea koji ima sve binlogove(ne pitaj kako znam).

5. Onda dizemo maxscale i vidimo da li sve radi za nas galera cluster.

        maxctrl list servers

6. Dizemo i asinkronu repliku i onda joj kazemo da je asinkrona:

        CHANGE MASTER TO 
        MASTER_HOST="mariam2", 
        MASTER_PORT=3306,
        MASTER_USER="repl",
        MASTER_PASSWORD="madmax",
        MASTER_USE_GTID=slave_pos,
        MASTER_SSL=0;

7. Sada je spojimo da ide preko maxscalea i RO listenera:

        CHANGE MASTER TO 
        MASTER_HOST='maxscale', 
        MASTER_PORT=4010, 
        MASTER_USER='maxscale', 
        MASTER_PASSWORD='rNtOdlAg30SkaF',
        MASTER_USE_GTID=slave_pos,
        MASTER_SSL=0;

8. Dignemo kontenjer sa sysbenchom i testiramo da li sve radi:

        sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=maxscale --mysql-port=4306 --mysql-user=sbtest2 --mysql-password=mbhND5CPm5yIIv --mysql-db=sbtest2 --table_size=100000 --tables=10 --threads=10 --report-interval=2 --time=60 --histogram=on run     

9.  Spojimo se na maxscale i kazemo mu

        maxctrl list servers
![Maxscale slika](./images/amaxctrl.png)    


### Vjezba08: Dodavanje TLS certifikata na Vjezbu07

[link](https://mariadb.com/resources/blog/securing-mariadb-server-mariadb-maxscale-connections-tls/)

1. Kreiramo direktorij gdje cemo cuvati certifikate

        mkdir certs

2. Udjemo u taj direktorij i kreiramo certifikate:

        openssl genrsa 2048 > ca-key.pem

        openssl req -new -x509 -nodes -days 3650 -key ca-key.pem -out ca.pem -subj "/CN=MyMariaDBClusterCA/O=MyOrg/C=BA"


        openssl req -newkey rsa:2048 -days 3650 -nodes -keyout server-key.pem -out server-req.pem -subj "/CN=server.mycluster.local/O=MyOrg/C=BA"

        openssl x509 -req -in server-req.pem -days 3650 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem

        openssl req -newkey rsa:2048 -days 3650 -nodes -keyout client-key.pem -out client-req.pem -subj "/CN=client.mycluster.local/O=MyOrg/C=BA"


        openssl x509 -req -in client-req.pem -days 3650 -CA ca.pem -CAkey ca-key.pem -set_serial 02 -out client-cert.pem

        rm *-req.pem  
        
3.  ![slika cert](./images/certdir.png)

    Dobijemo ovih 6 fileova, od kojih ca key je kljuc kojim pravimo ca.pem koji nam sluzi za pecacenje ostalih certifikata, kao npr serverskih i klijentskih.

4. Zbog jednostavnosti vjezbe kopirali smo ih u jos jedan folder i jednome smo folderu dali permisije od mariadb a drugom od maxscalea.

5. Onda smo ih [mountali](./images/certyaml2.png) u kontenjere s opcijom read only, maria u maria , max u max. 

6. Prvo smo ih ubacili u nodove od [galera clustera](./images/certyaml.png) da se enkriptira promet izmedju replika. Galerin interni SSL nam kaze da mozemo staviti server certifikate da ne treba client.

7. Onda serverske certifikate stavljamo na svaki node a klijentske u cnf maxscale ispod svakog definiranog servera. Da pokrijemo komunikaciju izmedju maxscalea i naseg galera clustera.

8. Iduci korak je da serverske certifikate stavljamo na maxscale listenere da pokrijemo veze koje se spajaju na njih. Npr. kao veza asinkrone replikacije i maxscalea koju onda moramo uspostaviti sa:

        CHANGE MASTER TO 
        MASTER_HOST='maxscale', 
        MASTER_PORT=4010, 
        MASTER_USER='maxscale', 
        MASTER_PASSWORD='rNtOdlAg30SkaF',
         MASTER_USE_GTID=slave_pos;
        MASTER_SSL=1,
        MASTER_SSL_CERT='/etc/mysql/certs/client-cert.pem',
        MASTER_SSL_KEY='/etc/mysql/certs/client-key.pem',
        MASTER_SSL_CA='/etc/mysql/certs/ca.pem',
        MASTER_SSL_VERIFY_SERVER_CERT=0;

Zadnja linija nam sluzi da iskljucimo verifikaciju prema serverima posto nismo ukljucili ime servera u CN pri pravljenju certifikata pa ga nece prepoznati.







## MONGODB

Uvijek [sluzbena dokumentacija](https://www.mongodb.com/docs/)

Offical [self managed path](https://learn.mongodb.com/learn/learning-path/mongodb-database-admin-self-managed-path) to scratch the surface

Unoffical [path](https://www.youtube.com/watch?v=cLsawKBUdTE&list=PLSmSa8KVdfSu-XFvjdWoN7z9WRoLly4my)




### Vjezba0: REPLICA SET S 3 MONGODB NODA [link](https://www.mongodb.com/docs/manual/replication/)

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
        cfg.members[2].priority = 2  //stavimo mu veci prioritet tako znamo da ce on biti izabran
        rs.reconfig(cfg)  // primjenjuje novu konfiguraciju
        rs.stepDown(60)  // automatski mu je 60 sekundi ali stavimo svakako

13. Sada ce replica izabrati node s _id 2 da bude primarni posto smo njemu dali veci prioritet.

### Vjezba1: Dodavanje hidden noda u replika set

Hidden nam sluzi da radimo backupe ili analitiku preko njega da ne bi opterecivali ostale nodove.
1. Prvo ga dodamo u [docker-compose.yaml](./images/mongohidden.png)
2. Napravimo i za njega [mongoh.conf](./images/mongoconf.png) koji cemo mountati kroz yaml
3. Napravimo foldere za datu i log fileove koje cemo mountati u yamlu s permisijama --> mongohidden_data  loghidden \\ dati permisije i vlasnistva
4. I njemu moramo mountati key file(procedura opisana u prijasnjoj vjezbi) 
5. Dignemo ga pomocu docker-composea
6. Spojimo se na primary instancu i pridruzimo ga replica setu

        rs.add({
        host: "mongohidden:27017",  // njegova adresa
        hidden: true,               // da je skriven
        priority: 0,                // da ne moze biti izabran za primarnog
        votes: 0,                   // da ne moze glasati
        secondaryDelaySecs: 3600    // da pise podatke s zaostatkom od 1h, ako se slucajno pobrise nesto s primarnog da mozemo brze vratiti podatke 
        })

7. Provjeriti status, njemu bi trebalo pisati hidden: true

### Vjezba2: DODAVANJE ARBITER NODA U REPLICA SET [link](https://www.mongodb.com/docs/manual/tutorial/add-replica-set-arbiter/)

 Arbiter nam sluzi u slucaju parnog broja nodova da bude dodatni glas kojim se moze izabrati primarni (najcesci slucaj je 2 noda + arbiter)

1. Prvo ga dodamo u [docker-compose.yaml](./images/mongoarbiter.png) file
2. Njemu ne treba folder za datu, mozemo napraviti samo za logove --> logarbiter // i dati mu permisije i vlasnistva
3. Napraviti i [mongoarb.conf](./images/mongoarbiterconf.png) file za njega, mozemo komentirati sve vezano za podatke a ostaviti postavke za replikaciju, i mountati ga u yamlu
4. I njemu se mora mountati key file 
5. Dizemo ga pomocu docker-composea
6. Pridruzimo ga replica setu s primarnog noda

        rs.addArb({"mongoarb:27017"})  //on vec u komandi addArb
7. Sada mozemo provjeriti stanje:

        rs.status()

### Vjezba3: BACKUP S MONGODUMP [link](https://www.mongodb.com/docs/database-tools/mongodump/)

1. Pravimo novi kontenjer 'mongobackup' s istom slikom kao i nodovi na kojima ce raditi backup --> [docker-compose.yaml](./images/mongoabackup.png) // nisam nasao sliku s mongo tools samo za alate, lakse mi je bilo staviti sliku cijelog monga
2. Mountamo novi folder gdje cemo cuvati backupe --> backups 
3. U enviromentu dadnemo novog usera za pravljenje backupa
4. Dignemo novi kontenjer pomocu docker-compose i spojimo se na njega
5. Moramo napraviti usera i na replici setu da se moze raditi backup, tako da se spojimo na primarnu instancu i kreiramo ga:

        db.createUser(
        {
        user: "BackupUser",  // bio sam lijen da stavim da se zove BackupRestoreUser tako da je ostao samo Backup
        pwd: "123", // 
        roles: [ { role: "backup", db: "admin" } ,
                { role: "restore", db: "admin" } ]
        }
        )

6. Na bash liniji u kontenjeru za backup kucamo:

        mongodump --host=mongors2:27017 --username=BackupUser --password=123  --authenticationDatabase admin -readPreference=secondary --oplog --gzip --archive=/var/backups

7. I imamo dump svih baza

### Vjezba4: RESTORE S MONGORESTORE [link](https://www.mongodb.com/docs/manual/tutorial/restore-replica-set-from-backup/)

1. Prvo spustimo sva tri noda posto cemo raditi "novi replica set"       // ovo necemo raditi kada je produkcijsko okruzenje vec dizemo sasvim novi replica set
2. Pobrisemo svu datu iz sva tri noda
3. Dignemo jedan node i na njemu uradimo inicijalizciju i dodavanje usera

        rs.initiate(
        {
        _id: "rs0",
        members: [
                { _id: 0, host : "mongors1:27017" } ] } )

        use admin
        db.createUser(
        {
        user: "BackupUser",                                      
        pwd: "123", // 
        roles: [ { role: "backup", db: "admin" } ,
                { role: "restore", db: "admin" } ]
        }
        )         

4. Iz kontenjera 'mongobackup' na bash liniji kucamo:

        mongorestore --archive=/var/backups/mongodump-2025-05-26.archive --gzip --host=mongors1:27017 -u BackupUser -p 123 --authenticationDatabase=admin 
5. Provjerimo jesu li podaci tu na prvom nodu al prvo se logiramo sa nasim userom.

        show dbs

6. Ako jesu dizemo i druga dva noda pomocu docker-compose
7. I dodajemo ih u replika set

        rs.add('mongors2:27017')
        rs.add('mongors3:27017')

### Vjezba5: RESTORE PITR [link](https://stackoverflow.com/questions/15444920/modify-and-replay-mongodb-oplog/15451297#15451297):

1. Scenario: U bazi smo napravili danas neku novu kolekciju i jos nesto radili, i sad slucajno pobrisemo neku staru kolekciju jer je bila slicnog imena.
2. Moramo imati backup s oplogom i on mora sadrzavati zajednicku tocku s novim oplogom, znaci scoop oploga mora uhvatiti i backupov.
3. Uradimo dump oploga iz backup kontenjera:

        mongodump --host=mongors2:27017 --username=BackupUser --password=123   --authenticationDatabase admin -d local -c oplog.rs -o oplogD

4. Premjestimo oplog u novi direktorij

        mv oplogD/local/oplog.rs.bson oplogR/oplog.bson

5. Sada mozemo procitati oplog file pomocu bsondump ili naci u bazi ako je jos imamo timestamp kada je dropana kolekcija
6. Posto u mene jos je ziva mongo instanca provjeravam pomocu

        db.oplog.rs.find({
        "op": "c",
        "o.drop": { "$exists": true } // Provjerava da li postoji polje 'drop' unutar 'o' objekta
        }).sort({
        "ts": -1 // Sortiraj po timestampu (najnoviji prvi)
        })

7. Nalazim timestamp 1749562420, i: 1 
8. Sada opet radimo zadnji restore backupa koji imamo i sav proces
9. Onda radimo restore iz oploga:

        mongorestore --host=mongors1:27017 -u BackupUser -p 123 --authenticationDatabase=admin --oplogReplay --oplogLimit 1749562420:1 oplogR

10. Trebali bi sada imati vracenu kolekciju

### Vjezba6: BACKUP SA SKRIPTOM

1. Pravimo svoju [Dockerfile](./images/mongodockerfile.png) sliku 
2. U [docker-compose.yaml](./images/mongobackupskripta.png) pravimo novi servis sa svojom Docker slikom
3. Pravimo [skriptu](./images/mongodoskripta.png) za backup
4. Sada bi se trebao backup praviti jednom dnevno u 02:00h

### Vjezba7: UPGRADE MONGODB NA NOVIJU VERZIJU [link]( https://www.mongodb.com/docs/manual/release-notes/7.0-upgrade-replica-set/)

1. Pogledamo status replica seta jel sve zdravo.
2. Odradimo backup
3. Upoznamo se s novim featursima u novoj verziji pogotovo s compatibilty changes.
4. Svi u setu moraju imati featureCompabilityversion istu, a da to provjerimo ukucamo na svakom:

        db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )

5. Svi nodovi moraju biti u zdravom stanju a da to provjerimo:

        db.adminCommand( { replSetGetStatus: 1 } )
6. Kad smo odradili pripreme mozemo poceti s nadogradnjom, prvo spustimo jedan sekundarni:

        db.adminCommand( { shutdown: 1 } )

7. Dodamo u yaml file novi image i pokrenemo novu instancu, trebala bi se automatski dodati u replica set.
8. Ponovimo istu komandu za provjeru statusa i obavezno pregledati logove da se synca uredno
9. Ovdje valja dodati da vidis da slucajno neki node nema drift ili kasnjenje u replikaciji:

        db.printSecondaryReplicationInfo()

10. Isto uradimo i s drugim nodom
11. Kad dodjemo do primaryu njega moram stepDown da odstupi s te pozicije da bi i njega mogli spustiti kao secundary:

        rs.stepDown(60)

12. I njemu radimo isti korak upgradea

13. Kad je sve gotovo spojimo se na primary gdje cemo potvrditi da smo sigurni da krecemo s novim featursima pomocu(u realnom vremenu pricekamo par dana da vidimo da necemo ici downgrade verzije posto poslije ovoga jedini je downgrade da zovemo direkt u mongo):

        db.adminCommand( { setFeatureCompatibilityVersion: "7.0" } )


## INDEXI [link](https://www.mongodb.com/docs/manual/indexes/)

        db.kolekcija.getIndexes()  # da vidimo indexe koje ima kolekcija

Defaultno imamo index na polju _id.

Single field index:

        db.test.createIndex ({ age: 1 }) 
        # ako je 1 asc a ako je -1 onda je kreiran kao descending
        age_1 # index dobije unique ime ako mu mi to ne definiramo

Mozemo koristiti komandu explain('executionStats' za detaljniji uvid) da vidimo da li query koristi index scan ili collection scan

        db.test.find({ age: { $gt: 30 } }).explain()



Compound index: 

        db.test.createIndex ({name: 1, age: 1})

Compound index moze biti puno efektivniji od single field indexa, jos ne moramo imati vise indexa nego samo jedan. Redoslijed s ljeva na desno je vazan da bi index bio efektivniji i pokrio query.A definiramo ga pomocu ESR (Equality, Sort, Range).

MultiKey index:

Indexi koji se kreiraju na polju koje ima niz.

Hashed index:

Pomazu nam pri shardiranju podataka na vise servera.


Postoje jos text indexi , geospatial... 

Opcije koje mozemo primijeniti na postojece indexe "unique", "sparse", "TTL"...

## SLOW QUERIES

        db.getProfilingStatus()

0 - ne hvata nikakvu datu

1 - hvata samo operacije koje prelaze slowms: 100 (stavljeno po defaultu)

2 - hvata sve operacije (nije preporucljivo, defaultna velicina 1MB)

        db.setProfilingLevel(1)

Stvara se nova kolekcija system.profile i na njoj mozemo izvrsavati upite.

        db.system.profile.find()

## LOGOVI

Za citanje logova [slika](./images/log.png)

Za lakse citanje logova mozemo koristiti alat jq

        sudo cat mongod.log | jq

### Vjezba0 Logrotate [link1](https://betterstack.com/community/guides/logging/database/how-to-start-logging-with-mongodb/) [link2](https://betterstack.com/community/guides/logging/how-to-manage-log-files-with-logrotate-on-ubuntu-20-04/#changing-the-system-logrotate-schedule)

1. U config file sva 3 noda dodajemo: 'logRotate: reopen'

2. U /etc/logrotate.d/ napravimo [skriptu](./images/logrotate.png)(nazovemo je mongodb) za rotiranje logova

3. Pomocu komande mozemo forsirati izvodjenje skripte da testiramo

        sudo logrotate -f /etc/logrotate.d/mongodb

4. Sad ovo uradimo i sa ostala 2 noda

## Dodatni materijali

Tinova [skripta](./images/function%20getRandomInt.txt) za unos vise podataka odjednom 

Mongo [Cheat Sheet](https://www.mongodb.com/developer/products/mongodb/cheat-sheet/)
