# Documentation — DST JBM Airlines (CDG → GVA)

## Plan de projet (progressif, 3 niveaux — [ENG-25](https://linear.app/testaafdkjslkdgj/issue/ENG-25))

| Niveau | Fichier | Portée | Gate |
|---|---|---|---|
| L1 — very-high → high | [plan/L1-strategy.md](plan/L1-strategy.md) | vision, scope, personas, roadmap Étapes 0→7, SWOT, KPIs, risques, équipe | **G1** |
| L2 — high → medium | [plan/L2-architecture.md](plan/L2-architecture.md) | architecture, sources, ingestion/quotas, schéma données, API, frontend, ML, déploiement, conformité, sprints | **G2** |
| L3 — medium → low | [plan/L3-metrics-and-acceptance.md](plan/L3-metrics-and-acceptance.md) | métriques formelles M/D/S/O/U/V/C, seuils, score 🟢🟠🔴, comportements d'acceptation AC1–AC29, tests | **G3** |

**Méthode** : chaque niveau est validé avant de détailler le suivant ; une découverte en bas remonte en haut (boucle documentée en L1 §5).

## Recherches parallèles (preuves à l'appui du plan)

| Fichier | Couverture | Étape servie |
|---|---|---|
| [research/01-data-sources.md](research/01-data-sources.md) | OpenSky/adsb.lol/aviationweather.gov/Open-Meteo/Kaggle BTS/EUROCONTROL — endpoints, quotas, échantillons **vérifiés en live 2026-10-15** | Étape 1 |
| [research/02-ml-delay-prediction.md](research/02-ml-delay-prediction.md) | état de l'art 2024-2026, métriques formelles, drift (PSI/KS), MLflow, features CDG→GVA | Étapes 2, 5 |
| [research/03-realtime-architecture.md](research/03-realtime-architecture.md) | Kafka/Redis/PG/Mongo, FastAPI, carte 30 s, docker-compose, Prometheus/Grafana, coûts | Étapes 2, 4, 6, 7 |
| [research/04-compliance-rse.md](research/04-compliance-rse.md) | RGPD trajectoires, disclaimer, RGESN, CO₂ (ICAO/TIM), WCAG 2.2, obligations | Étapes 2, 7 |

## Convention de lecture (ADHD-first)

- Chaque fichier commence par **sa promesse + son gate** ; les tableaux résument, les mermaid montrent le flux, les bullets descendent en profondeur seulement si nécessaire.
- `[INFERENCE]` marque une hypothèse non vérifiée — tout le reste est sourcé (URLs dans les rapports de recherche).