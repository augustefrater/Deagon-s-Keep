---
status: accepted
date: 2026-10-28
decision-makers: Quentin (responsable backend)
---

# Développer le serveur en C# sur .NET 10 (LTS)

## Contexte et énoncé du problème

Le sujet laisse la stack libre, à condition de la justifier dans la documentation technique (bloc 01-B). Le serveur porte les parties les plus exigeantes du produit :

* un **moteur de règles piloté par les données** : sorts, capacités et monstres sont décrits avec quelques types d'effets génériques (attaque, sauvegarde, dégâts, soins, état, modificateur, effet libre), interprétés sans code spécifique par sort (02-C, annexe A4) ;
* un **serveur qui fait autorité** : dés tirés par un générateur aléatoire sûr, actions de combat calculées et validées côté serveur (02-E, 03-B) ;
* un **combat assisté** dont chaque résultat est affiché avant application, modifiable, annulable et historisé (02-H, annexe A1) ;
* **deux vues synchronisées en direct**, la vue joueurs ne recevant jamais d'information cachée (02-I, 03-B) ;
* une application mobile qui utilise **le même moteur de règles** que le web (02-K).

Le projet dure de mi-octobre 2026 à fin janvier 2027, en groupe de trois. Chaque membre doit pouvoir expliquer n'importe quelle partie du code en soutenance (03-D). Le responsable backend maîtrise C# / .NET et Go, et n'a pas d'expérience en TypeScript.

Quel langage et quelle plateforme choisir pour le serveur ?

## Facteurs de décision

* **Modélisation du moteur de règles** (poids 25 %) : types somme ou hiérarchies scellées, pattern matching exhaustif, sérialisation JSON polymorphe.
* **Temps réel et filtrage par audience** (20 %) : groupes de diffusion, reconnexion, client navigateur, envoi d'une projection différente par vue.
* **Sécurité, authentification, persistance** (15 %) : hachage, sessions, autorisations, ORM et migrations, protections OWASP disponibles sans les écrire.
* **Maîtrise de l'équipe backend** (15 %) : productivité dès la première semaine, capacité à expliquer le code.
* **Partage de code avec les clients** (10 %) : réutilisation du moteur ou des types côté web et mobile.
* **Outillage, CI/CD, déploiement** (10 %) : Docker, tests en une commande, GitHub Actions, hébergement gratuit.
* **Lisibilité et simplicité** (5 %) : relecture en pull request par un coéquipier qui n'a pas écrit le code.

## Options envisagées

* C# / .NET (ASP.NET Core, SignalR, EF Core)
* Go
* TypeScript (Node.js ou Bun)
* Kotlin (Ktor, Kotlin Multiplatform)
* Rust (Axum)
* Elixir (Phoenix)

## Résultat de la décision

Option retenue : **« C# / .NET »**, sur **.NET 10 (LTS)**, avec ASP.NET Core (Minimal APIs), SignalR, EF Core et PostgreSQL (voir [ADR-0002](0002-base-de-donnees-postgresql-hybride.md)). C'est la seule option qui excelle à la fois sur les deux facteurs les plus lourds (moteur de règles et temps réel) et sur la maîtrise de l'équipe. Elle obtient 4,55 / 5 dans la matrice pondérée, contre 3,50 pour TypeScript, deuxième.

.NET 10 est retenu parce qu'il est en support long jusqu'au 14 novembre 2028. .NET 8 et .NET 9 arrivent en fin de support le 10 novembre 2026, pendant le projet, et .NET 11 sera une version à support court.

| Brique | Choix |
| --- | --- |
| API | ASP.NET Core (Minimal APIs), OpenAPI généré nativement |
| Temps réel | SignalR, un groupe par audience (vue MJ, vue joueurs) |
| Persistance | EF Core + PostgreSQL |
| Authentification | ASP.NET Identity |
| Aléatoire | `System.Security.Cryptography.RandomNumberGenerator` |
| Tests | xUnit + Testcontainers |

Le moteur de règles est une **bibliothèque C# pure**, sans dépendance d'infrastructure, exécutée **uniquement côté serveur**. Le web et le mobile l'utilisent à travers l'API, ce qui satisfait l'exigence d'un moteur unique (02-K).

### Conséquences

* Bon, car un seul moteur est écrit et testé, et aucun client ne peut le contourner.
* Bon, car le format des effets s'exprime directement dans le système de types (records, `[JsonPolymorphic]`), et le compilateur signale tout type d'effet non traité.
* Bon, car les briques de sécurité sont éprouvées (Identity, politiques d'autorisation, anti-CSRF) au lieu d'être écrites à la main.
* Bon, car le moteur isolé rend triviaux les tests de personnages de référence exigés par le sujet.
* Mauvais, car le moteur ne peut pas être partagé avec un front web en TypeScript. Mesure : un endpoint sans état `POST /api/characters/evaluate` renvoie l'état calculé, les erreurs et les choix encore disponibles, pour que les clients n'aient aucune règle à dupliquer.
* Mauvais, car chaque étape de l'assistant de création coûte un aller-retour réseau. Mesure : debounce côté client, endpoint rapide et sans état.
* Mauvais, car quelques calculs purement visuels sont dupliqués côté client (gabarit survolé, distance affichée). Mesure : le serveur recalcule tout au moment de l'action ; cette duplication est assumée.
* Mauvais, car l'image Docker est plus lourde qu'en Go. Mesure : build multi-étapes et image runtime *chiseled*.
* Neutre, car si le mobile est développé en .NET MAUI, il peut référencer directement la bibliothèque du moteur. Le choix du framework mobile reste guidé par l'équipe mobile et l'accessibilité.

### Confirmation

* Un fichier `global.json` épingle le SDK .NET 10, et la CI GitHub Actions compile avec cette version.
* Le projet `Rules` ne référence ni ASP.NET Core ni EF Core ; un test d'architecture le vérifie en CI.
* Les clients web et mobile sont générés depuis le document OpenAPI produit par le serveur ; la CI échoue si le contrat généré diverge.
* Aucun code client ne contient de logique de règles : vérifié en revue de pull request.

## Avantages et inconvénients des options

### C# / .NET

* Bon, car les records immuables, le pattern matching et le JSON polymorphe natif (`[JsonDerivedType]`) conviennent idéalement au format des effets.
* Bon, car SignalR fournit les groupes, la reconnexion et un client JavaScript officiel ; le hub envoie à chaque groupe sa projection filtrée.
* Bon, car ASP.NET Identity, les politiques d'autorisation et EF Core couvrent l'authentification et les migrations.
* Bon, car le responsable backend maîtrise la plateforme.
* Neutre, car .NET 10 est une version LTS supportée au-delà de la fin du projet.
* Mauvais, car le langage demande plus de cérémonie que Go et produit des images Docker plus lourdes.
* Mauvais, car le partage du moteur avec le mobile n'existe que si celui-ci est en .NET MAUI.

### Go

* Bon, car la concurrence native (goroutines, canaux) convient très bien à la diffusion temps réel.
* Bon, car il produit un binaire unique et une image Docker minimale, et `crypto/rand` est en standard.
* Bon, car le langage est très lisible et maîtrisé par le responsable backend.
* Mauvais, car il n'a pas de types somme : chaque effet devient une interface, un switch sur un champ `type` et une désérialisation polymorphe écrite à la main, sans vérification d'exhaustivité.
* Mauvais, car les groupes, la reconnexion et l'authentification sont à assembler soi-même.
* Mauvais, car aucun partage n'est possible avec le web ou le mobile.

### TypeScript (Node.js ou Bun)

* Bon, car avec un web en React et un mobile en React Native, un monorepo permet un package `rules-engine` unique utilisé partout, avec des schémas Zod partagés.
* Bon, car Socket.IO fournit des rooms pour séparer les audiences.
* Mauvais, car le responsable backend n'a aucune expérience dans ce langage, sur un projet de trois mois où chacun doit pouvoir tout expliquer.
* Mauvais, car l'exécution mono-thread pèse sur le calcul de ligne de vue, qui est CPU-bound (gérable, mais à surveiller).
* Mauvais, car la sûreté des types s'arrête à la compilation sans validation à l'exécution, et l'arbre de dépendances demande une vigilance accrue.

### Kotlin (Ktor, Kotlin Multiplatform)

* Bon, car les `sealed class` et le `when` exhaustif conviennent idéalement aux effets.
* Bon, car Kotlin Multiplatform permettrait de partager le moteur avec une application Android.
* Mauvais, car le langage est nouveau pour le responsable backend.
* Mauvais, car l'écosystème serveur (Ktor, Exposed) est moins intégré qu'ASP.NET Core pour l'authentification.
* Neutre, car le gain de partage n'existe que si le mobile est en Kotlin.

### Rust (Axum)

* Bon, car les enums algébriques conviennent parfaitement au moteur et les performances sont maximales.
* Bon, car le moteur pourrait être compilé en WebAssembly pour le navigateur.
* Mauvais, car la courbe d'apprentissage (possession, durées de vie, async) est trop forte pour trois mois.
* Mauvais, car la relecture est difficile pour les coéquipiers, ce qui va à l'encontre de l'exigence d'explicabilité.

### Elixir (Phoenix)

* Bon, car Phoenix Channels et Presence sont parmi les meilleures briques temps réel existantes.
* Bon, car la machine virtuelle BEAM apporte la tolérance aux pannes.
* Mauvais, car le langage et le paradigme sont nouveaux pour l'équipe.
* Mauvais, car le typage dynamique est peu adapté à un format d'effets riche.
* Mauvais, car aucun partage n'est possible avec le web ou le mobile.

## Informations complémentaires

### Matrice de décision

Notes de 1 (inadapté) à 5 (excellent). Le score est la moyenne pondérée.

| Langage | Moteur (25 %) | Temps réel (20 %) | Sécurité (15 %) | Maîtrise (15 %) | Partage (10 %) | Outillage (10 %) | Lisibilité (5 %) | **Score** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C# / .NET | 5 | 5 | 5 | 5 | 2 | 4 | 4 | **4,55** |
| TypeScript | 4 | 4 | 3 | 1 | 5 | 4 | 4 | **3,50** |
| Go | 2 | 4 | 3 | 5 | 1 | 5 | 5 | **3,35** |
| Kotlin | 5 | 3 | 3 | 1 | 3 | 4 | 3 | **3,30** |
| Rust | 5 | 3 | 3 | 1 | 2 | 4 | 1 | **3,10** |
| Elixir | 3 | 5 | 3 | 1 | 1 | 3 | 2 | **2,85** |

**Analyse de sensibilité.** Même avec la note maximale en maîtrise (5 au lieu de 1), TypeScript monte à 4,10, toujours sous C#. Le choix ne dépend donc pas uniquement de la familiarité du responsable backend.

### Architecture cible

```text
src/
  Rules/           moteur, dés, effets, ligne de vue (sans infrastructure)
  Rules.Data/      format des effets, sérialisation, JSON Schema
  Srd.Importer/    import rejouable et versionné du SRD 5e (CC BY 4.0)
  Infrastructure/  EF Core, PostgreSQL, migrations
  Api/             endpoints, hubs SignalR, projections par audience
tests/
  Rules.Tests/     personnages de référence, dés, montée de niveau, vision
  Api.Tests/       filtrage de la vue joueurs, permissions, annulation
```

Exemple de format des effets :

```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "type")]
[JsonDerivedType(typeof(DamageEffect),    "degats")]
[JsonDerivedType(typeof(HealEffect),      "soins")]
[JsonDerivedType(typeof(ConditionEffect), "etat")]
[JsonDerivedType(typeof(ModifierEffect),  "modificateur")]
[JsonDerivedType(typeof(FreeEffect),      "libre")]
public abstract record Effect;

public sealed record DamageEffect(
    DiceExpression Dice, DamageType Type, bool HalfOnSave = false) : Effect;
public sealed record ConditionEffect(Condition Condition, Duration Duration) : Effect;
public sealed record FreeEffect(
    string Name, string Description, Trigger Trigger, Duration Duration) : Effect;
```

### Réévaluation

Cette décision sera réexaminée si :

* un mode hors ligne devient nécessaire côté mobile, ce qui imposerait d'exécuter le moteur sur l'appareil ;
* le serveur est repris par un membre expérimenté en TypeScript, avec un front React et un mobile React Native ;
* les mesures de performance montrent que la synchronisation ou la ligne de vue ne tiennent pas avec une carte chargée.

### Sources

* Sujet EpiQuest, Programme Grande École 5e année, Epitech Paris, 2026-2027 (blocs 01 à 05).
* [.NET and .NET Core Support Policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core), Microsoft, consultée le 28 octobre 2026.
