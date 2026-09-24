# Swipe CI — Facturation FNE mobile pour micro-entreprises

> Spécification issue du brainstorm du 24/09/2026 (session Cowork), mise à jour avec la procédure d'interfaçage API de la DGI (mai 2025). À affiner dans Claude Code avec `/plan-task` avant d'écrire du code.
> Porteur : [à définir]. Stack : NestJS, Angular, PostgreSQL.

## Initial User Prompt

« Je vais attaquer Swipe avec Claude Code après le brainstorm. » Swipe CI réplique en Côte d'Ivoire le modèle de [Swipe](https://getswipe.in), une startup YC de facturation mobile pour PME indiennes. On l'adapte à la **facture normalisée électronique (FNE)** de la DGI ivoirienne, obligatoire pour les micro-entreprises depuis le 1er août 2025.

### Requirements

#### Décisions validées pendant le brainstorm

| Sujet | Décision |
| --- | --- |
| Cible du MVP | Micro-entreprises au régime RMI (boutiques, artisans, prestataires) |
| Support | App mobile Android (Angular + Capacitor), publiée sur le Play Store |
| Périmètre du MVP | Facture de vente certifiée FNE, avoir FNE, catalogue produits et clients, partage WhatsApp + PDF, suivi des paiements (payé/impayé, lien de paiement, relances) |
| Hors périmètre du MVP | Mode hors ligne, iOS, bot WhatsApp, bordereau d'achat de produits agricoles, facturation en devises (B2F), tableau de bord pour cabinets comptables, branchement sur les API marchands Wave/Orange |
| Monétisation | Freemium + abonnement. Ordre de grandeur : 10 factures par mois gratuites, puis environ 2 000 à 3 000 FCFA par mois (à valider sur le terrain) |
| Prestataire de paiement | **KadevPay** (pas CinetPay), derrière une interface interchangeable |
| Approche technique | Option 1 : app Angular + Capacitor, API NestJS, PostgreSQL multi-tenant, certification FNE côté serveur |

#### Approches écartées (pour mémoire)

- **PWA seule** (proba. 0,82) : écartée, la cible attend une vraie app installable.
- **Socle partagé avec un autre produit** (0,80) : reporté. Swipe CI reste un projet autonome, sans dépendance externe.
- **Facturation par WhatsApp + LLM** (0,08), **branchement Wave/Orange marchands** (0,06), **distribution via cabinets comptables** (0,05) : leviers de croissance pour une V2.

#### Ce que dit la procédure API de la DGI (mai 2025)

**Accès et mise en production**

- **Une clé API par entreprise.** La clé est affichée dans l'onglet « Paramétrage » de l'espace FNE de l'entreprise. Elle n'est visible que par le **gestionnaire principal**, et seulement **après validation par la DGI**.
- **Procédure par entreprise :**
    1. Inscription de l'entreprise sur l'environnement de test (`http://54.247.95.108`).
    2. Développement de l'interfaçage.
    3. Tests de facture de vente, d'avoir et de bordereau.
    4. Envoi de spécimens de factures à `support.fne@dgi.gouv.ci`.
    5. Validation de la conformité par la DGI.
    6. Envoi de l'URL de production, puis affichage de la clé API.
- **Le mode API est « sur option »,** prévu pour les entreprises qui ont leur propre système informatisé de facturation. En face, la DGI propose aussi gratuitement un module web et une **application mobile** pour émettre des FNE.

**Technique**

- **API REST en JSON.** Méthode POST uniquement. Authentification `Authorization: Bearer <API KEY>`.
- **URL de test :** `http://54.247.95.108/ws` (en HTTP, pas HTTPS). L'URL de production est transmise par la DGI.

| Opération | Endpoint | Remarques |
| --- | --- | --- |
| Facture de vente | `POST {url}/external/invoices/sign` avec `invoiceType: "sale"` | Réponse 200 |
| Bordereau d'achat agricole | même endpoint avec `invoiceType: "purchase"` | Hors MVP |
| Avoir | `POST {url}/external/invoices/{id}/refund` | `id` = identifiant FNE de la facture d'origine ; corps = `items[{ id, quantity }]` avec les `id` d'articles renvoyés par la FNE. Réponse 201, référence préfixée `A` |

**Champs de la facture de vente**

- **Obligatoires :** `invoiceType`, `paymentMethod`, `template`, `isRne`, `clientCompanyName`, `clientPhone`, `clientEmail`, `pointOfSale`, `establishment`, `items[]`.
- **Obligatoires selon le cas :** `rne` si `isRne` est vrai ; `clientNcc` si `template = B2B`.
- **Facultatifs :** `clientSellerName`, `commercialMessage`, `footer`, `customTaxes`, `discount`.
- **Par article :** `description`, `quantity`, `amount` (prix unitaire HT) et `taxes[]` sont obligatoires ; `reference`, `discount` et `measurementUnit` sont facultatifs.
- **Valeurs possibles :**
    - `paymentMethod` : `cash`, `card`, `check`, `mobile-money`, `transfer`, `deferred` (à terme).
    - `template` : `B2C` (particulier), `B2B` (entreprise avec NCC), `B2G` (administration), `B2F` (international).
    - `taxes` : `TVA` (18 %), `TVAB` (9 %), `TVAC` (0 %, exonération conventionnelle), `TVAD` (0 %, exonération légale, **pour TEE et RME**).
    - Autres taxes (`customTaxes`, ex. AIRSI, DTD) : nom + taux.

**Réponse**

- `reference` : numéro fiscal, par exemple `9606123E25000000019`, soit le NCC suivi de l'année et d'un numéro séquentiel.
- `token` : URL de vérification à convertir en QR code.
- `warning` : alerte sur le **stock de stickers**.
- `balance_sticker` : nombre de stickers restants.
- `invoice` : la facture complète, avec son `id`, les `id` des articles et le détail des taxes. Il faut les conserver pour les avoirs.

**Erreurs**

- `400` : requête invalide, par exemple `"Point of sale is not valid"`.
- `401` : `"Invalid API Key"`.
- `500` : endpoint non disponible.
- Format : `{ message, error, statusCode }`.

**Ce que le document ne prévoit pas**

- Aucun endpoint de consultation (GET) ni de clé d'idempotence n'est documenté. Si l'appel expire, on ne peut pas savoir si la facture a été certifiée.

#### Conséquences pour le produit

1. **L'onboarding FNE est le vrai goulot.** Chaque commerçant doit avoir un compte FNE et obtenir sa clé API après validation DGI. Swipe CI doit l'accompagner pas à pas : inscription, spécimens, puis saisie de la clé dans l'app. Il faut vérifier si la DGI valide le logiciel une fois pour toutes, ou chaque entreprise (voir questions ouvertes).
2. **Swipe CI est en concurrence avec l'app mobile gratuite de la DGI.** La certification seule ne suffit pas à justifier un abonnement. La valeur vendue, c'est : catalogue en 3 taps, envoi WhatsApp, lien de paiement, suivi des impayés et relances, tableau de bord du CA.
3. **Micro-entreprises = TVAD par défaut.** Le profil fiscal du tenant fixe la taxe par défaut. Un RME facture en `TVAD` (0 %), ce qui simplifie les calculs du MVP.
4. **Stickers.** Chaque certification consomme un sticker. On affiche `balance_sticker` et on alerte le commerçant dès que `warning` est vrai.
5. **Impayés.** Une facture émise non payée part avec `paymentMethod: "deferred"`. Une facture payée comptant part avec `cash` ou `mobile-money`.
6. **Client comptoir.** `clientPhone` et `clientEmail` sont obligatoires, même en B2C. L'app doit prévoir une saisie rapide (téléphone du client) et une règle claire quand le client n'a pas d'e-mail. Cette règle est à confirmer avec la DGI.

#### Architecture cible

**1. App mobile (Angular + Capacitor, Android)**

- **Écrans :**
    - connexion OTP par SMS ;
    - **assistant de connexion FNE** : guide d'inscription et saisie de la clé API ;
    - tableau de bord (CA du mois, impayés, stickers restants) ;
    - nouvelle facture, liste des factures, détail de facture (avec bouton « Faire un avoir ») ;
    - catalogue, clients, abonnement ;
    - paramètres (NCC, raison sociale, point de vente, établissement, taxe par défaut, logo).
- Pas de logique fiscale côté app : elle appelle l'API.
- Cache en lecture seule (catalogue, 50 dernières factures) pour l'affichage quand le réseau est lent. Aucune création hors ligne.
- Partage natif (plugin Capacitor Share) du PDF et du lien de la facture vers WhatsApp.

**2. API NestJS — modules**

| Module | Rôle |
| --- | --- |
| `auth` | OTP SMS, JWT court + refresh token, un utilisateur appartient à un tenant |
| `tenants` | Une micro-entreprise = un tenant : NCC, régime fiscal, taxe par défaut, raison sociale, `pointOfSale`, `establishment`, logo, clé API FNE chiffrée, environnement FNE (test / prod) |
| `catalogue` | Articles (désignation, prix unitaire HT, taxe FNE, unité de mesure, référence) |
| `clients` | Clients du commerçant (nom, téléphone, e-mail, NCC si B2B, type B2C/B2B/B2G) |
| `invoices` | Machine d'états (voir plus bas), calcul des totaux, avoirs |
| `fne` | **Seul module qui parle à la DGI.** `FneGateway` : `signSale()`, `refund()`. Mapping vers le format FNE, stockage de la réponse brute, des `id` FNE (facture et articles), de la `reference`, du `token` et de `balance_sticker` |
| `pdf` | PDF de la facture certifiée avec QR code (généré depuis `token`), numéro fiscal et visuel FNE ; stockage S3 |
| `payments` | Interface `PaymentProvider` ; implémentation `KadevPayProvider` ; lien de paiement par facture ; webhook → facture « payée » |
| `billing` | Abonnement Swipe du commerçant (plans, quotas, facturation via KadevPay, webhook → activation du plan) |
| `reminders` | Relances d'impayés (lien WhatsApp pré-rempli dans le MVP) |
| `jobs` | BullMQ + Redis : certification FNE, génération PDF, relances |

**3. Données (PostgreSQL)**

- Multi-tenant par colonne `tenant_id` + Row Level Security.
- Tables principales : `tenants`, `users`, `products`, `customers`, `invoices`, `invoice_lines`, `fne_certifications` (réponse brute, `fne_invoice_id`, `reference`, `token`, `balance_sticker`), `fne_invoice_items` (lien ligne locale ↔ `id` article FNE), `credit_notes`, `payment_links`, `payments`, `subscriptions`, `usage_counters`, `audit_logs`.
- Montants en entiers (FCFA). Les totaux affichés après certification viennent de la réponse FNE (`amount`, `vatAmount`), pas d'un recalcul local.
- Clé API FNE chiffrée au repos (clé de chiffrement hors base). Elle n'est jamais renvoyée à l'app.
- Une facture certifiée ne se modifie jamais : toute correction passe par un avoir FNE, puis une nouvelle facture.

#### Flux principaux

**Connecter le commerçant à la FNE**

1. L'assistant explique les étapes DGI et renvoie vers l'inscription.
2. Le commerçant saisit sa clé API et choisit l'environnement (test pendant la validation, prod ensuite). La clé est chiffrée côté serveur.
3. L'API fait une certification de test en environnement de test pour vérifier la clé, puis propose d'exporter les spécimens à envoyer à `support.fne@dgi.gouv.ci`.

**Créer et certifier une facture**

1. Le commerçant choisit le client et les articles, indique si la facture est payée (cash / mobile money) ou à terme (`deferred`), puis valide. L'API crée la facture au statut `à_certifier` et vérifie le quota du plan.
2. Le job `fne.certify` appelle `POST /external/invoices/sign`.
3. En cas de 200 : `reference`, `token`, `id` FNE et `id` des articles sont enregistrés, statut `certifiée`, puis job `pdf.generate`. Si `warning` est vrai, alerte « stickers bientôt épuisés ».
4. L'app reçoit la mise à jour (polling court ou notification push) et propose « Partager sur WhatsApp ».

**Faire un avoir**

1. Depuis une facture certifiée, le commerçant choisit les articles et les quantités à reprendre.
2. `POST /external/invoices/{id FNE}/refund` avec les `id` d'articles FNE enregistrés. En cas de 201, on enregistre la référence de l'avoir (préfixe `A`) et son `token`, puis on génère le PDF.

**Encaisser une facture**

1. Pour une facture certifiée à terme, l'API crée un lien de paiement via `PaymentProvider` (KadevPay).
2. Le lien est inclus dans le message WhatsApp.
3. Le webhook KadevPay (signature vérifiée, traitement idempotent) enregistre le paiement et passe la facture en `payée`. Le commerçant peut aussi marquer « payée en espèces ».

**Payer l'abonnement Swipe**

1. Quand le quota gratuit est atteint, l'app propose un plan.
2. Paiement via KadevPay, le webhook active ou prolonge le plan dans `subscriptions`.

#### Machine d'états d'une facture

```mermaid
stateDiagram-v2
    [*] --> a_certifier
    a_certifier --> certifiee : 200 FNE
    a_certifier --> rejetee : 400 FNE
    a_certifier --> a_verifier : timeout / réponse perdue
    rejetee --> a_certifier : correction
    a_verifier --> certifiee : vérifiée manuellement
    a_verifier --> a_certifier : confirmée non certifiée
    certifiee --> payee : paiement reçu
    certifiee --> avoir_partiel : avoir partiel
    certifiee --> annulee : avoir total
```

#### Gestion des erreurs

- **Pas d'idempotence côté FNE : risque de double certification.** On ne réessaie automatiquement **que** si l'échec est certain avant envoi (connexion refusée, DNS) ou en cas de `500 Endpoint non disponible`. En cas de timeout après envoi, la facture passe en `à_vérifier` : l'admin ou le commerçant vérifie dans l'espace FNE avant tout renvoi. Aucun renvoi automatique dans ce cas.
- **400 (données invalides)** : statut `rejetée`, `message` FNE traduit en français simple (ex. « Point de vente non valide » → lien vers les paramètres), facture corrigeable puis renvoyée.
- **401 (clé invalide)** : blocage des certifications du tenant, notification « Reconnectez votre compte FNE ».
- **Stickers épuisés** : message clair et lien vers la procédure DGI de rechargement (à documenter).
- **Webhook KadevPay** : vérification de signature, rejeu toléré, réconciliation quotidienne avec l'API KadevPay.
- **Quota Swipe dépassé** : blocage de la certification uniquement ; consultation et partage restent possibles.
- **Journal d'audit** de toutes les actions fiscales (création, certification, avoir, vérification manuelle).

#### Tests

- **Unitaires** : mapping vers le format FNE (champs obligatoires, B2B ⇒ `clientNcc`, taxe par défaut `TVAD` pour RME), machine d'états, quotas.
- **Contrat FNE** : `FneGateway` testé contre un faux serveur reproduisant les exemples du PDF (200 vente, 201 avoir, 400, 401, 500, timeout), puis contre l'environnement de test DGI `http://54.247.95.108/ws`.
- **Avoirs** : les `id` d'articles FNE sont bien conservés et renvoyés.
- **Paiements** : `KadevPayProvider` testé avec webhooks simulés (signature valide/invalide, doublons).
- **Intégration** : isolation multi-tenant (tests RLS), clé API jamais exposée par l'API publique.
- **E2E mobile** : parcours « connecter la FNE → créer → certifier → partager → marquer payée → avoir » sur émulateur Android.

#### Questions ouvertes (à trancher avant ou pendant `/plan-task`)

1. **Validation DGI :** la DGI valide-t-elle chaque entreprise ou le logiciel une fois pour toutes ? Un éditeur peut-il faire valider Swipe CI une seule fois, pour que ses clients activent ensuite leur clé plus vite ? À demander à `support.fne@dgi.gouv.ci`.
2. **Micro-entreprises et API :** un RME peut-il opter pour le mode API alors que la procédure vise les entreprises « disposant de leur propre système informatisé » ? À confirmer avec la DGI.
3. **Stickers :** comment sont-ils obtenus et rechargés (achat, dotation) et à quel coût pour le commerçant ?
4. **Vérification d'une facture après timeout :** existe-t-il un endpoint de consultation non documenté ? Sinon, comment vérifier dans l'espace FNE ?
5. **Client comptoir :** quelles valeurs accepter pour `clientPhone` et `clientEmail` quand le client est un particulier anonyme ?
6. **Remises :** `discount` est-il un pourcentage ? Les exemples le suggèrent, à vérifier en environnement de test.
7. **Architecture :** présentée mais pas encore validée explicitement.
8. **KadevPay** ([pay.kadev.ci](https://pay.kadev.ci/)) **:** documentation de l'API (liens de paiement, webhooks, signature), frais, moyens de paiement couverts (Wave, Orange Money, MTN, Moov).
9. **Prix de l'abonnement** et nombre de factures gratuites : à tester avec 20 micro-entreprises, face à l'app gratuite de la DGI.
10. **OTP SMS :** prestataire et coût, à choisir plus tard (hors priorité actuelle).

#### Critères de succès du MVP

- Un commerçant déjà muni de sa clé API FNE crée et certifie sa première facture en moins de 2 minutes.
- 20 micro-entreprises abonnées payantes à Abidjan.
- Taux de certification FNE réussie supérieur à 98 % (hors indisponibilité DGI), et zéro double certification.

## Description

> Plan d'implémentation rédigé le 24/09/2026. Les hypothèses marquées **(H)** remplacent provisoirement les questions ouvertes et sont à confirmer.

### Hypothèses de travail

- **(H1) Architecture validée** (question 7) : monorepo pnpm avec `apps/api` (NestJS), `apps/mobile` (Angular + Capacitor) et `packages/shared` (types DTO, enums FNE). PostgreSQL 16, Redis 7, stockage compatible S3 (MinIO en local). ORM : Prisma, avec des migrations SQL brutes pour la RLS.
- **(H2) Validation DGI par entreprise** (question 1) : l'assistant FNE couvre toute la procédure pour chaque commerçant.
- **(H3) Client comptoir** (question 5) : si le client est anonyme, on envoie le téléphone du commerçant et une adresse e-mail technique par tenant (`comptoir+<tenant>@swipe.ci`). Cette valeur est paramétrable pour pouvoir changer la règle sans redéployer.
- **(H4) Remise** (question 6) : on la traite comme un pourcentage (entier de 0 à 100) et on la vérifie en environnement de test avant la production.
- **(H5) KadevPay** (question 8, [pay.kadev.ci](https://pay.kadev.ci/)) : tant que sa documentation n'est pas reçue, on s'appuie sur `PaymentProvider` avec un `FakePaymentProvider`. `KadevPayProvider` est un lot séparé, bloqué par cette documentation.
- **(H6) OTP SMS** (question 10) : interface `SmsSender`. En dev, les codes sont écrits dans les logs. Le choix du prestataire est reporté.
- **(H7) Vérification après timeout** (question 4) : manuelle (écran admin « à vérifier »), sans endpoint GET.

### Structure du dépôt

```
apps/api/        NestJS (modules de la spec)
apps/mobile/     Angular + Ionic + Capacitor (Android)
packages/shared/ types, enums FNE, schémas de validation (zod)
tools/fne-mock/  faux serveur FNE (tests de contrat)
docker-compose.yml  postgres, redis, minio, fne-mock
```

### Lots (dans l'ordre)

Chaque lot est livrable seul, testé et fusionné avant de passer au suivant.

| # | Lot | Contenu | Dépend de | Terminé quand |
| --- | --- | --- | --- | --- |
| 0 | Socle | Monorepo pnpm, lint et format, CI GitHub Actions (lint, tests, build), docker-compose, config typée, logs structurés | aucun | la CI est verte et `docker compose up` démarre tout |
| 1 | Données et multi-tenant | Schéma Prisma de toutes les tables, montants en entiers, politiques RLS par `tenant_id` (`SET app.tenant_id` par requête), `audit_logs` | 0 | les tests RLS montrent qu'un tenant ne lit jamais les données d'un autre |
| 2 | Auth | OTP SMS (`SmsSender`), JWT court + refresh token rotatif, garde de tenant, limitation de débit sur l'OTP | 1 | e2e : login, refresh, révocation |
| 3 | Tenants et paramètres | Profil fiscal (NCC, régime, taxe par défaut `TVAD` pour RME, point de vente, établissement, logo), clé API FNE chiffrée (AES-256-GCM, clé hors base), environnement test/prod | 2 | la clé n'est jamais renvoyée (test d'intégration) |
| 4 | Catalogue et clients | CRUD produits (taxe FNE, unité, référence) et clients (type B2C/B2B/B2G, NCC obligatoire en B2B) | 3 | tests unitaires et e2e |
| 5 | Passerelle FNE | `tools/fne-mock` (200, 201, 400, 401, 500, timeout), `FneGateway.signSale()` / `refund()`, mapping pur facture → JSON FNE, classement des erreurs (échec certain avant envoi / 400 / 401 / 500 / timeout ambigu) | 1, 3 | tests de contrat verts contre le mock, et un appel manuel réussi sur `http://54.247.95.108/ws` |
| 6 | Factures et certification | Machine d'états (fonction pure et table de transitions), calcul des totaux avant certification, quota, job BullMQ `fne.certify` (réessais uniquement pour un échec certain ou un 500, sinon `a_verifier`), enregistrement de `fne_certifications` et `fne_invoice_items`, alerte stickers, blocage du tenant sur 401, audit | 4, 5 | aucune double certification simulée (tests timeout + réessai) et 100 % des transitions testées |
| 7 | Avoirs | Choix des articles et quantités, `refund` avec les `id` FNE, états `avoir_partiel` et `annulee`, contrôle des quantités restantes | 6 | tests des cas partiel, total et dépassement |
| 8 | PDF et partage | Job `pdf.generate` (QR depuis `token`, numéro fiscal, visuel FNE), stockage S3, URL signée, lien public de consultation | 6 | PDF de référence validé à l'œil, puis tests de snapshot |
| 9 | App mobile, parcours cœur | Login OTP, assistant FNE, paramètres, catalogue, clients, nouvelle facture (3 taps), liste et détail, polling du statut, partage WhatsApp (Capacitor Share), cache en lecture (catalogue et 50 dernières factures) | 2 à 8 | le parcours prend moins de 2 minutes sur un émulateur |
| 10 | Paiements | `PaymentProvider`, lien de paiement, webhook idempotent (signature, table des événements traités), « payée en espèces », réconciliation quotidienne | 6, puis H5 pour KadevPay | tests des webhooks (signature valide ou invalide, doublons) |
| 11 | Relances et tableau de bord | Relances (lien WhatsApp pré-rempli), CA du mois, impayés, stickers restants | 9, 10 | e2e |
| 12 | Abonnement | Plans, `usage_counters` (10 factures gratuites par mois, valeur paramétrable), paiement de l'abonnement, webhook d'activation, blocage limité à la certification | 10 | tests des quotas |
| 13 | Admin et exploitation | Écran admin des factures `a_verifier` (confirmer certifiée ou non certifiée), traduction des erreurs FNE, supervision des jobs (Bull Board), sauvegardes | 6 | revue de la procédure d'exploitation |
| 14 | Mise en production | E2E Android complet, export des spécimens pour la DGI, build Play Store (signature, fiche), environnement de prod | tous | premier commerçant pilote en prod |

**Chemin critique :** 0 → 1 → 3 → 5 → 6 → 9. La passerelle FNE (lot 5) est le lot le plus risqué, on la commence donc tôt, dès que les lots 1 et 3 sont prêts.

### Points de conception clés

- **Anti-double certification :** avant chaque appel, on insère une ligne `fne_certifications` à l'état `pending` (clé unique `invoice_id`, verrou en base). Le worker ne renvoie jamais la requête si cette ligne est dans un état ambigu. Seul un admin (ou le commerçant) peut la débloquer, et l'action est auditée.
- **Classement des erreurs réseau :** `ECONNREFUSED`, `ENOTFOUND` et une erreur TLS avant envoi comptent comme des échecs certains. Un timeout de lecture ou un socket fermé après écriture sont ambigus.
- **Montants :** entiers en FCFA partout. Après certification, l'affichage utilise les montants renvoyés par la FNE.
- **Immutabilité :** une facture certifiée est en lecture seule, garanti par un trigger SQL qui refuse toute modification des colonnes fiscales.
- **Secrets :** la clé de chiffrement est dans une variable d'environnement ou un gestionnaire de secrets. On prévoit sa rotation avec un champ `key_version`.

### Actions externes à lancer tout de suite (en parallèle du code)

1. Écrire à `support.fne@dgi.gouv.ci` au sujet des questions 1, 2, 4 et 5, et inscrire l'entreprise porteuse sur l'environnement de test.
2. Demander à KadevPay sa documentation d'API et ses frais (question 8).
3. Recruter les 20 micro-entreprises pilotes pour tester le prix (question 9).
