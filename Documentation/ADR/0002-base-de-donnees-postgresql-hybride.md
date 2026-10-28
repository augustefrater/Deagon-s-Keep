---
status: accepted
date: 2026-10-28
decision-makers: Quentin (responsable backend)
---

# Utiliser PostgreSQL en modèle hybride (relationnel + JSONB)

## Contexte et énoncé du problème

Le serveur doit persister deux familles de données de nature différente :

* **des données fortement relationnelles** : utilisateurs, campagnes, appartenance avec un rôle (MJ dans l'une, joueur dans l'autre), personnages rattachés à une campagne, cartes, pions, rencontres, combats, jets. Les permissions vérifiées côté serveur traversent toutes ces relations ;
* **des données semi-structurées** : définitions du SRD (sorts, monstres, capacités) et format des effets, de structure hétérogène et polymorphe, ainsi que les choix de création d'un personnage et la géométrie des cartes (murs, portes).

Plusieurs exigences du sujet pèsent directement sur la base :

| Exigence du sujet | Réf. | Ce qu'elle demande à la base |
| --- | --- | --- |
| Résultat d'une action modifiable, annulable et historisé | 02-H, A1 | Écrire plusieurs entités de façon atomique ; journal ordonné des actions |
| Permissions vérifiées côté serveur (MJ, joueur, campagne) | 03-B | Requêtes transverses fiables entre utilisateurs, campagnes et personnages |
| Import du SRD rejouable et versionné ; règles décrites comme des données | 02-C | Stocker des documents hétérogènes, rattachés à une version d'import |
| Bestiaire : recherche et filtres (type, facteur de puissance, taille) | 02-G | Index sur des attributs et recherche textuelle tolérante |
| Installation en une commande ; nouveau contributeur opérationnel en 15 minutes | 03-B | Démarrage local simple, sans étape d'initialisation manuelle |

Faut-il une base relationnelle (PostgreSQL) ou une base documentaire (MongoDB) ?

## Facteurs de décision

* **Atomicité multi-entités** : une action de combat touche plusieurs créatures, des ressources et le journal, et doit pouvoir être annulée d'un bloc.
* **Intégrité référentielle** : campagnes, rôles et personnages doivent rester cohérents même en cas de bug applicatif.
* **Stockage de documents polymorphes** : les définitions du SRD et le format des effets varient d'une entité à l'autre.
* **Recherche** : filtres et recherche textuelle tolérante dans le bestiaire.
* **Simplicité d'installation** : le test du nouvel arrivant (15 minutes avec le seul README).
* **Intégration à la stack .NET** retenue dans l'[ADR-0001](0001-langage-serveur-csharp-dotnet.md) : EF Core, migrations, Testcontainers.

## Options envisagées

* PostgreSQL en modèle hybride (tables relationnelles + colonnes JSONB)
* MongoDB (base documentaire)

## Résultat de la décision

Option retenue : **« PostgreSQL en modèle hybride »**, car le cœur du sujet (combats atomiques et annulables, permissions par campagne, import versionné) repose sur l'intégrité et les transactions, qui sont les points forts de PostgreSQL. La souplesse documentaire n'est utile que sur quelques zones du modèle, et JSONB les couvre.

Chaque donnée est rangée selon sa nature :

| Donnée | Stockage |
| --- | --- |
| Utilisateurs, campagnes, appartenances, personnages, cartes, rencontres, combats | Tables relationnelles |
| Définitions SRD (sorts, monstres, capacités, effets) | Colonnes clés indexées + `definition jsonb` |
| Choix de création, inventaire, notes du personnage | `jsonb` dans la table du personnage |
| Murs, portes, décor d'une carte | `jsonb` (liste de segments), chargé en mémoire pour la ligne de vue |
| Journal de combat | Table en ajout seul : `(combat_id, seq bigserial, type, payload jsonb)` |

### Conséquences

* Bon, car une action de combat et son annulation sont atomiques, quel que soit le nombre de cibles.
* Bon, car la base garantit l'intégrité des campagnes, des rôles et des personnages, même en cas de bug applicatif.
* Bon, car une seule image Docker démarre sans initialisation, ce qui garde le test du nouvel arrivant simple.
* Bon, car le journal de combat est une table ordonnée : rejeu et annulation s'écrivent en SQL simple.
* Bon, car le JSONB absorbe les changements du format des effets sans migration de colonnes ; seul l'import est rejoué.
* Mauvais, car les migrations demandent de la discipline. Mesure : migrations EF Core versionnées, appliquées automatiquement au démarrage en local et vérifiées en CI.
* Mauvais, car le contenu des colonnes JSONB est moins contraint que des colonnes typées. Mesure : validation par le JSON Schema du format des effets à l'import, et contraintes `CHECK` minimales.
* Mauvais, car une fiche de personnage se répartit entre colonnes et JSONB. Mesure : l'agrégat est chargé en une requête et reconstruit en C# par EF Core.

### Confirmation

* Le `docker-compose.yml` démarre l'image officielle PostgreSQL, avec une version majeure épinglée.
* Les migrations EF Core sont versionnées dans le dépôt ; la CI échoue si le modèle a des changements sans migration correspondante.
* Les tests d'intégration tournent sur une base PostgreSQL éphémère (Testcontainers), y compris les tests d'annulation d'une action de combat multi-cibles.
* L'import du SRD valide chaque définition contre le JSON Schema du format des effets avant écriture.

## Avantages et inconvénients des options

### PostgreSQL en modèle hybride

* Bon, car les transactions multi-lignes ACID fonctionnent sans configuration : une boule de feu sur quatre cibles (PV de quatre créatures, emplacement de sort, état, entrée de journal) s'applique ou s'annule dans un seul `BEGIN … COMMIT`.
* Bon, car l'intégrité est vérifiée par la base : clé étrangère `character.campaign_id` avec la règle `ON DELETE` adaptée, `UNIQUE (campaign_id, user_id)` sur l'appartenance, `CHECK` sur les valeurs (PV ≥ 0, niveau entre 1 et 20).
* Bon, car JSONB combine colonnes indexées et documents : index GIN, opérateur `@>`, `jsonb_path_query`, `CHECK` sur la présence d'un champ ; Npgsql mappe ces colonnes vers des types C# avec EF Core.
* Bon, car la recherche dans le bestiaire se fait sans service externe : index B-tree pour les filtres, `pg_trgm` pour la tolérance aux fautes (« gobelin » contre « goblin »), `tsvector` avec `unaccent` pour le plein texte en français.
* Bon, car les requêtes transverses (« tous les personnages des campagnes où cet utilisateur est MJ ») s'écrivent en une jointure optimisée par le planificateur.
* Bon, car l'import du SRD se versionne par une table `srd_version` référencée par chaque entité, et se rejoue dans une transaction qui réussit en entier ou ne laisse aucune trace.
* Bon, car l'outillage est mature : migrations EF Core, Testcontainers, image Docker officielle.
* Mauvais, car les parties imbriquées d'un agrégat (fiche, carte) se répartissent entre colonnes et JSONB.
* Mauvais, car chaque évolution des colonnes relationnelles demande une migration.

### MongoDB

* Bon, car le modèle document correspond aux agrégats : une fiche de personnage ou une carte complète se lit et s'écrit en une seule opération.
* Bon, car un combat modélisé comme un seul document est mis à jour atomiquement par un `updateOne` avec `$inc`, `$set` et `$push`, sans transaction.
* Bon, car les structures hétérogènes d'Open5e ou de la 5e API s'insèrent telles quelles, et le driver C# gère nativement les discriminateurs polymorphes (`_t`, `[BsonKnownTypes]`).
* Bon, car le schéma évolue sans migration, ce qui fait gagner du temps au début du projet.
* Neutre, car les change streams pourraient alimenter la synchronisation des vues, mais le serveur pousse déjà les changements via SignalR au moment où il les écrit.
* Mauvais, car le journal de combat doit sortir du document de combat, à cause de la limite de 16 Mo par document.
* Mauvais, car l'intégrité entre campagnes, rôles et personnages est à la charge du code, et `$lookup` est moins performant qu'une jointure.
* Mauvais, car les transactions multi-documents et les change streams exigent un *replica set*, qu'il faut initialiser par script dans le `docker-compose`, ce qui complique le test du nouvel arrivant.
* Mauvais, car la recherche textuelle avancée passe par Atlas Search, et les index texte locaux sont basiques.
* Mauvais, car le provider EF Core pour MongoDB est plus jeune et moins complet que Npgsql.

## Informations complémentaires

### Comparaison sur les besoins du sujet

| Besoin du sujet | PostgreSQL | MongoDB |
| --- | --- | --- |
| Action de combat multi-cibles, atomique et annulable | Transaction native | Atomique si tout est dans un seul document ; sinon transaction multi-documents, qui exige un *replica set* |
| Comptes, campagnes, rôles, permissions | Clés étrangères, jointures, contraintes | Intégrité à la charge du code, `$lookup` moins performant |
| Définitions de règles polymorphes | JSONB + colonnes indexées | Natif |
| Fiche de personnage, état d'une carte | JSONB pour les parties imbriquées | Natif, un document par agrégat |
| Recherche dans le bestiaire | `pg_trgm`, `tsvector`, sans service externe | Index texte basiques ; la recherche avancée passe par Atlas Search |
| Import rejouable et versionné | Transactionnel, référencé par clé étrangère | Simple à insérer, cohérence du rejeu à gérer |
| Installation en une commande | Une seule image | *Replica set* mono-nœud à initialiser |
| Écosystème .NET | EF Core + Npgsql, migrations | Driver officiel solide ; provider EF Core plus jeune |

### Illustration du modèle hybride

```sql
CREATE TABLE srd_spell (
    id           uuid PRIMARY KEY,
    version_id   uuid NOT NULL REFERENCES srd_version(id),
    slug         text NOT NULL,
    name         text NOT NULL,
    level        smallint NOT NULL CHECK (level BETWEEN 0 AND 9),
    school       text NOT NULL,
    definition   jsonb NOT NULL CHECK (definition ? 'resolution'),
    UNIQUE (version_id, slug)
);
CREATE INDEX ON srd_spell USING gin (definition jsonb_path_ops);

CREATE TABLE combat_event (
    combat_id    uuid NOT NULL REFERENCES combat(id) ON DELETE CASCADE,
    seq          bigserial,
    type         text NOT NULL,          -- action, effet_mj, annulation...
    payload      jsonb NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (combat_id, seq)
);
```

### Réévaluation

Cette décision sera réexaminée si :

* le groupe choisit de modéliser tout le jeu en agrégats autonomes (un document par combat, un par carte), sans requêtes transverses ;
* le format des données de règles devient si instable que même le JSONB et les migrations freinent le travail ;
* les mesures de performance montrent que le chargement de l'état d'une partie devient un goulot d'étranglement, ce qui justifierait d'abord un cache en mémoire avant de changer de base.

### Sources

* Sujet EpiQuest, Programme Grande École 5e année, Epitech Paris, 2026-2027 (blocs 01 à 05).
* [ADR-0001 : Développer le serveur en C# sur .NET 10 (LTS)](0001-langage-serveur-csharp-dotnet.md).
