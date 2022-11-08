# tspDB

Az alábbi dokumentáció alapján felteéepítettem az Ubuntut, majd a Microsoft Storeból Ubuntu alkalmazást.
https://ubuntu.com/tutorials/install-ubuntu-on-wsl2-on-windows-10#1-overview

Az Ubuntu gépen beregisztráltam, elvégeztem mindent, amit a fenti dokumentációban említenek, ezután pedig a következő dokumentáció szerint telepíteni kezdtem a tspDB-t és a hozzá tartozó eszközöket.
https://tspdb.mit.edu/installation/#ubuntu

Természetesen az Ubuntura leírtak alapján. Feltelepült tehát a PostgreSQL is, illetve ezt az eszközt telepítettem a Windowsra is, amit a pgAdmin eszköz segítségével vizsgáltam.
Az eltérés a python verziókban mutatkozhatott meg csak, én 3.8-at használtm ugyanis, ez volt az Ubuntun.

https://tspdb.mit.edu/installation/#testing-tspdb
A fenti link a tesztelést mutatja be, ez is sikeresen leutott, tehát a tspDB működőképes.

Az alábbi repót klónoztam, hogy példákat nézzek a tspDB használatára:
https://github.com/AbdullahO/tspdb

Itt a 'notebook_examples' mappában található két jupyter notebook, az egyik valós adatokkal dolgozik és timestampek alapján, a másik szintetikus adatokkal és t-időpillanatokkal, unitokkal.

A példák alapján, teljesen megegyezően elkezdtem készíteni egy saját notebook-ot, ami a megfelelő csv fájlt bemenetül véve futott volna. Ezt a notebookot ide a repóba is feltöltöttem. Bejelentkeztem tehát az adatbázis kezelőbe, beolvastam ugyanúgy a csv fájlt és átadtam az sql-nek.Létrehoztam a pindexet, azt megvizsgáltam, majd megpróbáltam a hiányzó adatok behelyettesítését, illetve predikálni. A hiányzó adatok jó eredményeket kaptak, predikció esetén teljesen rossz eredmények születtek.

Ezután kezdtem el próbálkozni a buszos nyers adatok vizsgálatával és az azokon való predikálással. Ehhez beléptem a postgreSQL szerverre ubuntun keresztül a következő oranccsal:
"sudo -u postgres psql postgres"

Majd létrehoztam a megfelelő táblát:
"CREATE TABLE pgfutter1
(vehicle_id bigint, time_stamp timestamp, velocity double precision);"

Közben megtaláltam az Ubuntu fájlkönyvtárát és oda bewmásoltam a repóban is jelenlévő "2020_05_03__23_59_50__2020_05_10__23_59_50_2.csv" csv fájlt. a könyvtár elérési útja:
"\\wsl$\Ubuntu\home\{Ubuntun a username}"

A csv fájl bemásolásához az alábbi parancsot adtam ki:
"COPY pgfutter1 FROM '/home/erik/2020_05_03__23_59_50__2020_05_10__23_59_50_2.csv' CSV DELIMITER ';';"

Ezután létrehoztam a pindexet:
"select create_pindex('pgfutter1', 'timestamp', '{"velocity"}', 'pindex', k => 4);"

De erre hibát dobott, szerintem túl sok rekord volt, így leszűkítettem, hogy cask minden 25. legyen benne.
Ehhez új táblát hoztam létre:
"CREATE TABLE newtable AS SELECT * FROM (SELECT *, row_number() over() rn FROM pgfutter1) foo WHERE foo.rn % 25 = 0;"

Létrehoztam az új pindexet:
"select create_pindex('newtable', 'time_stamp', '{"velocity"}', 'pindex2');"

Majd predikálni kezdtem:
"select * from predict('newtable', 'velocity', '2020-05-08 01:00:00', '2020-05-08 10:00:00', 'pindex2');"
 
Az eredmények itt se voltak megfelelőek.

Eldobtam tehát a time_stamp oszlopot hátha az lehet a gond, illetve egy rn oszlopot amit nem tudok azóta se micsoda:
"ALTER TABLE newtable DROP COLUMN time_stamp;"
"ALTER TABLE newtable DROP COLUMN rn;"

Felvettem egy új oszlopot, mint t-időpillanat, vagy unit:
"ALTER TABLE newtable ADD COLUMN time SERIAL PRIMARY KEY;"

Felépítettem egy másik pindexet a fentiek alapján, és újból predikáltam, hasonlóan rossz eredményekkel:
"select * from predict('pgfutter2','velocity', 500, 'pindex_timenew2');"