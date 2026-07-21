# Présentation — Design patterns & architecture du projet compliance

**Légende étiquettes** : `SOLID` (Single Responsibility / Open-Closed / Liskov / Interface Segregation / Dependency Inversion) · `DRY` (Don't Repeat Yourself) · `KISS` (Keep It Simple) · `Loi de Déméter` (un objet ne parle qu'à ses voisins directs).

## Sommaire

**Patterns** — 01 Hexagonale · 02 DDD · 03 CQRS · 04 Repository/Loader · 05 Factory (create/load) · 06 Clock/IdGenerator · 07 Strategy · 08 Builder · 09 Event-Driven · 10 Value Object · 11 Mediator

**Validation** — 12 Contrôleur vs Domain

**Tests** — 13 Découpage miroir · 14 4 classes de base · 15 ApplicationCore · 16 InputAdapter · 17 OutputAdapter · 18 Given/When/Then · 19 Bénéfices

**Outillage** — 20 Qualité & doc API

**Métier** — 21 Anomalies

---

## Patterns

### 01 — Architecture hexagonale (Ports & Adapters)

`SOLID · DIP` `Loi de Déméter`

**Problème résolu** : Découpler la logique métier des détails techniques (DB, HTTP, RabbitMQ). Le Domain ne connaît rien de l'Infrastructure.


Dépendance dans un seul sens : `Infrastructure → Application → Domain`

**Exemple réel — use case création de recette, 3 couches successives**
- `src/Recipe/Model/Recipe.php` — Règles métier
- `src/Recipe/Application/Command/CreateRecipe/CreateRecipeCommandHandler.php` — Orchestration
- `src/Recipe/Infrastructure/Http/Controller/CreateRecipeController + RecipeDoctrineRepository.php` — I/O concret

> Domain ignore tout de Symfony et Doctrine. Ça permet de changer la DB ou le framework sans toucher aux règles métier, et de tester le métier sans base de données via des repositories in-memory.

### 02 — DDD — Domain-Driven Design

`SOLID · SRP` `Loi de Déméter`

**Problème résolu** : Avec un modèle anémique (getters/setters publics + logique dans des services), rien n'empêche un service d'appeler `setPrice(0)` sans passer par les vérifications métier (impossible de modifier un montant de facture) — l'invariant "Toute facture transmise est immuable" peut être violé depuis n'importe quel point d'entrée, et la même règle doit être recopiée dans chaque service qui touche l'entité. DDD répond à ça : coller le code au langage métier et protéger les invariants dans le modèle lui-même, pas dans des services autour.

**Exemple réel**
- `src/Billing/Domain/Model/Invoice.php` — Pas de setters publics pour un attribut, entité riche porteuse de ses propres règles — ex. noms de méthode métier : `addNewCreditNote(...)`,

Les règles métier vivent dans le Domain — ce n'est jamais SQL (contrainte, trigger) qui porte une règle métier, c'est le code PHP du modèle.

Distinction Aggregate / Entity : `Recipe` est la racine d'agrégat (Aggregate Root) qui garantit la cohérence globale ; ses sous-objets (`ListOfIngredients`, `Cooking`) sont des Entity internes, modifiées uniquement via la racine. Ça permet une gestion plus fine de la concurrence — un verrou/version sur l'agrégat racine plutôt que sur chaque sous-partie séparément.

> Le modèle domain n'est pas un sac de données — il expose des comportements métier nommés et garantit ses propres invariants, pas les services autour.

### 03 — CQRS

`SOLID · ISP` `KISS`

**Problème résolu** : Séparer écriture (mutation + invariants) et lecture (projection optimisée, sans règle métier).

**Exemple réel — Command**
- `AddIngredientToRecipeCommand` — `final readonly class` implements `CommandInterface`
- `AddIngredientToRecipeCommandHandler` — Constructeur + `__invoke`

**Exemple réel — Query**
- `ReadRecipeQuery` — `final readonly class` implements `QueryInterface`
- `ReadRecipeQueryHandler` — Constructeur + `__invoke`
- `RecipeDto` — DTO, retourné au contrôleur

> Le write side passe par le domain model et ses invariants. Le read side court-circuite le domain, va direct en DTO via un Loader — pas besoin d'objet métier juste pour afficher une liste, un simple DTO suffit.

### 04 — Repository vs Loader

`SOLID · ISP` `KISS`

**Problème résolu** : Repository = persistance d'agrégats (écriture, cohérence). Loader = lecture de projections (perf, sans règle métier).

**Exemple réel**
- `Domain/Interface/RecipeRepository` — `save`, `getById` — interface
- `RecipeDoctrineRepository` — `extends DoctrineRepository implements RecipeRepository`
- `Application/Loader/GetRecipeLoader` — retourne `PaginatedResult<RecipeDto>`
- `Infrastructure/Loader/RecipeDoctrineLoader` — QueryBuilder → DTO direct, pas d'hydratation domain

`QueryHandler` → jamais `Repository` · `CommandHandler` → jamais `Loader`

> Si je charge une entité pour la modifier → Repository. Si j'affiche juste des données → Loader. Mélanger les deux casse la séparation CQRS et alourdit les lectures pour rien et compliquera les futures optimisations.

### 05 — Factory Method statique — create() vs load()

`SOLID · SRP`

**Problème résolu** : Constructeur public absent → force à passer par une factory nommée qui exprime l'intention : création métier ou reconstitution technique.

**Exemple réel**
- `Domain/Model/Recipe::create(...)` — Nouvelle entité, applique les règles métier
- `Domain/Model/Recipe::load(array $data): self` — Réhydratation depuis Doctrine, saute la validation (donnée déjà validée)

`load()` est réservé à l'infrastructure (hydratation Doctrine) et aux builders de test — jamais appelé depuis un `CommandHandler` pour créer une entité.

> Deux entrées, deux intentions. `create` = use case métier avec validation. `load` = reconstitution de confiance, réservée à l'infra et aux builders de test.

### 06 — Injection de dépendance

`SOLID · DIP`

**Problème résolu** : Aucun `new DateTimeImmutable()` ni `Uuid::v4()` caché dans le domain → tests déterministes.

**Exemple réel**
- `CreateNewInvoiceCommandHandler` — Injecte `ClockInterface` et `IdGeneratorInterface` au constructeur
- `Shared/Infrastructure/Utils/Clock, UidSymfonyGenerator` — Implémentations concrètes

Lien direct avec le TDD : injecter Clock/IdGenerator plutôt que les générer dans le domain, c'est ce qui rend un test unitaire déterministe — sans ça, impossible de prédire l'UUID ou l'horodatage attendu dans une assertion.

> Le handler injecte l'ID et la date générés, le domain les reçoit en paramètre. En test, je fournis un clock/id fake → assertions déterministes, pas de flakiness liée au temps.

### 07 — Strategy Pattern

`SOLID · OCP`

**Problème résolu** : Choisir dynamiquement l'algorithme de sérialisation selon le type d'event RabbitMQ reçu.

**Exemple réel**
- `Shared/Infrastructure/Serializer/StrategyRabbitMqEventSerializer` — Itère `#[AutowireIterator('app.rabbitmq_serializer')]`
- `CreateCustomerEventSerializer` — Une stratégie parmi d'autres, expose `support()` pour se déclarer candidate

> Au lieu d'un switch géant, chaque type d'event a sa propre stratégie de sérialisation, injectée via tag Symfony. Ajouter un event = ajouter une classe, zéro modif du serializer central — Open/Closed.

### 08 — Builder Pattern (test helpers)

`DRY` `KISS`

**Problème résolu** : Construire des objets domain complexes pour les tests sans dupliquer des dizaines de paramètres partout.

> Les builders vivent uniquement côté test, jamais en prod — ils construisent des fixtures lisibles, pas des entités dans le vrai flux métier.

> lien utile : https://refactoring.guru/design-patterns/builder

### 09 — Event-Driven / Observer

`SOLID · OCP` `SOLID · DIP`

**Problème résolu** : Découpler les effets de bord (notifier d'autres services) de la logique de mutation.

```markdown
Event -> EventListener -> commandBus->dispatch()
```

**Exemple réel**
- `Shared/Application/Interface/EventBusInterface` — `dispatch(SharedEventInterface $event)`
- `Shared/Infrastructure/Bus/RabbitMqEventBus` — Wrap Messenger + `AmqpStamp`
- `BoundedContext/Infrastructure/EventListener/CreateInvoiceEventListener` — traduit l'event entrant `RepairedVehicleEvent` en `CreateInvoiceCommand`

Ça découple totalement l'émission de l'event et la lecture des règles métier qui en découlent : celui qui publie ne sait pas qui consomme, ni ce que le consommateur en fait.

> Le handler ne sait pas qui écoute — il publie un event, point. Côté consommateur, on ne traite jamais l'event brut : on le traduit immédiatement en Command, donc les invariants métier restent centralisés dans le CommandHandler.

### 10 — Value Object

`Loi de Déméter` `KISS`

**Problème résolu** : Encapsuler une valeur immuable avec son propre comportement, éviter la primitive obsession.

**Exemple réel**
- `Shared/Domain/Model/ImmutableDateTime.php:13` — `final class extends DateTimeImmutable`, factories `createFromInterface` / `createFromFormat`
- `ImmutableDateTime::createFromInterfaceSafe` — Retourne `?self` — variante "Optional"

Exemple métier plus parlant qu'une date : un montant facturé devrait porter sa devise avec lui (`Amount` + `Currency`, pas un simple `float`) — deux montants dans des devises différentes ne s'additionnent pas, et c'est le VO qui refuse l'opération, pas un `if` éparpillé dans les handlers.

> Toutes les dates du domain passent par ce VO, jamais `\DateTimeImmutable` brut — garantit l'immutabilité et un point unique pour la logique de date.

### 11 — Mediator — CommandBus / QueryBus

`SOLID · DIP` `Loi de Déméter`

**Problème résolu** : Découpler le contrôleur du handler concret — il ne connaît que l'interface du bus.

**Exemple réel**
- `Shared/Infrastructure/Bus/MessengerBus.php:16` — Implémente `CommandBusInterface` ET `QueryBusInterface` via `HandleTrait` (Symfony Messenger)

> Le contrôleur fait juste `$queryBus->ask()` ou `$commandBus->dispatch()`. Il ignore totalement quel handler traite la demande — ajouter ou retirer un handler n'impacte jamais le contrôleur.

---

## Validation

### 12 — Validation — deux niveaux, deux responsabilités

`SOLID · SRP`

**Constat** : Aucune entrée validation dans les chaînes de bus Messenger — `config/packages/validator.yaml` ne configure que le service Symfony Validator de base.

**Niveau 1 — contrôleur (basique)** : Sur les DTOs HTTP (payload/query), via attributs `#[Assert]`, validés avant dispatch sur le bus. Contrôle de forme uniquement : type, présence, format — aucun contrôle métier.
- `Infrastructure/Http/Controller/CreateBillingComment/CreateBillingCommentPayload.php:12` — `#[Assert\NotBlank(message: 'Le commentaire ne peut pas être vide.')]`
- `Shared/Application/Model/PaginationContext.php:12,14` — `#[Assert\Positive]` sur page/itemsPerPage

**Niveau 2 — Domain (métier)** : Tous les contrôles métier vivent dans `create()` et les mutateurs du modèle Domain (voir 05 Factory) — c'est le seul endroit qui peut lever une exception métier (`ForbiddenToModifyACompliantRideException`, etc).

> Deux niveaux, deux responsabilités : le contrôleur vérifie que la donnée est bien formée, le Domain vérifie qu'elle a du sens métier. Le premier ne remplace jamais le second.

---

## Tests

La structure de `tests/` reproduit celle de `src/`, couche par couche — chaque couche a sa classe de base et son propre contrat de vérification.

### 13 — Découpage des tests — miroir de l'architecture

`KISS`

**Principe**
- `tests/Compliance/ApplicationCore/` — Unit — CommandHandlers / QueryHandlers
- `tests/Compliance/InputAdapter/` — Contrôleurs HTTP + EventListeners RabbitMQ
- `tests/Compliance/OutputAdapter/` — Repositories (DB réelle)
- `tests/Compliance/Gateway/` — Gateways externes
- `tests/Helper/Builder|InMemory|Factory/` — Fixtures domain, repos in-memory, factories — `Builder` interdit dans `src/`
- `tests/Integration/` — Workflows complets bout-en-bout

> Le découpage des tests suit exactement Deptrac : si je sais dans quelle couche vit le code, je sais dans quel dossier vit son test, et quelle classe de base utiliser.

### 14 — Quatre classes de base, quatre contrats de test

`SOLID · ISP`

| Base | Étend | Rôle |
|---|---|---|
| `UnitTestCase` | `TestCase` | ApplicationCore — unit pur, pas de kernel, repos in-memory |
| `HttpTestCase` | `WebTestCase` | InputAdapter HTTP — boot kernel, MockQueryBus/MockCommandBus |
| `RabbitMqTestCase` | `KernelTestCase` | InputAdapter EventListeners — simule la réception RabbitMQ |
| `DatabaseTestCase` | `IntegrationTestCase` | OutputAdapter repositories — boot kernel, vraie DB de test |

> Chaque classe de base porte les dépendances minimales nécessaires à sa couche — pas plus. Un test ApplicationCore n'a même pas accès au kernel Symfony, donc impossible d'y introduire une dépendance DB par accident.

### 15 — ApplicationCore — in-memory + Builders, zéro DB

La suite de tests métier tourne sans I/O disque ni réseau. Domain models construits via `Builder`, stockés dans des repos in-memory (`tests/Helper/InMemory/`).
- `tests/Helper/InMemory/AnomalyInMemoryRepository.php, BillingCommentInMemoryRepository.php…` — Implémentent les mêmes interfaces Domain que la version Doctrine, mais en tableau PHP

> Suite ApplicationCore complète en quelques secondes, exécutable en boucle pendant le dev — TDD réel. Aucune dépendance à un état DB partagé donc zéro flakiness inter-tests.

### 16 — InputAdapter — HTTP (MockBus) vs RabbitMQ (round-trip)

**HTTP** — `HttpTestCase` boot le kernel mais remplace les bus réels par `MockQueryBus`/`MockCommandBus` : on vérifie juste que le contrôleur dispatch la bonne Command/Query, pas ce que fait le handler.

**RabbitMQ** — `RabbitMqTestCase` construit une enveloppe brute (`constructRabbitMqMessage`), la fait passer par `receivedRabbitMqEvent()` qui décode → ré-encode → re-décode, puis appelle le listener. Teste aussi les cas de version incompatible.

> Le round-trip détecte une rupture de contrat de sérialisation avec les autres services avant la prod — un champ renommé côté wire format casserait ce test immédiatement, pas un déploiement.

### 17 — OutputAdapter — vraie DB, rollback automatique

Premier test = `SchemaTool` recrée le schéma (pas de migrations en test). Tests suivants = truncate. Chaque test tourne dans une transaction, rollback automatique en `tearDown`.

Pattern : Builder → `save()` via repository → `flush()` puis `clear()` (force le rechargement depuis la DB) → assertion via `XxxAssert::assertEquals()`.

> On teste le vrai mapping Doctrine — colonnes, types, contraintes — sans jamais polluer l'état entre deux tests. Chaque test repart d'une DB propre sans script de reset externe.

### 18 — Convention Given/When/Then + assertException

Chaque test commente ses 3 blocs `// Given / // When / // Then`.
- `tests/Compliance/ApplicationCore/UpdateTollBilledToCustomerAmountCommandHandlerTest.php:81,89,92` — Given / Then / When
- `tests/Helper/UnitTestCase.php:12` — `assertException(Throwable $throwable, int $code)` — vérifie classe + message + code en une seule assertion

> Given/When/Then n'est pas cosmétique — un test qui suit cette structure se lit comme la spec du use case. Un nouveau dev comprend le comportement métier en lisant les tests avant même d'ouvrir le handler.

### 19 — Bénéfices d'ensemble de ce découpage

- **Vitesse** — la couche la plus testée (ApplicationCore) est aussi la plus rapide : pas de DB, pas de kernel.
- **Localisation du bug par la couche qui casse** — test ApplicationCore rouge = bug métier. Test OutputAdapter rouge = bug de mapping Doctrine. Test InputAdapter rouge = bug de contrat HTTP/RabbitMQ.
- **Déterminisme** — Clock/IdGenerator fakes + repos in-memory → zéro flakiness liée au temps ou à un état DB partagé.
- **Zéro nettoyage manuel** — rollback automatique en tearDown pour les tests DB.
- **Détection de rupture de contrat externe avant la prod** — round-trip RabbitMQ + tests de version mismatch.
- **Documentation vivante** — Given/When/Then + Builders lisibles documentent le comportement réel.

---

## Outillage

### 20 — Outils qualité — appliqués, pas juste documentés

`DRY` `KISS`

Ce qui garantit la qualité du code en continu, pas juste à la revue de code.
- `phpstan.neon:23 — level: 8` — Analyse statique au niveau le plus strict
- `rector/rector ^2.4, php-cs-fixer ^3.94.2` — Fix automatique via `make lint` — élimine la réécriture manuelle répétitive (DRY appliqué à l'outillage lui-même)
- `deptrac/deptrac ^4.6 — quality-rules/Deptrac/` — Fait respecter les règles de dépendance hexagonale au moment du build
- `infection/infection ^0.34.0` — Mutation testing — vérifie que les tests détectent vraiment une régression, pas juste qu'ils passent
- `nelmio/api-doc-bundle ^5.9 + zircote/swagger-php ^6.1` — Chaque contrôleur porte des attributs `#[OA\...]` — `make api-doc-validate` fait échouer la CI si la doc ne génère pas, `make api-doc-coverage` si un contrôleur en est dépourvu
- `dependabot` — PR automatiques de mise à jour de dépendances groupées — ex. `chore: bump the all group with 3 updates`

> Deptrac fait respecter l'architecture au build, pas à la revue. Infection va plus loin que la couverture classique : il mute le code pour vérifier que les tests détectent vraiment une régression. La doc API est un artefact vérifié en continu, pas un wiki externe qui se périme. Dependabot garde les dépendances à jour sans intervention manuelle.

---

## Métier

### 21 — Anomalies — Pipeline / Aggregator de règles de conformité

`SOLID · OCP` `SOLID · SRP`

**Architecture** : Interface `RideAuditor`, chaque règle de conformité est une classe indépendante taguée `app.ride_auditor`. Plus proche d'un **Pipeline** ou d'un **Aggregator** que d'un Strategy pur : toutes les règles s'exécutent (pas un choix exclusif d'une seule), chacune produit indépendamment son verdict, et l'orchestrateur agrège les anomalies détectées.

**8 auditors, règle exacte**
- `TotalAmountRideAuditor.php:42-54` — Tolérance : si estimation ≤ 100€ → seuil 10€ ; sinon seuil = 10% du montant estimé. Écart absolu ≥ seuil → anomalie
- `TollAmountRideAuditor.php:39-48` — Tout écart non nul péage transmis vs estimé → anomalie
- `WaitingTimeRideAuditor.php:36-44` — Attente > 10 min ET mode distribution WINBOOK → anomalie
- `OnSpotGpsAuditor.php:39-47` — Écart GPS > 1000 m entre adresse prise en charge et étape "arrivé sur place"
- `DprAuditor.php:26-34` — Course flaggée `isDpr()` → anomalie
- `CodeRideAuditor.php:27-35` — `RideCodeStatus::CODE_NOT_RECEIVED` → anomalie
- `LitigationRideAuditor.php:27-35` — `CustomerBillingStatus::UNPAID` → anomalie
- `EstimateRideAuditor.php:33-40` — Aucun `Estimate` trouvé pour la course → anomalie

> Ce n'est pas un Strategy où un seul candidat est choisi — c'est un pipeline d'auditeurs indépendants, tous exécutés, dont les résultats sont agrégés. Ajouter une règle d'anomalie = ajouter une classe taguée, zéro modif de l'orchestrateur ni des autres règles.

---

## Méthode de révision

- 1 pattern/jour — relire la section, ouvrir le fichier réel cité, vérifier que ça matche toujours le code actuel.
- Ré-expliquer sans notes — à voix haute, comme en entretien, 30 à 60 secondes max.
- Lier les patterns entre eux — CQRS + Repository/Loader + Mediator forment un seul système cohérent, pas 3 concepts isolés. Savoir raconter le flux complet : HTTP → Controller → Bus (Mediator) → Handler → Domain (Factory) → Repository → DB.

**Mock Q&A**
- Pourquoi Repository ET Loader séparés ici ?
- Pourquoi `create()` vs `load()` plutôt qu'un seul constructeur ?
- Que se passe-t-il si un CommandHandler appelle `load()` ?
- Pourquoi injecter Clock/IdGenerator plutôt que les générer dans le domain ?
- Pourquoi les anomalies (§21) ne sont pas un Strategy classique ?

---

*compliance · fiche de révision entretien · généré depuis le code réel du projet*
