# Articles Medium — Projet Caresse

> Source : analyse des repos `caresse-app` (KMP), `caresse-api` (.NET 10), `assets` (landing page).
> Objectif : articles tech à approfondir, rédigés pour une audience dev sur Medium.

---

## Architecture & Backend

### 1. Hexagonal Architecture in .NET 10: How We Kept Our AI Pipeline Maintainable

**Accroche** : La plupart des articles sur l'architecture hexagonale s'arrêtent à la théorie. Celui-ci montre comment l'implémenter dans une vraie API .NET 10 qui orchestre plusieurs fournisseurs LLM et TTS — sans jamais toucher au domaine quand on change de provider.

**Points clés à couvrir** :
- Structure concrète : `Domain/Ports/Driving`, `Domain/Ports/Driven`, `DrivenAdapters/`, `DrivingAdapters/`
- Convention de nommage : `*RestAdapter`, `I*Port`, `*UseCase`
- Pattern registre de providers : `Dictionary<AiProvider, IPromptExecutorPort>` et `Dictionary<TtsProvider, ITtsExecutorPort>`
- Comment ajouter un nouveau provider sans modifier le domaine
- Comparaison avec l'approche classique "tout dans le contrôleur"

**Ce qui est rare** : exemple complet en .NET avec plusieurs providers IA branchés via ports & adapters — pas un exemple jouet.

**Mots-clés SEO** : hexagonal architecture .NET, ports and adapters C#, clean architecture .NET 10, DDD .NET

---

### 2. Orchestrating a Multi-Phase LLM Pipeline: 8 Stages from Prompt to Audio

**Accroche** : Un seul appel LLM ne suffit pas pour générer une expérience audio immersive. Voici comment nous avons découpé la génération en 8 phases séquentielles, chacune avec son propre modèle, ses propres prompts, et sa logique de fallback.

**Points clés à couvrir** :
- Les 8 phases : Setup → Immersion → FirstTouches → ExplorationAndForeplay → IntimacyCrescendo → Improver → Summary → TextToSpeech
- Pourquoi chaque phase peut tourner sur un modèle différent (coût vs qualité)
- Pattern Strategy pour les prompt builders (couple vs solo)
- Gestion des erreurs inter-phases et stratégie de retry
- Pourquoi un seul gros prompt échoue là où 8 petits réussissent

**Ce qui est rare** : architecture de pipeline LLM multi-phases en production, avec modèle différent par phase — très peu documenté.

**Mots-clés SEO** : LLM pipeline orchestration, multi-phase prompt engineering, .NET AI pipeline, LLM production architecture

---

### 3. Testing .NET APIs Without Touching Production: Testcontainers + WireMock + Fake Auth

**Accroche** : Tests d'intégration avec une vraie base PostgreSQL, Azure Storage émulé, Firebase remplacé par un handler custom, et RevenueCat mocké avec WireMock.Net. Zéro mocks inutiles, zéro dépendance à l'environnement de prod.

**Points clés à couvrir** :
- Setup Testcontainers pour PostgreSQL 16 en tests
- Azurite comme émulateur Azure Blob Storage
- `TestAuthHandler` pour bypasser Firebase JWT sans toucher le code métier
- WireMock.Net pour simuler l'API RevenueCat
- Organisation des tests : convention `Method_Condition_ShouldBehavior`
- Comparaison temps d'exécution vs bénéfices de fiabilité

**Ce qui est rare** : combinaison des 4 outils simultanément dans un seul projet .NET documentée de bout en bout.

**Mots-clés SEO** : Testcontainers .NET, integration testing .NET, WireMock .NET, xUnit integration tests, ASP.NET Core testing

---

### 4. Multi-Provider AI: Never Get Locked to a Single LLM Vendor Again

**Accroche** : OpenAI coupe l'accès, les prix doublent, un nouveau modèle sort. Si ton architecture est couplée à un provider, tu es bloqué. Voici comment nous avons intégré Grok, DeepSeek, OpenRouter et d'autres derrière une interface unique.

**Points clés à couvrir** :
- Interface `IPromptExecutorPort` et ses implémentations
- Sélection du provider au runtime (config, feature flag, par phase)
- Gestion des différences d'API (streaming, tokens, formats de réponse)
- Tests unitaires provider-agnostiques avec NSubstitute
- Critères de choix de provider par phase (latence, coût, qualité)

**Ce qui est rare** : implémentation concrète multi-LLM en .NET avec tests, pas juste une liste de providers.

**Mots-clés SEO** : multi LLM provider .NET, LLM vendor lock-in, OpenRouter integration .NET, strategy pattern LLM

---

### 5. PostgreSQL JSONB with EF Core: Flexible Schemas Without Sacrificing Type Safety

**Accroche** : Le profil intime d'un utilisateur évolue en permanence — nouvelles pratiques, nouveaux paramètres. Plutôt qu'une migration à chaque ajout, nous stockons `IntimateProfile` en JSONB PostgreSQL tout en conservant la sérialisation C# typée via EF Core.

**Points clés à couvrir** :
- Mapping JSONB avec Npgsql EF Core et snake_case naming convention
- Sérialisation/désérialisation typée du JSONB en C#
- Quand choisir JSONB vs colonnes relationnelles classiques
- Migrations EF Core avec des colonnes JSONB
- Performances et indexation JSONB en PostgreSQL 16

**Ce qui est rare** : guide pratique JSONB + EF Core en production, avec les pièges réels rencontrés.

**Mots-clés SEO** : PostgreSQL JSONB EF Core, Entity Framework JSONB, Npgsql JSONB mapping, flexible schema .NET

---

## Mobile & Cross-Platform

### 6. Kotlin Multiplatform in Production: What Nobody Tells You

**Accroche** : KMP promet "write once, run anywhere". La réalité est plus nuancée. Après avoir livré une app en production sur Android et iOS, voici ce qui fonctionne vraiment, ce qui reste douloureux, et ce qu'on aurait fait différemment.

**Points clés à couvrir** :
- Ce qui marche sans friction : Ktor, Koin, kotlinx-serialization, Compose Multiplatform
- Ce qui est encore douloureux : gestion audio iOS vs Android (Media3 vs AVPlayer), expect/actual verbeux
- Firebase multiplatform via GitLive : état réel de la lib
- Structure `commonMain` / `androidMain` / `iosMain` : comment décider où mettre quoi
- Retour honnête sur la maturité de l'écosystème en 2025

**Ce qui est rare** : REX honnête KMP en production, pas un tutorial "hello world".

**Mots-clés SEO** : Kotlin Multiplatform production, KMP iOS Android, Compose Multiplatform review, KMM 2025

---

### 7. Multi-Environment Setup in KMP: BuildKonfig, Flavors, and Firebase Switching

**Accroche** : Gérer Debug, UAT et Production dans une app Kotlin Multiplatform sans leaker les clés API dans les builds publics. Un setup concret avec BuildKonfig, product flavors Gradle et un script de switch Firebase.

**Points clés à couvrir** :
- BuildKonfig : configuration build-time multiplatform (API URLs, feature flags, clés)
- Product flavors `uat` / `prd` dans Gradle KMP
- Script bash pour switcher `google-services.json` selon l'environnement
- Sécurisation des secrets : ce qui ne doit jamais aller dans le repo
- Mode mock API (`useMockApi`) pour le dev local sans backend

**Ce qui est rare** : BuildKonfig est peu documenté, la combinaison avec flavors KMP l'est encore moins.

**Mots-clés SEO** : BuildKonfig KMP, Kotlin Multiplatform environments, KMP secrets management, gradle product flavors KMP

---

### 8. Cross-Platform In-App Purchases with RevenueCat and Kotlin Multiplatform

**Accroche** : Un SDK, deux plateformes, un backend de vérification. Voici l'architecture complète pour intégrer RevenueCat dans une app KMP — du mobile jusqu'à la vérification côté serveur en .NET.

**Points clés à couvrir** :
- Intégration RevenueCat SDK 2.x dans un projet KMP
- Gestion des `expect/actual` pour les appels natifs
- Vérification des receipts côté serveur (.NET + RevenueCat API)
- Webhooks RevenueCat pour synchroniser l'état des abonnements
- Tests : WireMock pour simuler l'API RevenueCat, Testcontainers pour la DB

**Ce qui est rare** : guide RevenueCat KMP + backend .NET — aucun article existant sur cette combinaison.

**Mots-clés SEO** : RevenueCat Kotlin Multiplatform, KMP in-app purchases, RevenueCat .NET backend, cross-platform subscriptions

---

### 9. Compose Multiplatform: Sharing 95% of Your UI Between Android and iOS

**Accroche** : Compose Multiplatform permet de partager la quasi-totalité de l'UI. Mais les 5% restants — animations spécifiques, comportements natifs, gestion du clavier — sont ceux qui font la différence entre une app qui "ressemble à du web" et une vraie app native.

**Points clés à couvrir** :
- Ce qui est partagé : Material3, navigation, ViewModels, Lottie (Compottie)
- Ce qui ne peut pas l'être : audio (Media3 vs AVPlayer), certains comportements tactiles
- Système d'icônes custom partagé
- Gestion du theming et des animations Lottie multiplatform
- Performances Compose sur iOS vs Android en 2025

**Ce qui est rare** : comparaison concrète de ce qui est partageable ou non en Compose Multiplatform, avec code.

**Mots-clés SEO** : Compose Multiplatform iOS Android, shared UI KMP, Compose iOS performance, Compottie Lottie KMP

---

## Audio & IA Générative

### 10. From Text to Immersive Audio: LLM + TTS + FFmpeg in a .NET Pipeline

**Accroche** : Comment un scénario textuel généré par LLM devient un fichier audio multi-voix avec musique d'ambiance — de la génération des phrases à l'export final via FFmpegCore, en passant par ElevenLabs et Cartesia pour les voix.

**Points clés à couvrir** :
- Architecture du pipeline : génération texte → TTS par `Sentence` → mixage audio
- Multi-provider TTS : ElevenLabs, Cartesia, Murf, Astica, Minimax — comment choisir
- FFmpegCore : mixage des voix avec les musiques d'ambiance (cozy, nature, torrid, zen)
- Analyse des waveforms avec NLayer pour la synchronisation
- Gestion des erreurs dans un pipeline audio asynchrone

**Ce qui est rare** : pipeline complet text-to-audio en .NET de bout en bout, pas juste un appel TTS.

**Mots-clés SEO** : .NET audio pipeline, FFmpegCore .NET, ElevenLabs .NET integration, TTS pipeline .NET, text to audio C#

---

### 11. Choosing the Right TTS Provider for Your Product: A Technical Comparison

**Accroche** : Nous avons intégré 6 providers TTS (ElevenLabs, Cartesia, Murf, Astica, Minimax, OpenAI) dans la même interface. Voici ce que les benchmarks ne disent pas : latence réelle, qualité par langue, coût par minute, et fiabilité en production.

**Points clés à couvrir** :
- Tableau comparatif : latence, qualité, langues supportées, prix, API design
- Cas d'usage où chaque provider gagne
- Pattern d'abstraction pour rester provider-agnostique
- Gestion du streaming audio vs génération complète
- Retour sur la qualité du français (souvent négligée dans les benchmarks)

**Ce qui est rare** : comparaison technique multi-provider TTS avec retour d'expérience production, focus francophone.

**Mots-clés SEO** : TTS provider comparison, ElevenLabs vs Cartesia, text to speech API comparison, TTS production

---

### 12. Prompt Engineering for Narrative Generation: Lessons from 8 LLM Phases

**Accroche** : Générer des récits immersifs avec un LLM demande bien plus qu'un prompt bien tourné. Voici les patterns de prompt engineering que nous avons appris à la dure sur 8 phases de génération, avec des exemples concrets.

**Points clés à couvrir** :
- Pourquoi le découpage en phases améliore la cohérence narrative
- Techniques de chaînage de contexte entre phases
- Prompt builders dynamiques : couple vs solo, langue, niveau de language
- Phase "Improver" : faire critiquer et améliorer par le modèle lui-même
- Gestion de la longueur et du format de sortie pour la TTS

**Ce qui est rare** : prompt engineering appliqué à la génération narrative longue, pas au Q&A ou au code.

**Mots-clés SEO** : prompt engineering narrative, LLM story generation, multi-phase prompt chaining, generative AI storytelling

---

## Stratégie & Méta

### 13. Building a Sensitive-Category App: Privacy, App Store, and SEO Strategies

**Accroche** : Les règles du jeu changent quand ton app est dans une catégorie sensible. App Store Review, SEO sans pénalité, RGPD, screenshots localisés — voici ce qu'on a appris à la dure pour qu'une app intime soit acceptée, trouvée, et utilisée.

**Points clés à couvrir** :
- App Store / Google Play : règles spécifiques aux contenus adultes, stratégie de soumission
- SEO multilingue (fr, en, es, pt) avec hreflang pour un sujet sensible
- RGPD : collecte minimale, consentement, mentions légales
- Screenshots localisés : 22 variantes par langue, impact sur la conversion
- JSON-LD schema.org `MobileApplication` pour le référencement

**Ce qui est rare** : guide technique + stratégique pour une app de contenu sensible — presque personne ne publie là-dessus.

**Mots-clés SEO** : app store sensitive content, adult app SEO, GDPR mobile app, multilingual app store optimization

---

### 14. From Monolith to Multi-Repo: Organizing a KMP + .NET Product

**Accroche** : Trois repos, une app. Comment nous organisons le code, les dépendances, et les releases entre `caresse-app` (KMP), `caresse-api` (.NET) et les assets marketing — et ce qu'on changerait aujourd'hui.

**Points clés à couvrir** :
- Découpage multi-repo vs monorepo : nos critères de décision
- Synchronisation des versions d'API entre mobile et backend
- Gestion des environnements cohérente cross-repo (UAT/PRD)
- CI/CD cross-repo : comment un changement API déclenche les tests mobile
- Retour honnête : ce qui freine, ce qu'on garderait

**Ce qui est rare** : retour d'expérience multi-repo sur un vrai produit KMP + .NET, pas une opinion abstraite.

**Mots-clés SEO** : multi-repo architecture, KMP .NET product architecture, mobile backend organization, cross-repo CI/CD

---

### 15. Building a Bootstrapped AI Product in 2025: Tech Choices That Saved Us Time

**Accroche** : Chaque décision technique dans un projet bootstrappé a un coût direct. Voici les choix qui nous ont fait gagner du temps (KMP, hexagonale, RevenueCat), ceux qui nous en ont coûté, et ce qu'on ferait différemment avec le recul.

**Points clés à couvrir** :
- Pourquoi KMP plutôt que Flutter ou React Native pour une équipe C#/Kotlin
- .NET 10 en prod : maturité réelle, avantages de performance
- RevenueCat vs implementation custom des IAP : le vrai coût comparé
- Multi-provider LLM dès le départ : overhead ou investissement rentable ?
- Ce qu'on aurait externalisé plus tôt (audio processing, analytics)

**Ce qui est rare** : analyse tech choices vs contraintes bootstrapped — le pragmatisme souvent absent des articles "best practices".

**Mots-clés SEO** : bootstrapped AI startup tech stack, KMP vs Flutter 2025, .NET 10 production, indie developer tech choices

---

## Ordre de publication suggéré

| Priorité | Article | Raison |
|----------|---------|--------|
| 1 | #6 — KMP in Production | Forte demande, peu d'offre, audience large |
| 2 | #2 — Multi-Phase LLM Pipeline | Tendance AI, très concret, différenciant |
| 3 | #1 — Hexagonal Architecture .NET | Audience .NET large, bon SEO |
| 4 | #10 — Text to Audio Pipeline | Niche porteur, peu de concurrence |
| 5 | #8 — RevenueCat + KMP | Article inexistant sur cette combo |
| 6 | #7 — Multi-Environment KMP | Utile, BuildKonfig sous-documenté |
| 7 | #13 — Sensitive Category App | Angle différenciant fort, courageux |
| 8 | #3 — Testing .NET sans prod | Audience .NET, pratique immédiate |
| 9 | #15 — Bootstrapped AI 2025 | Large audience, personnel et crédible |
| 10 | #11 — TTS Comparison | Utile mais plus benchmarks que récit |

---

## Stratégie public vs premium (member-only)

**Règle** : 1 article public tous les 3-4 articles dans la série, le reste en premium.

| | Public (free) | Member-only (premium) |
|---|---|---|
| Revenu direct | $0 | ~$0.01–0.05 / min lue par membre |
| Audience | Tout internet + Google + Reddit + HN | Membres Medium uniquement |
| SEO Google | ✅ indexation complète | ❌ partielle |
| Partage Reddit / HN | Illimité | 1 friend link only |
| Conversion vers caresse.app | Audience large = + clics absolus | Audience qualifiée mais petite |
| Followers Medium gagnés | **Élevé** (non-membres peuvent follow) | Modéré |
| Commission referral Medium | ✅ levier actif | ❌ levier mort |

### Plan de publication recommandé

```
#1 Hexagonal              → PUBLIC   → SEO + Reddit/HN + funnel followers + clics caresse.app
#2 Multi-stage pipeline   → PREMIUM  → revenu sur audience acquise par #1
#3 Testing .NET           → PREMIUM
#4 Multi-LLM              → PUBLIC   → nouveau coup de boost SEO/funnel
#5 JSONB EF Core          → PREMIUM
#6 KMP in production      → PUBLIC   → audience large, sujet partageable
#7 RevenueCat KMP         → PREMIUM
...
```

### Pourquoi le #1 d'une nouvelle série doit être public

1. **Pas encore d'audience sur ce thème** — les followers de "the right way" sont .NET pur, Building Caresse est un nouveau territoire. Il faut prouver la valeur avant de paywaller.
2. **Funnel vers les #2+ premium** — un lecteur gratuit qui aime le #1 te follow, reçoit la notif du #2 (premium), et peut s'abonner à Medium pour le lire (→ commission referral à vie).
3. **Boost Medium plus probable** — l'algo diffuse plus volontiers les articles publics, le boost retombe ensuite sur tes premium via tes nouveaux followers.
4. **Partageable sans friction** — pas de paywall qui fait fermer l'onglet sur Reddit / HN.

---

## Tuto — Importer un article Markdown sur Medium

Medium n'accepte pas le Markdown brut. Trois méthodes possibles, par ordre de qualité du rendu.

### Méthode 1 — Import par URL (recommandée) ⭐

Préserve le mieux le formatage (titres, listes, code blocks, liens).

**Étapes** :
1. Publier l'article en HTML public quelque part. Le plus simple : **[hackmd.io](https://hackmd.io)** (gratuit, rend le Markdown en HTML propre).
   - Créer un compte → nouvelle note → coller le Markdown (sans le frontmatter YAML).
   - Cliquer **Publish** en haut à droite → récupérer l'URL publique.
2. Sur Medium : photo de profil → **Stories** → **Import a story**.
3. Coller l'URL → Medium scrape le HTML et crée un draft.
4. Supprimer la note HackMD une fois l'import vérifié.

**Conservé** : titres, paragraphes, listes, code blocks (avec coloration), images, liens.
**Perdu** : frontmatter YAML, `<!--more-->`, éléments custom.

### Méthode 2 — Copier-coller depuis le rendu Markdown

Plus rapide, mais retouches manuelles nécessaires.

1. Ouvrir le `.md` dans **VS Code preview**, **Typora**, **Obsidian**, ou **github.com**.
2. Sélectionner tout le rendu → copier → coller dans Medium (Write a story).
3. Recréer chaque code block manuellement avec `⌘+Option+6` (Mac) ou `Ctrl+Alt+6` (Win) puis coller le code dedans.
4. Refaire les tables à la main si présentes.

### Méthode 3 — API Medium (automatisation)

Pour publier en masse via script :

```bash
npx markdown-to-medium ton-article.md --token=YOUR_MEDIUM_TOKEN
```

⚠️ L'API Medium est en mode "maintenance" depuis 2023 — fonctionne mais aucune nouvelle feature.

---

### Workflow recommandé pour la série Building Caresse

#### À l'import (depuis HackMD)

1. Titre Medium : copier depuis le frontmatter `title:`.
2. Sous-titre Medium : copier depuis `subtitle:`.
3. Tags (5 max) : `dotnet`, `hexagonal-architecture`, `software-architecture`, `ai`, `building-caresse`.
4. Image de couverture : upload la cover 1500×750 px.
5. **Member-only story** : selon le plan ci-dessus (#1 public, #2+ premium par défaut).
   - Public → laisser décoché (maximise SEO + funnel followers)
   - Premium → cocher ✅ (revenu Partner Program)

#### Avant publication

- [ ] Vérifier les code blocks (Medium les casse parfois à l'import)
- [ ] Vérifier que les liens `caresse.app` et Play Store sont cliquables
- [ ] Member-only activé
- [ ] Générer un **Friend Link** (pour partage externe → gens non-membres peuvent lire et potentiellement s'abonner)
- [ ] Si soumission à une publication (Better Programming, ITNEXT…) : soumettre **avant** publication, pas après

#### Après publication

- 📣 Partager le Friend Link sur Reddit (r/dotnet, r/csharp), HN, LinkedIn, Twitter/X
- 📣 Répondre à tous les commentaires dans les 24h → l'algo Medium boost les articles à fort engagement
- 📊 Vérifier les stats à J+7 → si "boost" Medium apparaît, multiplicateur ×5 à ×20 sur les vues

---

### Pièges connus à l'import

1. **Code blocks** : vérifier chacun manuellement même via HackMD. Medium a un mode "code gist multi-ligne" qu'il faut parfois réappliquer.
2. **Images** : upload-les directement sur Medium plutôt que via URL externe (Medium re-héberge de toute façon).
3. **Liens externes** : Medium les rend `no-follow` → pas de juice SEO direct pour caresse.app, mais le clic compte quand même pour le trafic.
4. **`<!--more-->`** : ignoré par Medium. Le paywall est placé automatiquement après les 3-4 premiers paragraphes pour les member-only stories.
5. **Frontmatter YAML** : à supprimer manuellement avant import.

---

*Généré le 2026-05-31 à partir de l'analyse des repos caresse-app, caresse-api et assets.*
