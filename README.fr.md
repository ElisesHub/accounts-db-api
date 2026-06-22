# Accounts API (Partie 3)

## Présentation

Accounts API est l’API orientée base de données du système de gestion des comptes. Elle expose des endpoints HTTP liés aux comptes et récupère les données depuis une base MySQL.

Ce service se trouve derrière l’Accounts Application API et prend en charge l’accès direct à la base de données.

Le frontend et l’Accounts Application API ne se connectent pas directement à MySQL. L’accès à la base de données est isolé dans cette API.

## Architecture

```text
Portfolio Frontend
        ↓ HTTP + clé API
Accounts Application API
        ↓ HTTP + clé API
Accounts API
        ↓ connexion MySQL
Base de données MySQL
```

## Responsabilités

Ce projet est responsable des éléments suivants :

* Exposer des endpoints HTTP liés aux comptes à destination de l’Accounts Application API
* Authentifier les requêtes à l’aide d’une clé API
* Se connecter à la base de données MySQL des comptes
* Exécuter les opérations de base de données liées aux comptes
* Retourner les données des comptes à l’Accounts Application API
* Gérer les erreurs de validation avec un format de réponse cohérent
* Gérer les exceptions inattendues via un gestionnaire d’exceptions global
* Fournir un endpoint de health check tenant compte de l’état de la base de données

## Structure du projet

```text
AccountsAPI
├── Application
├── Domain
├── Infrastructure
└── Presentation
```

Responsabilités typiques :

* `Domain` — modèles métier liés aux comptes et règles métier principales
* `Application` — interfaces de services, services applicatifs et orchestration des cas d’usage
* `Infrastructure` — repositories MySQL, accès à la base de données, validation de la clé API et configuration
* `Presentation` — contrôleurs API, configuration de l’authentification, politiques d’autorisation, réponses d’erreur, gestion des exceptions et modèles de requête/réponse

## Technologies

* .NET / ASP.NET Core Web API
* Clean Architecture
* Domain-Driven Design
* MySQL
* MySqlConnector
* Authentification par clé API
* Swagger / OpenAPI
* Health checks
* Docker / Docker Compose

## Authentification

L’Accounts API utilise une authentification par clé API.

Les requêtes doivent inclure la clé API dans l’en-tête HTTP suivant :

```text
x-api-key
```

Tous les contrôleurs mappés requièrent la politique d’autorisation `RequireApiKey`.

```csharp
app.MapControllers().RequireAuthorization("RequireApiKey");
```

Si la clé API est absente ou invalide, la requête est rejetée avant d’atteindre l’action du contrôleur.

L’endpoint de health check ne nécessite pas d’authentification afin que Docker puisse vérifier l’état du conteneur sans clé API.

```http
GET /health
```

## Configuration et secrets

Ce dépôt ne contient aucun secret d’exécution, aucune clé API ni aucune chaîne de connexion à la base de données.

Pour le développement local, les valeurs de configuration sensibles sont gérées avec les user secrets .NET. Ces valeurs sont stockées en dehors du dépôt et ne sont pas versionnées dans Git.

Lorsque l’application est exécutée dans le cadre du système complet de gestion des comptes, les valeurs sensibles sont fournies par un dépôt de déploiement séparé via Docker Compose.

L’application prend également en charge les secrets montés dans les conteneurs via `/run/secrets`, lorsqu’ils sont fournis par l’environnement d’exécution. Cette option est facultative et principalement destinée aux déploiements conteneurisés.

## User secrets requis en local

Les user secrets .NET suivants sont requis pour le développement local :

```text
ConnectionStrings:AccountsDb=
AccountsApiKey=
```

Initialiser les user secrets depuis le répertoire du projet Accounts API :

```bash
dotnet user-secrets init
```

Définir les valeurs requises :

```bash
dotnet user-secrets set "ConnectionStrings:AccountsDb" "Server=server;Database=dbname;Uid=userName;Pwd=yourpassword;"
dotnet user-secrets set "AccountsApiKey" "your-accounts-api-key"
```

Exemple de chaîne de connexion locale :

```text
Server=localhost;Port=3306;Database=AccountsDb;Uid=accounts_user;Pwd=yourpassword;
```

Ne versionnez jamais de vraies clés API, des noms d’utilisateur de base de données, des mots de passe ou des chaînes de connexion contenant des informations sensibles.

## Configuration de la clé API

L’Accounts API requiert une seule valeur de clé API :

```text
AccountsApiKey
```

`AccountsApiKey` est utilisée pour valider les requêtes entrantes provenant de l’Accounts Application API.

Si cette valeur est absente, l’application échoue au démarrage.

## Configuration de la base de données

L’API se connecte à MySQL à l’aide de la chaîne de connexion `AccountsDb` :

```text
ConnectionStrings:AccountsDb
```

En développement local, cette chaîne de connexion est chargée depuis les user secrets .NET.

L’application lit la chaîne de connexion avec :

```csharp
builder.Configuration.GetConnectionString("AccountsDb")
```

La chaîne de connexion est enregistrée avec MySqlConnector :

```csharp
builder.Services.AddMySqlDataSource(accountsDbConnectionString);
```

La même chaîne de connexion est également utilisée par le health check :

```csharp
builder.Services.AddHealthChecks()
    .AddMySql(accountsDbConnectionString);
```

L’API échoue au démarrage si `ConnectionStrings:AccountsDb` est absent.

## Prérequis côté base de données

La base MySQL doit exister et contenir le schéma attendu par l’API.

Base attendue :

```text
AccountsDb
```

La base de données doit être initialisée par la configuration de déploiement à l’aide du fichier SQL de seed ou de dump requis.

L’API peut dépendre de la présence de procédures stockées, de tables et de données initiales dans la base. Si la base n’est pas correctement initialisée, les endpoints liés aux comptes peuvent échouer même si l’API démarre correctement.

## Endpoints de l’API

### Health check

```http
GET /health
```

Retourne l’état de santé du service.

Cet endpoint vérifie si l’API peut se connecter à la base de données Accounts.

Il est volontairement non authentifié afin que Docker puisse l’utiliser pour les health checks du conteneur.

### Comptes

Le service expose des endpoints liés aux comptes via ses contrôleurs.

Endpoint de compte attendu :

```http
GET /api/accounts
```

Cet endpoint récupère les données des comptes depuis la base MySQL via la couche repository.

## Flux de requête

Une requête typique suit le flux suivant :

```text
Accounts Application API
  ↓
validation x-api-key
  ↓
Contrôleur Accounts API
  ↓
IAccountsService
  ↓
IAccountsRepository
  ↓
Base de données MySQL
```

L’application enregistre le service de comptes :

```csharp
builder.Services.AddScoped<IAccountsService, AccountsService>();
```

Elle enregistre également le repository de comptes :

```csharp
builder.Services.AddScoped<IAccountsRepository, AccountsRepository>();
```

Cela permet de séparer la logique des contrôleurs, la logique applicative et l’accès aux données.

## Gestion des erreurs

L’application utilise un gestionnaire d’exceptions global :

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
app.UseExceptionHandler();
```

Les erreurs inattendues sont gérées de manière centralisée et retournées avec un format de réponse d’erreur API cohérent.

## Erreurs de validation

Les réponses liées à un état de modèle invalide sont personnalisées.

Les erreurs de validation retournent une réponse structurée contenant :

```text
Code
Message
FieldErrors
TraceId
```

Exemple de structure de réponse pour une erreur de validation :

```json
{
  "code": "ValidationError",
  "message": "One or more validation errors occurred.",
  "fieldErrors": {
    "fieldName": [
      "Validation error message"
    ]
  },
  "traceId": "request-trace-id"
}
```

## Swagger

Swagger est activé uniquement dans les environnements de développement.

Lorsque l’application s’exécute en environnement de développement, Swagger UI est disponible via l’endpoint Swagger configuré.

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Swagger n’est pas activé dans les environnements hors développement.

## Exécution en local

Restaurer les dépendances :

```bash
dotnet restore
```

Configurer les user secrets locaux requis :

```bash
dotnet user-secrets set "ConnectionStrings:AccountsDb" "Server=localhost;Port=3306;Database=AccountsDb;Uid=accounts_user;Pwd=yourpassword;"
dotnet user-secrets set "AccountsApiKey" "your-accounts-api-key"
```

Lancer le projet API :

```bash
dotnet run
```

Si l’exécution se fait depuis la racine de la solution, fournir le chemin du projet :

```bash
dotnet run --project path/to/AccountsAPI
```

La base MySQL doit également être démarrée et accessible via la valeur configurée dans `ConnectionStrings:AccountsDb`.

## Exécution avec Docker Compose

Ce projet est conçu pour être exécuté dans le cadre du système complet de gestion des comptes, via un dépôt de déploiement séparé.

Le dépôt de déploiement contient le fichier `docker-compose.yaml` utilisé pour démarrer ensemble le frontend, l’Accounts Application API, l’Accounts API et la base de données MySQL.

Depuis le dépôt de déploiement, exécuter :

```bash
docker compose up
```

Le dépôt de déploiement est responsable de la configuration d’exécution : clés API, chaînes de connexion à la base de données, variables d’environnement, secrets Docker et fichiers d’initialisation de la base.

Pour ce service, Docker Compose fournit la chaîne de connexion MySQL sous forme de variable d’environnement :

```yaml
environment:
  ConnectionStrings__AccountsDb: "Server=accountsdb;Port=3306;Database=AccountsDb;User ID=accounts_user;Password=${MYSQL_PASSWORD};"
```

Dans la configuration ASP.NET Core, les doubles underscores sont convertis en clés de configuration imbriquées. Ainsi :

```text
ConnectionStrings__AccountsDb
```

est lu par l’application comme :

```text
ConnectionStrings:AccountsDb
```

L’API lit ensuite cette valeur avec :

```csharp
builder.Configuration.GetConnectionString("AccountsDb")
```

Le mot de passe de la base de données n’est pas codé en dur dans le fichier Compose. Il est fourni via la variable d’environnement `MYSQL_PASSWORD` :

```yaml
Password=${MYSQL_PASSWORD}
```

Le même mot de passe est également utilisé par le conteneur MySQL lors de la création de l’utilisateur de base de données :

```yaml
environment:
  MYSQL_DATABASE: AccountsDb
  MYSQL_USER: accounts_user
  MYSQL_PASSWORD: ${MYSQL_PASSWORD:?MYSQL_PASSWORD is required}
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?MYSQL_ROOT_PASSWORD is required}
```

La syntaxe `:?` signifie que Docker Compose échoue au démarrage si la variable d’environnement requise est absente.

L’Accounts API reçoit également sa clé API via un secret Docker :

```yaml
secrets:
  - source: AccountsApiKey
    target: AccountsApiKey
```

Le secret est défini dans le dépôt de déploiement comme suit :

```yaml
secrets:
  AccountsApiKey:
    file: ./secrets/AccountsApiKey.txt
```

À l’exécution, ce secret est monté dans le conteneur et chargé par l’application via le provider de configuration optionnel `/run/secrets`.

La base MySQL est initialisée à l’aide du fichier de dump SQL monté dans le conteneur MySQL :

```yaml
volumes:
  - ./mySQLDb/accountsdb_dump_v1.sql:/docker-entrypoint-initdb.d/accountsdb_dump_v1.sql:ro
```

Ce fichier est exécuté uniquement lors de la première création du volume de données MySQL.

Les données MySQL sont persistées dans le volume nommé :

```yaml
volumes:
  mysql-data:
```

Si le dump SQL change et que la base doit être recréée dans un environnement de développement local, supprimer le volume puis redémarrer le système :

```bash
docker compose down -v
docker compose up
```

Ce dépôt contient uniquement le code source de l’Accounts API. L’orchestration d’exécution, le câblage des services, les variables d’environnement, les secrets et l’initialisation de la base de données sont gérés en dehors de ce dépôt.

## Validation de la configuration

L’application valide la configuration requise au démarrage.

Le démarrage échoue si :

* `AccountsApiKey` est absent
* `ConnectionStrings:AccountsDb` est absent

Le health check échoue si :

* La base MySQL est indisponible
* La chaîne de connexion est incorrecte
* L’utilisateur configuré n’a pas accès à la base
* Le conteneur de base de données est encore en cours de démarrage
* La base de données n’a pas été correctement initialisée

## Dépannage

### L’API échoue au démarrage

Vérifiez que toutes les valeurs de configuration requises sont présentes :

```text
ConnectionStrings:AccountsDb
AccountsApiKey
```

Vérifiez également que la chaîne de connexion est valide.

### Les requêtes retournent une erreur non autorisée

Vérifiez que la requête inclut l’en-tête de clé API requis :

```text
x-api-key
```

Vérifiez également que la clé fournie correspond à la valeur configurée dans `AccountsApiKey`.

### Le health check échoue

Vérifiez que :

* Le conteneur MySQL est en cours d’exécution
* L’hôte de la base de données dans la chaîne de connexion est correct
* Le port de la base de données est correct
* Le nom de la base de données est correct
* Le nom d’utilisateur et le mot de passe sont corrects
* L’utilisateur de base de données a accès à la base configurée
* L’initialisation de la base est terminée

### Les endpoints de comptes échouent

Vérifiez que :

* La base MySQL est en cours d’exécution
* L’API peut se connecter à la base de données
* Les tables requises existent
* Les procédures stockées requises existent
* Le fichier de seed ou de dump de la base s’est exécuté correctement
* L’utilisateur de base de données configuré dispose des droits nécessaires pour exécuter les requêtes ou procédures stockées requises

Si l’API retourne une erreur du type :

```text
PROCEDURE AccountsDb.GetAccounts does not exist
```

cela signifie que la base de données n’a pas été initialisée avec la procédure stockée attendue.

Dans ce cas, vérifiez le fichier de seed ou de dump de la base dans le dépôt de déploiement et recréez le volume de base de données si nécessaire.

Pour les environnements Docker Compose locaux, les scripts d’initialisation MySQL ne s’exécutent que lorsque le volume de base de données est créé pour la première fois. Si le fichier de seed change, il peut être nécessaire de recréer le volume de base de données :

```bash
docker compose down -v
docker compose up
```

## Notes de sécurité

Les secrets ne sont pas stockés dans ce dépôt.

Ne versionnez pas :

* Les clés API
* Les mots de passe de base de données
* Les chaînes de connexion contenant des identifiants
* Les identifiants propres à un environnement
* Les valeurs de configuration de production
* Les fichiers de user secrets locaux
* Les fichiers de secrets générés

Pour le développement local, utilisez les user secrets .NET.

Pour l’exécution conteneurisée, les valeurs sensibles requises sont injectées par la configuration de déploiement via Docker Compose.

La chaîne de connexion à la base de données contient des identifiants sensibles et doit être traitée comme un secret.

## Avertissement

Ce projet est un prototype simple créé uniquement à des fins de démonstration. Il est fourni « en l’état », sans aucune garantie.

L’auteur n’est pas responsable des problèmes pouvant résulter de l’utilisation, de la modification, du déploiement ou de la distribution de ce projet, y compris les pertes de données, les problèmes de sécurité ou les interruptions de service.

Ce projet n’est pas destiné à être utilisé tel quel dans un environnement de production. Avant tout déploiement public ou commercial, il convient de passer en revue la configuration de sécurité, la gestion des secrets, la configuration de la base de données, le flux d’authentification, la gestion des erreurs, les logs et les paramètres d’infrastructure.