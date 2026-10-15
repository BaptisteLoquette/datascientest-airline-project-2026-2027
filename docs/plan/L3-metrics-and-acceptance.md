# L3 — Metrics, Thresholds & Acceptance (medium → low level)

> **Level 3 of 3** — everything formal, trackable, interpretable: metric definitions, thresholds, and the essential behaviors that — once validated — **prove the solution exists**.
> Prérequis : **G2 approuvé** ([L2](L2-architecture.md)). **Gate G3** = entry ticket to sprint execution.
> Chaque gate est mesurable : script/pytest, Prometheus/Grafana, MLflow, ou protocole humain documenté.

---

## 1. Comment lire ce document

- **Gate** = seuil à franchir pour dire « cette capacité est livrée ». Vert = avance ; rouge = boucle de révision (jamais de gate silencieusement baissée : toute baisse remonte à G2/G1).
- **Où c'est mesuré** : `pytest` (contrats), Prometheus/Grafana (ops), MLflow (ML), protocole manuel (NPS, conformité).
- Les gates se raccrochent aux KPIs stratégiques de [L1 §7](L1-strategy.md) (K1–K7) et aux décisions de [L2](L2-architecture.md).

## 2. Catalogue de métriques — définitions formelles

### 2.1 ML — gates modèle (K1, K2, K4, K7 ; Étape 5)

Split **chronologique** (train = passé, test = futur) ; toutes les métriques calculées **par bucket de lead-time** T-6h / T-3h / T-1h, jamais agrégées seules.

| ID | Métrique | Définition formelle | Gate | Tracké par |
|---|---|---|---|---|
| **M1** | MAE | $\text{MAE}=\frac{1}{N}\sum_i \lvert \hat y_i - y_i \rvert$ (minutes) — $y$ = retard d'arrivée réel | **< 15 min** (K1) **et** ≥ 20 % de mieux que le baseline naïf (prédire la moyenne historique de la route) | `pytest` + run MLflow |
| **M2** | RMSE | $\sqrt{\frac{1}{N}\sum_i(\hat y_i-y_i)^2}$ | rapportée seulement ; ratio RMSE/MAE ≫ 3 ⇒ queue catastrophique → revue | MLflow |
| **M3** | Median AE | $\mathrm{median}\lvert \hat y_i - y_i \rvert$ | rapportée (robuste à la queue des retards) | MLflow |
| **M4** | Accuracy ±10 min | $\frac{1}{N}\sum_i \mathbf{1}\{\lvert \hat y_i - y_i \rvert \le 10\}$ (K2) | rapportée par bucket ; seuil fixé **après le 1er run baseline** (pas de valeur littérale justifiable en littérature `[INFERENCE]`) | MLflow |
| **M5** | Rappel classe « retardée » | $TP/(TP+FN)$ sur $y \ge 15$ min (convention EUROCONTROL) | **≥ 0.70** (état de l'art publié ≈ 0.62–0.64) ; precision + F1 rapportés | MLflow |
| **M6** | Discipline d'anticipation (K4) | métriques calculées **uniquement** sur prédictions émises **≥ 1 h avant** l'affichage officiel | tous gates M1/M4/M5 validés **dans chaque bucket** ; stamp `lead_time_min` stocké avec chaque prédiction | `pytest` + `ml_predictions.lead_time_min` |
| **M7** | Calibration du score (K7) | Brier $=\frac{1}{N}\sum_i(p_i-o_i)^2$ ; diagramme de fiabilité ; ECE sur 10 bins équiprobables | **ECE ≤ 0.05** `[INFERENCE]` ; un affiché « 85 % » doit tomber dans la fréquence observée de son bin | MLflow artefact |
| **M8** | Calibration régression (si quantiles) | couverture empirique d'un intervalle 80 % ∈ [75 %, 85 %] | si `reg:quantileerror` utilisé à la place du classifieur | MLflow |

**Règles anti-fuite (tests dédiés, §4-B)** : aucun `ArrDelay/DepDelay/*Delay` post-départ en features ; rotation = jambe **précédente** uniquement ; météo = forecast **émise à l'instant de la prédiction** ; split chronologique (interdiction de CV aléatoire).

### 2.2 Data & ingestion (Étape 4 ; ops)

| ID | Métrique | Définition | Gate | Tracké par |
|---|---|---|---|---|
| **D1** | Lag d'ingestion | `time() - ingest_last_success_timestamp_seconds` | ≤ 60 s nominal ; **alerte > 90 s** (3 cycles manqués) | Prometheus/Grafana |
| **D2** | Cadence positions | nb de snapshots `pos.raw` / durée | 1 tick / 30 s ± 5 s sur 1 h d'observation | `pytest` + jauge Grafana |
| **D3** | Fraîcheur Redis | `now - ts` lu depuis `pos:{icao24}` | < 70 s (TTL 60 s marge) | `pytest` |
| **D4** | Budget quotas | `credits_remaining` (header `X-Rate-Limit-Remaining`) | soft-stop à **90 %** du quota/jour ; **alerte < 10 %** | Prometheus `ingest_credits_remaining` |
| **D5** | Taux d'erreur upstream | 429 / total sur la fenêtre | 429 gérés (backoff) sans perte de tick ; 429 spikes = alerte | `ingest_api_requests_total{status}` |
| **D6** | Rotation de token | occurrences 401 sur 24 h | **= 0** (refresh proactif < 30 min, marge 30 s) | log + `pytest` TokenManager |
| **D7** | Intégrité relationnelle | FK orphelines | 0 (contraintes `ON DELETE` + check nocturne) | `pytest` |
| **D8** | Job 30 j (RGPD) | lignes > 30 j encore portant `icao24` | **0** après job ; journal = lignes affectées/jour | script + `pytest` |

### 2.3 API & serving (Étape 6 ; K3, K6)

| ID | Métrique | Définition | Gate | Tracké par |
|---|---|---|---|---|
| **S1** | Latence API | histogramme `api_request_duration_seconds{handler,method,status}` | **p95 ≤ 300 ms** sur les 7 endpoints ; alerte si p95 > 300 ms | Prometheus/Grafana |
| **S2** | Fraîcheur carte (K3) | age de l'affichage côté client à chaque tick | **rafraîchissement ≤ 30 s** (+ ≤ 4 s de traitement) | `pytest` e2e + mesure manuelle chrono |
| **S3** | Disponibilité | `/healthz` successes / probes | ≥ 99 % sur 7 j (démo) ; `stale: true` servi pendant outage upstream = **pass**, crash = fail | jauge Grafana |
| **S4** | Cache | `cache_hits / (hits+misses)` | **≥ 80 %** | app counters → Prometheus |
| **S5** | Sécurité/limite | requêtes rejetées par notre rate-limiter | limiteur actif (test flood → 429) | `pytest` |

### 2.4 Observabilité (Étape 7)

| ID | Métrique | Gate | Tracké par |
|---|---|---|---|
| **O1** | Métriques exposées | les métriques D1/D4/S1/S4 + `kafka_consumergroup_lag` + CPU/RAM existent et sont scrapées | check Prometheus `/api/v1/query` vide ou non |
| **O2** | Dashboards | ≥ 3 dashboards provisionnés : RED API · ingestion/quota · infrastructure | dossier `infra/grafana/` versionné |
| **O3** | Alertes | ≥ 4 règles actives : lag > 90 s, crédits < 10 %, p95 > 300 ms, consumer lag croissant | règles YAML + webhook de test reçu |
| **O4** | Logs | JSON structuré sur stdout + rotation docker (`max-size`) | `docker inspect` + lecture |

### 2.5 Produit & utilisateurs (K5)

| ID | Métrique | Définition | Gate | Tracké par |
|---|---|---|---|---|
| **U1** | Satisfaction interface (K5) | moyenne des notes ≥ 7/10 sur l'échelle proposée | **> 7/10** sur ≥ 10 testeurs (protocole : tâches F1–F5, 10 min, papier) | feuille de test, tableau `docs/` |
| **U2** | Compréhension du score | % testeurs qui interprètent 🟢/🟠/🔴 correctement sans aide | ≥ 80 % | idem |
| **U3** | Anticipation perçue | % testeurs trouvant l'info « avant l'affichage officiel » | ≥ 80 % | idem |
| **U4** | Accessibilité | axe (automatique) : violations bloquantes ; parcours clavier seul complet | axe = 0 bloquant ; parcours documenté passant | axe CI + revue manuelle |

### 2.6 Dérive des données & retrain (Étape 5 « drift »)

| ID | Métrique | Définition | Gate / déclencheur | Tracké par |
|---|---|---|---|---|
| **V1** | PSI par feature | $\sum_i (c_i-e_i)\ln(c_i/e_i)$ (bins, réf. = snapshot d'entraînement) | **< 0.1 stable** · 0.1–0.25 = enquête · **> 0.25 = retrain d'urgence** (2 semaines consécutives en 0.1–0.25 ⇒ retrain) | Evidently → métriques MLflow |
| **V2** | KS / JS | KS pour n ≤ 1000, Jensen–Shannon catégoriel | significatif (p < 0.05) sur feature critique (rotation, ATFM) ⇒ enquête | Evidently |
| **V3** | Drift de cible | PSI de la distribution des retards réels | > 0.25 ⇒ retrain + revue features | Evidently |
| **V4** | Drift de prédiction | PSI des prédictions | > 0.25 ⇒ enquête | Evidently |
| **V5** | Politique de retrain** | calendrier | **mensuel périodique** (colonne vertébrale — étude 3 933 runs : périodique > réactif) ; V1/V3 = override d'urgence ; tout run promu `champion` doit repasser M1–M8 | calendrier + MLflow |
| **V6** | Détection | comptage features en drift vs total | alerte dataset quand **≥ 50 % des colonnes** dérivent ou 1 feature critique > 0.25 | Evidently preset |

### 2.7 Conformité/RSE (Étape 7 ; engagements [L2 §10](L2-architecture.md))

| ID | Preuve | Gate | Tracké par |
|---|---|---|---|
| **C1** | Mentions RGPD (intérêt légitime énoncé) + test de balancement | documents existants, datés, versionnés | check dossier `docs/legal/` |
| **C2** | Effacement jet privé | requête → purge complète ≤ **48 h**, log de requêtes | `pytest` sur le script + log |
| **C3** | Décorrélation 30 j | voir D8 | idem |
| **C4** | Hébergement UE | infra = région EU documentée | vérification manuelle + doc |
| **C5** | AIPD-lite (CNIL PIA) | datée **avant** déploiement tier 1 | document |
| **C6** | Disclaimer couverture + « estimations » CO₂ | visible sur la carte et la fiche | `pytest` e2e (texte présent) |
| **C7** | Écoconception | NumÉcoDiag : score + captures ; budget CPU d'entraînement déclaré | rapport |
| **C8** | Titres de sources | attribution des datas (OpenSky, adsb.lol ODbL, CARTO…) | `pytest` (footer) |

## 3. Score de retard 🟢/🟠/🔴 — définition fermée (F5)

Décision à seuils formels (à confirmer G3) :

| Statut | Condition sur la **prédiction** $\hat y$ (retard arrivée) | Affichage |
|---|---|---|
| 🟢 À l'heure | $\hat y < 10$ min | vert + icône ✓ + texte « À l'heure » |
| 🟠 Retard | $10 \le \hat y < 30$ min | orange + icône ! + texte « Retard ≈ X min » |
| 🔴 Retard sévère / annulé | $\hat y \ge 30$ min **ou** statut annulé | rouge + icône ✗ + texte |

- **Jamais la couleur seule** (WCAG 1.4.1 : couleur + icône + texte — U4).
- Le bandeau affiche toujours : $\hat y$, fenêtre, `proba` calibrée (M7), météo explicative, `model:version`.

## 4. Comportements d'acceptation essentiels (BDD) — « si tout ceci est vert, la solution existe »

### A. Ingestion & données (Étape 4)

- **AC1** *Positions 30 s* — Étant donné des identifiants OpenSky valides, Quand le tick 30 s se déclenche, Alors un instantané du corridor est en Redis avec `ts` à jour, TTL ≥ 60 s, et D2/D3 sont verts.
- **AC2** *Quota* — Étant donné un quota journalier, Quand la consommation atteint 90 %, Alors le poller s'arrête proprement (soft-stop), ne dépasse jamais, et D4 reste vert.
- **AC3** *429* — Étant donné un 429 upstream avec `Retry-After`, Alors le poller backoffe et reprend sans perdre le prochain tick (D5).
- **AC4** *Token* — Étant donné un token de 30 min, Alors il est rafraîchi avant expiration : **zéro 401 sur 24 h** (D6).
- **AC5** *Batch T-1* — Étant donné `/flights/departure?airport=LFPG`, Alors le backfill nocturne peuple `flights` ; 404 = état légal non fatal.
- **AC6** *Météo* — Étant donné LFPG/LSGG, Alors METAR + TAF sont cacheés ≥ 15 min côté serveur ; aucune requête client directe vers les sources météo.
- **AC7** *Labels EU* — Étant donné horaire + réel, Alors les labels de retard sont construits et **validés** contre EUROCONTROL OTP15/ACI A15 (R2 documenté).

### B. Modèle (Étape 5)

- **AC8** *Anti-fuite* — Étant donné le jeu d'entraînement, Alors aucun test de fuite ne détecte de colonne post-départ / rotation future / météo observée dans les features ; split chronologique unique.
- **AC9** *Gates* — Étant donné un run candidate, Alors M1 (MAE < 15 **et** −20 % vs naïf), M5 (rappel ≥ 0.70), M6 (buckets ≥ 1 h), M7 (ECE ≤ 0.05) sont **verts dans chaque bucket** avant promotion.
- **AC10** *Registry* — Étant donné un run vert, Alors il est enregistré dans MLflow, versionné, alias `champion` ; un run rouge ne peut pas devenir `champion` (le pipeline le bloque).
- **AC11** *Drift* — Étant donné PSI > 0.25 (ou breach 2 fenêtres), Alors un événement de retrain est déclenché et loggé dans MLflow (V1–V5).
- **AC12** *Régression de modèle* — Étant donné le même split, Alors le nouveau `champion` ne dégrade aucun gate vs l'ancien (rollback = revenir à la version N-1).

### C. API & carte (Étapes 6)

- **AC13** *Contrats* — Étant donné les 7 endpoints, Alors schemas valides, ETag/cache-headers conformes au tableau [L2 §6](L2-architecture.md), p95 ≤ 300 ms (S1).
- **AC14** *Recherche F1* — « AF118 » ou « Paris → Genève » → vol trouvé < 2 s.
- **AC15** *Fiche F2* — tout vol → provenance, escale, destination, appareil, météo locale affichés avec source.
- **AC16** *Dégradation* — Étant donné une coupure OpenSky, Alors l'API sert le dernier instantané avec `stale: true`, la carte reste peuplée (S3).
- **AC17** *Carte 30 s* — sur 10 min d'observation, chaque rafraîchissement ≤ 30 s + 4 s (S2), marqueurs diffés par `icao24`, rotation `true_track`.
- **AC18** *Prédiction F4* — Étant donné un vol, Alors la carte/fiche affiche $\hat y$, proba calibrée, motif, météo, `model:version`, stamp de lead-time ≥ 1 h (M6).

### D. Frontend & UX (F3/F5, K5)

- **AC19** *Score* — 🟢/🟠/🔴 exact selon §3, **jamais** couleur seule (U2).
- **AC20** *Contrôle utilisateur* — bouton pause du refresh actif ; `prefers-reduced-motion` respecté ; cadence modifiable (30 s ↔ 5 min).
- **AC21** *Clavier & ARIA* — parcours : Tab → carte → flèches → +/- → Entrée → sortie documentée ; mises à jour retard en `aria-live="polite"` groupées par tick.
- **AC22** *NPS/satisfaction* — protocole U1 exécuté : moyenne > 7/10 sur ≥ 10 testeurs, feuilles conservées.

### E. Conformité & RSE (Étape 7, [L2 §10](L2-architecture.md))

- **AC23** *Effacement* — requête propriétaire privé → purge live + historique + backups ≤ 48 h, ligne de log écrite (C2).
- **AC24** *Décorrélation* — job 30 j : 0 ligne âgée portant `icao24` après exécution, journal de preuves (D8/C3).
- **AC25** *Transparence* — mentions RGPD + disclaimer couverture + mention « estimation » sur CO₂ visibles (C1/C6).
- **AC26** *AIPD* — document daté avant le déploiement tier 1 (C5).

### F. Plateforme (Étapes 6–7)

- **AC27** *CI* — push/PR → ruff + pytest verts ; images build ; `compose config` valide.
- **AC28** *Prod* — tier 1 up : healthchecks, TLS, redémarrage auto, backup nocturne ; dashboards O1–O3 verts, 4 alertes testées (O3).
- **AC29** *Smoke e2e* — script : search → fiche → carte 30 s → prédiction visible → métriques bougent dans Grafana, transcript archivé.

## 5. Stratégie de tests (conventions repo : TDD/BDD, sémantique/property/fuzz, **nombre minimal**)

- **Rédiger le test avant l'implémentation** (rouge → vert → refactor) pour chaque AC.
- **Property-based (Hypothesis)** là où c'est généralisable : parsing payloads OpenSky (toutes shapes possibles → jamais d'exception non gérée), respect invariant TTL/cadence, transformations de fuseaux horaires des vols, seuils du score §3 monotones.
- **Fuzz léger** sur parsers météo (METAR brut) + JSON upstream.
- **Contract tests** pour les 7 endpoints (schéma Pydantic) plutôt que 50 tests de cas particuliers.
- **Pas de tests** : copies, valeurs par défaut incidentes, « ne lève pas d'erreur » vide.
- Chaque AC = un ou deux tests max ; un tableau de traceability AC ↔ test maintenu dans ce fichier.

## 6. Definition of Done par Étape + solution finale

| Étape | Done quand | Gates concernés |
|---|---|---|
| 1 Sources | rapport + échantillons réels archivés | AC7 |
| 2 CdC/veille | docs rendus (déjà largement produits par `research/`) | — |
| 3 Indicateurs | **G3 signé** = ce document approuvé | tout |
| 4 Ingestion | AC1–AC7 verts 24 h continus | D1–D8 |
| 5 ML | AC8–AC12 verts, run `champion` en place | M1–M8, V1–V6 |
| 6 API/carte | AC13–AC21 verts en dev | S1–S5, U2, U4 |
| 7 Prod/RSE | AC22–AC29 verts en tier 1 | O1–O4, C1–C8, U1 |
| **Solution livrée** | **tous les gates de §2 vert + tous les AC de §4 validés** = indique formellement que la plateforme promise (CdC §8 + §5 KPIs) existe | — |

## 7. Checklist de gate G3 — avant exécution des sprints

- [ ] M1–M8 : formules et seuils acceptés (dont le protocole « seuil M4 fixé après 1er baseline »)
- [ ] D1–D8, S1–S5, O1–O4 : seuils ops acceptés (300 ms p95, 90 s lag, 80 % cache…)
- [ ] V1–V6 : bandes PSI et politique de retrain mensuelle acceptées
- [ ] U1–U4 : protocole testeurs (≥ 10, tâches, barème > 7/10) accepté
- [ ] C1–C8 : preuves de conformité listées et attribuées
- [ ] §3 seuils du score 🟢/🟠/🔴 confirmés
- [ ] AC1–AC29 : chaque comportement a un responsable et un sprint
- [ ] §5 stratégie de tests (TDD/BDD minimal, property) alignée avec ENG-24 conventions

**Retour arrière possible** : tout point ci-dessus non acceptable → révision L2 (ou L1 si stratégique), jamais de contournement silencieux.