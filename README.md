# Stack Prometheus + Grafana sur Dokploy

## Arborescence
```
monitoring-stack/
├── docker-compose.yml
├── .env.example
├── prometheus/
│   ├── prometheus.yml
│   └── rules/base-alerts.yml
├── alertmanager/alertmanager.yml
├── grafana/provisioning/
│   ├── datasources/datasource.yml
│   └── dashboards/dashboard.yml   (+ dashboards-json/ à remplir)
└── logrotate/docker-containers    (à installer côté HÔTE, hors Dokploy)
```

## 1. Déploiement sur Dokploy

1. Pousse ce dossier sur un repo Git (GitHub/GitLab/Gitea) — Dokploy build à partir d'un repo, ou via import "Raw".
2. Dans Dokploy : **Project → Create Service → Compose**, type **Docker Compose** (pas "Stack", pour garder `healthcheck` et les options complètes).
3. Connecte ton repo, chemin vers `docker-compose.yml` à la racine du dossier `monitoring-stack/`.
4. Onglet **Environment** : colle le contenu de `.env.example` en remplaçant les valeurs (mot de passe Grafana fort, domaine réel).
5. **Ne mappe aucun `ports:`** dans le compose — Dokploy attache automatiquement tes services au réseau `dokploy-network` et route via son Traefik intégré.
6. Onglet **Domains** (recommandé, plutôt que des labels Traefik manuels) :
   - Ajoute un domaine pour le service `grafana`, port interne `3000`, HTTPS + Let's Encrypt.
   - Si tu exposes Prometheus/Alertmanager publiquement, ajoute une **Basic Auth** dans les options de domaine Dokploy (sinon garde-les uniquement accessibles via le réseau interne — voir sécurité plus bas).
7. Déploie. Vérifie les logs dans l'onglet **Logs** du service.

## 2. Rotation des logs vs rétention des métriques — ne pas confondre

- **Logs des conteneurs** (stdout de Prometheus/Grafana/etc.) : gérés par le driver Docker `json-file` défini en ancre YAML `x-logging` dans le compose (`max-size: 20m`, `max-file: 5`, `compress: true`). C'est une rotation **par taille**, pas par date.
- **Pour une rotation vraiment hebdomadaire** des fichiers de logs bruts sur le serveur : installe `logrotate/docker-containers` sur l'**hôte** (là où tourne le daemon Docker, pas dans un conteneur) :
  ```bash
  sudo cp logrotate/docker-containers /etc/logrotate.d/docker-containers
  sudo logrotate -d /etc/logrotate.d/docker-containers   # dry-run pour vérifier
  ```
  logrotate est déjà appelé quotidiennement par cron sur la plupart des distros ; la directive `weekly` fera qu'il ne tourne réellement qu'une fois par semaine.
- **Rétention des métriques Prometheus** (ce n'est pas du "log", ce sont des séries temporelles) : contrôlée par `--storage.tsdb.retention.time=15d` et `--storage.tsdb.retention.size=10GB` dans `docker-compose.yml`. Ajuste selon ton espace disque VPS. Une alerte (`PrometheusTSDBRetentionApproaching`) prévient à 90% de la limite en taille.

## 3. Dashboards Grafana

Le provisioning charge automatiquement tout JSON déposé dans `grafana/provisioning/dashboards-json/`. Dashboards communautaires recommandés à télécharger (via Grafana.com, ID à saisir dans Grafana ou JSON à placer directement) :
- **1860** – Node Exporter Full
- **14282** – cAdvisor / Docker containers
- **3662** – Prometheus 2.0 overview

## 4. Durcissement DevSecOps

- **Jamais de port publié** (`ports:`) pour Prometheus/Alertmanager/exporters : ils ne doivent être joignables que via le réseau Docker interne. Seul Grafana (avec auth) est exposé.
- Si tu dois quand même exposer Prometheus (ex: Grafana externe qui le requête), passe par la **Basic Auth de Dokploy** ou un reverse proxy dédié — Prometheus n'a aucune authentification native.
- `prometheus` tourne en `user: "65534:65534"` (non-root).
- Change `GF_ADMIN_PASSWORD` avant tout déploiement, désactive `GF_USERS_ALLOW_SIGN_UP`.
- Sauvegarde régulièrement les volumes `prometheus_data` et `grafana_data` (Dokploy propose des backups planifiés au niveau volume/base — configurable dans l'onglet **Backups** du projet).
- Pense à un scrape "self" sur cAdvisor/node-exporter uniquement depuis le réseau interne, jamais exposés à Internet.

## 5. Alerting

`alertmanager/alertmanager.yml` est un squelette : décommente et configure Slack ou SMTP selon ton besoin, puis redéploie.
