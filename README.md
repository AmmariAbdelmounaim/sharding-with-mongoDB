### Rapport du TP : **Sharding avec MongoDB**

---

### **1. Installation de MongoDB et Configuration Initiale**

#### **Étapes d'installation et de configuration :**
1. **Création des répertoires nécessaires** :
   - Création d’un répertoire pour le serveur de configuration :
     ```bash
     mkdir /data/configdb
     ```
   - Création des répertoires pour les shards :
     ```bash
     mkdir /data/shard1 /data/shard2 /data/shard3
     ```

2. **Lancement des serveurs** :
   - Lancer le serveur de configuration :
     ```bash
     mongod --configsvr --dbpath /data/configdb --port 28000 --replSet configReplSet
     ```
   - Initialiser le replica set du serveur de configuration :
     ```javascript
     mongosh --port 28000
     rs.initiate();
     ```

   - Lancer les shards :
     ```bash
     mongod --shardsvr --dbpath /data/shard1 --port 30000 --replSet shard1ReplSet
     mongod --shardsvr --dbpath /data/shard2 --port 30001 --replSet shard2ReplSet
     mongod --shardsvr --dbpath /data/shard3 --port 30002 --replSet shard3ReplSet
     ```
   - Initialiser chaque replica set :
     ```javascript
     mongosh --port 30000
     rs.initiate({ _id: "shard1ReplSet", members: [{ _id: 0, host: "localhost:30000" }] });
     ```

3. **Lancer le routeur `mongos`** :
   ```bash
   mongos --configdb configReplSet/localhost:28000 --port 30005
   ```

4. **Connexion au routeur** :
   ```bash
   mongosh --port 30005
   ```

5. **Ajout des shards** :
   ```javascript
   sh.addShard("shard1ReplSet/localhost:30000");
   sh.addShard("shard2ReplSet/localhost:30001");
   sh.addShard("shard3ReplSet/localhost:30002");
   ```

---

### **2. Test de la Base Distribuée**

#### **Nombre de documents par shard** :
1. Insérer des données dans une collection `publis` :
   ```javascript
   use DBLP;
   for (let i = 2000; i < 2025; i++) {
       db.publis.insertMany(
           Array.from({ length: 1000 }, (_, j) => ({ title: `Publication ${j}`, year: i }))
       );
   }
   ```

2. Vérifier la distribution :
   ```javascript
   sh.status();
   ```

#### **Vérification de l’ajout dans le bon shard** :
1. Ajouter un document :
   ```javascript
   db.publis.insert({ title: "Test Publication", year: 2023 });
   ```

2. Vérifier quel shard contient le document :
   ```javascript
   db.publis.find({ year: 2023 }).explain("executionStats");
   ```

#### **Exécution d'une requête `mapReduce`** :
1. Tester une requête `mapReduce` pour compter les publications par année :
   ```javascript
   db.publis.mapReduce(
       function () { emit(this.year, 1); },
       function (key, values) { return Array.sum(values); },
       { out: "yearly_publications" }
   );
   ```

2. Comparer les performances avant et après le sharding.

---

### **3. Importation et Tests sur la Base `Flight`**

#### **Importation de la base `Flight`** :
1. Charger les données avec le fichier NDJSON :
   ```javascript
   let data = cat("C:/Users/Abdel/flights-1m.json");
   let lines = data.split("\n");
   for (let i = 0; i < lines.length; i++) {
       if (lines[i].trim() !== "") {
           let doc = JSON.parse(lines[i]);
           db.flights.insertOne(doc);
       }
   }
   ```

2. Vérifier l'importation :
   ```javascript
   db.flights.countDocuments();
   db.flights.findOne();
   ```

#### **Création des requêtes** :
- **Requête simple** :
   ```javascript
   db.flights.find({ FL_DATE: "2006-01-01" });
   ```
- **Requête complexe** :
   ```javascript
   db.flights.find({ 
       FL_DATE: "2006-01-01", 
       ARR_DELAY: { $gt: 15 } 
   });
   ```

#### **Modification de la clé de répartition** :
1. Activer le sharding pour la base `Flight` :
   ```javascript
   sh.enableSharding("Flight");
   ```

2. Définir la clé de répartition initiale `{ FL_DATE: 1 }` :
   ```javascript
   sh.shardCollection("Flight.flights", { FL_DATE: 1 });
   ```

3. Tester les requêtes avec cette clé et mesurer les performances :
   ```javascript
   db.flights.find({ FL_DATE: "2006-01-01" }).explain("executionStats");
   db.flights.find({ 
       FL_DATE: "2006-01-01", 
       ARR_DELAY: { $gt: 15 } 
   }).explain("executionStats");
   ```

4. Modifier la clé de répartition pour `{ FL_DATE: 1, ARR_DELAY: 1 }` :
   ```javascript
   db.flights.drop();
   sh.shardCollection("Flight.flights", { FL_DATE: 1, ARR_DELAY: 1 });
   ```

5. Réimporter les données et retester les requêtes.

---

### **4. Résultats et Conclusion**

- **Nombre de documents par shard** : La distribution est visible avec `sh.status()`. Les documents sont répartis selon la clé de partitionnement.
- **Impact de la clé de répartition** :
   - Une clé basée sur `{ FL_DATE: 1 }` est efficace pour des requêtes simples sur les dates.
   - Une clé composée `{ FL_DATE: 1, ARR_DELAY: 1 }` optimise les requêtes complexes combinant plusieurs champs.
- **Performances** : Les requêtes routées à un seul shard (`SINGLE_SHARD`) sont plus rapides. Les clés mal choisies mènent à des scans globaux (`COLLSCAN`).

Ce TP montre l'importance de bien choisir la clé de partitionnement pour équilibrer les données et optimiser les performances des requêtes.

--- 
