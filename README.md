# API REST avec Express

Cette doc synthétise les notions vues dans les quêtes Express 2 à 5 sur :
* La lecture de données (GET)
* La création de données (POST)
* La mise à jour de données (PUT)
* L'effacement de données (DELETE)

## Mise en place de la base de données

### Création d'un fichier de configuration

> :warning: Cette doc a été faite il y a longtemps. Cette section sur la configuration est un peu obsolète. Utiliser plutôt [dotenv](https://www.npmjs.com/package/dotenv) pour gérer les paramètres de configuration d'une app.

Pour ne pas stocker dans le dépôt GitHub les paramètres de la base de données, il est nécessaire de stocker ces paramètres dans un fichier non suivi par git. Vous pouvez le nommer, par exemple, `db-config.json` ou `db-settings.json`, et le stocker directement sous `back`.

Le nom de ce fichier doit être ajouté dans le `.gitignore` **du dossier `back`**. Par contre, à des fins de documentation, pour qu'une personne puisse récupérer le dépôt et l'utiliser, vous pouvez écrire un fichier `db-config.sample.json` ou `db-settings.sample.json`, contenant des valeurs "bidon" et qui, lui, peut (et doit) être commité

Voici un exemple de fichier `db-config.sample.json` :

```json
{
  "host": "localhost",
  "database": "myprojectdb",
  "user": "myuser",
  "password": "mypassword"
}
````

Vous pouvez copier ce fichier en `db-config.json` et adapter les valeurs (par exemple pour un hypothétique projet "TodoList") :

```json
{
  "host": "localhost",
  "database": "wcs_project3_todolist",
  "user": "todolist",
  "password": "WCSTodoList"
}
```

### Initialisation de la base de données

Sur le même principe, vous pouvez créer un fichier `init.sql` (par exemple sous `back/database`) qui servira uniquement à créer la base de données (ou à la recréer si vous voulez repartir d'une base de données "propre").

Ce fichier est également à référencer dans `.gitignore`. Voici à quoi il ressemblerait dans mon projet "TodoList" :

```sql
CREATE DATABASE wcs_project3_todolist CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'todolist'@'localhost' IDENTIFIED BY 'WCSTodoList';
GRANT ALL PRIVILEGES ON wcs_project3_todolist.* TO 'todolist'@'localhost';
FLUSH PRIVILEGES;
```

Il faut évidemment mettre des valeurs concordantes avec le fichier `db-config.json` ou `db-settings.json` :
* remplacer le nom de la bdd sur la 1ère ligne (`CREATE DATABASE`) et la 3ème (`GRANT ALL PRIVILEGES`)
* remplacer le nom d'utilisateur sur la 2ème ligne et la 3ème
* remplacer le password sur la 2ème ligne

**La personne qui crée ces deux fichiers peut les communiquer à ses camarades.** Cela permet à tout le monde d'avoir une même configuration.

## Mise en place sur Express

### Installation du module `mysql`

Sur vos projets, dans le dossier `back`, il faut commencer par installer le module `mysql` :

    npm install mysql

### Création du code de connexion à la base de données

Pour utiliser MySQL avec Node.js, il faut procéder en deux étapes :
* Créer un fichier qui initialise la connexion à la base de données, et qui *exporte* l'objet `connection`.
* Importer ce fichier depuis tous les fichiers qui en ont besoin, pour pouvoir appeler la méthode `query` sur la connexion.

Vous pouvez appeler le fichier de connexion : `connection.js`, `db.js`, etc. Dans les quêtes, il est appelé `conf.js`, mais personnellement, je ne trouve pas ça très explicite.

Voici son contenu (notez qu'il importe le fichier de paramètres de la base de données et qu'il faut donc adapter le nom le cas échéant) :

```javascript
// db.js
const mysql = require('mysql');
const settings = require('./db-settings.json');

const connection = mysql.createConnection(settings);

module.exports = connection;
```

Puis il faut l'importer dans un fichier Express, par exemple pour ma Todo-list (ce n'est pas un problème d'appeler la const `db` au lieu de `connection` dans le fichier qui importe) :

```javascript
const db = require('./db');

app.get('/api/tasks', (req, res) => {
  // Appeler la méthode query sur l'object connection
  db.query('SELECT id, title, status FROM task', (err, results) => {
    if (err) {
      return res.status(500).json({
        error: err.message,
        sql: err.sql
      })
    }
    return res.json(results);
  });
});
```

### Structuration des routes

Dans le cours API REST avec Express, on a parlé des quatre opérations sur une API : Create, Read, Update, Delete, abrégées sous l'acronyme CRUD. Cela suggérerait que pour chaque type de données, on a quatre routes à créer. En réalité, il y en a cinq :

* Une route pour récupérer toutes les lignes d'une table ("Read all", accessible en GET)
* Une route pour récupérer une ligne d'une table ("Read one", accessible en GET), typiquement par son `id`
* Une route pour créer une ligne dans une table ("Create one", accessible en POST)
* Une route pour mettre à jour une ligne dans une table ("Update one", accessible en PUT)
* Une route pour supprimer une ligne dans une table ("Delete one", accessible en DELETE)

Si vous écrivez toutes ces routes dans un seul fichier `index.js` ou `app.js` ou `server.js`, celui-ci va vite devenir *très lourd* et impossible à maintenir.

Il faut donc créer des fichiers de routes : grosso modo, un par entité dans votre base de données (excepté pour les tables de jointure).

Par exemple, dans mon appli "TodoList", je veux gérer deux types de données :
* Des "tâches" (Table `task`)
* Des utilisateurs, chacun pouvant avoir ses propres tâches (Table `user`)

Je vais donc structurer mon appli comme ceci, au niveau du backend :

    back/
      db-settings.js
      db.js
      index.js
      routes/
        tasks.js
        users.js

Je stocke les fichiers de routes dans un répertoire dédié.

Voici le contenu de mon fichier `tasks.js` :

```javascript
// tasks.js
const express = require('express');
const db = require('../db');

// Création d'un "routeur" en appelant la méthode Router()
// fournie par Express
const router = express.Router();

// Route "READ all"
// NOTES
//  * ici je mets la racine et pas explicitement /api/tasks
//  * sur un "READ all", on veut TOUJOURS renvoyer un tableau,
//    même vide, par conséquent la route READ all ne renvoie
//    JAMAIS une erreur 404
router.get('/', (req, res) => {
  // Appeler la méthode query sur l'object connection
  db.query('SELECT id, title, status FROM task', (err, results) => {

    // Gestion d'erreur
    if (err) {
      return res.status(500).json({
        error: err.message,
        sql: err.sql
      })
    }
    // IMPORTANT: il faut garder à l'esprit que results
    // est toujours (genre, TOUJOURS) un tableau
    return res.json(results);
  });
});

// Route "READ one"
// NOTES
//   * ici je mets /:id et pas /api/tasks/:id
//   * sur un "READ one", on veut TOUJOURS renvoyer un OBJET,
//     par conséquent si on ne trouve pas l'objet demandé dans la BDD,
//     on doit renvoyer une erreur 404
router.get('/:id', (req, res) => {
  // Appeler la méthode query sur l'object connection
  // Dans la requête on limite les résultats à ceux dont l'id
  // correspond à celui passé en paramètre
  db.query('SELECT id, title, status FROM task WHERE id = ?', req.params.id, (err, results) => {
    if (err) {
      return res.status(500).json({
        error: err.message,
        sql: err.sql
      })
    }
    // results est TOUJOURS un tableau
    // MAIS ici on veut renvoyer UN objet de ce tableau
    // Si le tableau est vide, cela signifie qu'on n'a trouvé aucun
    // résultat correspondant au paramètre req.params.id
    // Il faut alors renvoyer une 404
    if (results.length === 0) {
      return res.status(404).json({
        error: `No record with id ${req.params.id}`
      });
    }
    // Si un résultat est trouvé, il n'y en a forcément qu'UN SEUL
    // (les id étant uniques). On doit renvoyer l'objet à l'index 0
    return res.json(results[0]);
  });
});

// Route "CREATE one"
router.post('/', (req, res) => {
  db.query('INSERT INTO task SET ?', req.body, (err, status) => {
    // Toujours vérifier les erreurs
    if (err) {
      return res.status(500).json({
        error: err.message,
        sql: err.sql
      })
    }

    // Si aucune erreur ne s'est produite, status n'est pas un
    // tableau de status comme sur un SELECT. C'est un objet indiquant
    // des statistiques sur l'exécution de la requête. Il contient
    // une clé insertId qui nous indique l'id de la ligne qui vient
    // d'être ajoutée à la base de données
    // Il faut faire une DEUXIEME requête pour récupérer l'objet et l'envoyer au client
    db.query(
      'SELECT id, title, status FROM task WHERE id = ?',
      status.insertId,
      (err, results) => {
        // ATTENTION comme results est un tableau on envoie à nouveau
        // son premier (et seul) élément
        // Le code status 201 indique qu'on a créé une ressource
        res.status(201).json(results[0]);
      }
    )
  });
});

// Route "UPDATE one"
router.put('/:id', (req, res) => {
  db.query('UPDATE task SET ? WHERE id = ?', [req.body, req.params.id], (err, status) => {
    // Toujours vérifier les erreurs
    if (err) {
      return res.status(500).json({
        error: err.message,
        sql: err.sql
      })
    }
    // Ici on peut renvoyer le body tel quel
    res.json(req.body);
  });
});

// Route "DELETE one"
router.delete('/:id', (req, res) => {
  db.query('DELETE FROM task WHERE id = ?', req.params.id, (err, status) => {
    // Toujours vérifier les erreurs
    if (err) {
      return res.status(500).json({
        error: err.message,
        sql: err.sql
      })
    }
    // Ici on peut renvoyer le code 204 et aucune donnée
    res.sendStatus(204);
  });
});

// On exporte le routeur
module.exports = router;

```

Voici comment je vais l'utiliser dans mon appli Express (sur les dépôts projets, le mieux est de le nommer `back/index.js`):

```javascript
const express = require('express');
const bodyParser = require('body-parser');
const tasksRouter = require('./routes/tasks');

const app = express();

// Permettre le décodage de données JSON envoyées
// par le client
app.use(bodyParser.json());

// Intégrer les routes du routeur "tâches",
// sous le préfixe commun /api/tasks
// Ainsi une requête en GET sur /api/tasks/4
// arrivera sur la route router.get('/:id' ...) du routeur
app.use('/api/tasks', tasksRouter);

app.listen(process.env.PORT || 5000, err => {
  if (err) {
    console.error(err.message);
  } else {
    console.log(`Server is listening on port ${process.env.PORT || 5000}`);
  }
});
```
