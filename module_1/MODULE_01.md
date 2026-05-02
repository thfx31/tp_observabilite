# Module 1 - Prometheus

## Exercice 1 : Installer Prometheus et accéder à l'interface web
Objectif : lancer un seul conteneur Prometheus, accéder à l'interface web sur le port 9090 et vérifier que Prometheus se scrape lui-même.

### Lancer Prometheus
```shell
docker run -d --name prometheus -p 9090:9090 prom/prometheus:latest
```
&nbsp;
### Vérifier si la cible est UP
Ouvrir dans un navigateur http://localhost:9090

![Prometheus UP](../img/module_1/exercice_01.png)

Chercher le répertoire de storage (/etc/prometheus/prometheus.yml)
Vérification des logs
```shell
docker logs prometheus
```

Extrait des logs :
```shell
time=2026-04-27T08:49:34.326Z level=INFO source=main.go:1502 msg="Loading configuration file" filename=/etc/prometheus/prometheus.yml
```
&nbsp;

### Lecture du fichier prometheus.yml
```shell
docker exec -it prometheus cat /etc/prometheus/prometheus.yml
```
```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
       # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          app: "prometheus"
```

---

## Exercice 2 : Écrire votre premier prometheus.yml
Objectif : Remplacer la configuration par défaut par votre propre prometheus.yml. Définir un intervalle de scrape global de 10s, un external label environment=lab, et recharger Prometheus sans le redémarrer.

### Supprimer le container
```shell
 docker rm -f prometheus
```


### Créer le fichier avec les paramètres demandés
```yaml
global:
  scrape_interval: 10s
  external_labels:
    environment: lab

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

### Lancer le container
```shell
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_02/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

### Reload de la conf
```shell
curl -X POST http://localhost:9090/-/reload
```

### Vérification de la configuration sur l'IHM Prometheus
**Status > Configuration**
```
global:
  scrape_interval: 10s
  scrape_timeout: 10s
  scrape_protocols:
  - OpenMetricsText1.0.0
  - OpenMetricsText0.0.1
  - PrometheusText1.0.0
  - PrometheusText0.0.4
  evaluation_interval: 1m
  external_labels:
    environment: lab
  metric_name_validation_scheme: utf8
runtime:
  gogc: 75
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  track_timestamps_staleness: false
  scrape_interval: 10s
  scrape_timeout: 10s
  scrape_protocols:
  - OpenMetricsText1.0.0
  - OpenMetricsText0.0.1
  - PrometheusText1.0.0
  - PrometheusText0.0.4
  always_scrape_classic_histograms: false
  convert_classic_histograms_to_nhcb: false
  metrics_path: /metrics
  scheme: http
  enable_compression: true
  metric_name_validation_scheme: utf8
  metric_name_escaping_scheme: allow-utf-8
  follow_redirects: true
  enable_http2: true
  static_configs:
  - targets:
    - localhost:9090
otlp:
  translation_strategy: UnderscoreEscapingWithSuffixes
```

---
## Exercice 3 : Ajouter node_exporter et scraper les métriques système
Objectif : Lancer node_exporter et configurer Prometheus pour le scraper. Vérifier que la métrique node_cpu_seconds_total apparaît dans l'expression browser.


### Lancer Node Exporter et récupérer l'adresse IP du conteneur
```shell
docker run -d --name node-exporter -p 9100:9100 prom/node-exporter:latest
039fdaf21b0a413d39b8394b50c241f344a4f1c8c42266684c69da185bb91e46

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' node-exporter
172.17.0.3
```
&nbsp;
### Modification du fichier prometheus.yml
```yaml
global:
  scrape_interval: 10s
  external_labels:
    environment: lab

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

# Ajouté pour l'exercice 3
  - job_name: "node" 
    static_configs:
      - targets: ["172.17.0.3:9100"]
```

&nbsp;
### Reload de la conf
```shell
curl -X POST http://localhost:9090/-/reload
```
&nbsp;

### Requête CPU `node_cpu_seconds_total`

![Node UP](../img/module_1/exercice_03_1.png)

![Requete CPU](../img/module_1/exercice_03_2.png)

---
## Exercice 4 : Découverte de service : par fichier ou Kubernetes
Objectif : Remplacer les static_configs par un mécanisme de découverte. Sous Docker, utiliser la découverte par fichier ; sous Kubernetes, utiliser kubernetes_sd_configs avec un ServiceMonitor ou un bloc de découverte brut.

### Création targets.json
```json
[
  {
    "targets": ["172.17.0.3:9100"],
    "labels": {
      "job": "node",
      "source": "file_sd"
    }
  },
  {
    "targets": ["localhost:9090"],
    "labels": {
      "job": "prometheus",
      "source": "file_sd"
    }
  }
]
```

### Création nouvelle version du fichier prometheus.yml
```yaml
global:
  scrape_interval: 10s
  external_labels:
    environment: lab

scrape_configs:
  - job_name: "dynamic-targets"
    file_sd_configs:
      - files:
          - '/etc/prometheus/sd/targets.json'
        refresh_interval: 5s
```

### Arrêt du conteneur
```shell
docker rm -f prometheus
```

### Lancement avec le nouveau montage pour le Service Discovery
```shell
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_04/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_04/targets.json:/etc/prometheus/sd/targets.json \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

### Vérification du status dynamique
![Dynamic targets](../img/module_1/exercice_04_1.png)

### Modification du targets.json
Permet de vérifier la MAJ dynamique
```json
[
  {
    "targets": [],
    "labels": { "job": "node", "note": "cible_desactivee" }
  },
  {
    "targets": ["localhost:9090"],
    "labels": { "job": "prometheus" }
  }
]
```

### Vérification sur l'IHM Prometheus
![Dynamic targets 2](../img/module_1/exercice_04_2.png)

---
## Exercice 5 : Règles d'enregistrement (recording rules)
Objectif : Pré-calculer une requête coûteuse sous forme de règle d'enregistrement. Créer un fichier de règles qui enregistre job:http_requests:rate5m toutes les 30 secondes.

### Utilisation du container demo-api
```shell
# Se placer dans le bon dossier
pwd
/home/thomas/Git/tp_observabilite/module_1/Python-App/demo-api/app

# Vérifier les fichiers disponibles
ls
app.py  Dockerfile  prometheus.yml  requirements.txt  traffic.sh

# Build du container
❯ docker build -t demo-api:1.0 .
[+] Building 28.8s (11/11) FINISHED
...
=> exporting to image                                                                                                                                  0.1s 
 => => exporting layers                                                                                                                                 0.1s 
 => => writing image sha256:3a90da6dca4a6c41416011e5add480b0c7089674a3144e7025a2535f413fcf0d                                                            0.0s
 => => naming to docker.io/library/demo-api:1.0

# Démarrage du container
 ❯ docker run -d --name demo-api -p 8000:8000 demo-api:1.0
4a0df226781f911da738f48754b3460b90afeaae66095d5b86f06a881d787099

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' demo-api
172.17.0.4
```

### Vérifier que l'app répond
```shell
curl http://localhost:8000/metrics
# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 310.0
python_gc_objects_collected_total{generation="1"} 39.0
python_gc_objects_collected_total{generation="2"} 0.0
...
demo_http_requests_in_flight 0.0
# HELP demo_active_users Number of currently active users (simulated)
# TYPE demo_active_users gauge
demo_active_users 151.0
```

### Créer `rules/api_rules.yml`

```yaml
groups:
  - name: api_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: rate(demo_http_requests_total[5m])
```

### Lancer Prometheus

```shell
docker rm -f prometheus

docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_05/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_05/rules:/etc/prometheus/rules \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle

prometheus
8914cbc0c6cd81e4df765642047def0e5bcd6c579871018e829ba1b61f7f0a0a
```

### Interroger la metric `job:http_requests:rate5m`
```shell
❯ chmod +x traffic.sh
❯ ./traffic.sh
Generating traffic against http://localhost:8000 - Ctrl+C to stop
```

![Rule job:http_requests:rate5m](../img/module_1/exercice_05.png)

---
## Exercice 6 : Règles d'alerte et Alertmanager
Objectif : Définir une alerte qui se déclenche lorsque le taux d'erreur de demo-api dépasse 5 % pendant 2 minutes, la router vers Alertmanager, puis observer l'alerte qui se déclenche.

### Créer un fichier alermanager.yml
```shell
route:
  receiver: 'default'

receivers:
  - name: 'default'
```
&nbsp;

### Lancer Alertmanager
```shell
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_06/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager
ddf423b84ca31febc91ab4cc691e2a2076a7c64e5a5731cfb9ee55b0ba71a6fb

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' alertmanager
172.17.0.5
```
&nbsp;
### Créer un fichier d'alertes
```shell
mkdir alerts
vim alerts/api_alerts.yml
```

```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(demo_http_requests_total{status=~"5.."}[5m])) 
          / 
          sum(rate(demo_http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Taux d'erreur élevé sur {{ $labels.instance }}"
          description: "Le taux d'erreur dépasse 5% (actuellement {{ $value | printf \"%.2f\" }}%)"
```
&nbsp;
### Relancer Prometheus
```shell
docker rm -f prometheus

docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_06/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_06/rules:/etc/prometheus/rules \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_06:/etc/prometheus/sd/targets.json \
  -v /home/thomas/Git/tp_observabilite/module_1/exercice_06/alerts:/etc/prometheus/alerts \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```
&nbsp;

### Vérification de l'alerte
**Prometheus**
![Alert Prometheus](../img/module_1/exercice_06_1.png)
&nbsp;
**Alertmanager**
![Alert Alertmanager](../img/module_1/exercice_06_2.png)

---
## Exercice 7 : PromQL - bases : vecteurs instantanés et vecteurs de plage
Objectif : Mettre en pratique la différence entre un vecteur instantané, un vecteur de plage et un scalaire. Répondre aux questions à partir des métriques de demo-api

### Test de requêtes dans Prometheus

**demo_http_requests_total**
- Type : vecteur instantané (un échantillon par série au moment de l'évaluation)
- Explication : C'est une "photo" à l'instant T. Prometheus renvoie un seul échantillon (la valeur la plus récente enregistrée) pour chaque série temporelle correspondante.

**demo_http_requests_total[1m]**
- Type : vecteur de plage une tranche d'historique par série
- Explication : au lieu d'une valeur unique, on obtient une "tranche d'historique" contenant tous les points enregistrés durant la dernière minute pour chaque série (grâce au [1])
- Observation : dans l'onglet Graph, Prometheus affiche une erreur car il ne peut pas tracer un graphique directement à partir d'une liste de valeurs par point temporel

**rate(demo_http_requests_total[1m])**
- Type : vecteur instantané
- Explication : la fonction `rate()` prend un vecteur de plage en entrée pour calculer une pente, mais elle extrait un vecteur instantané.
- Labels : chaque jeu de labels représente une combinaison unique de caractéristiques du trafic

**scalar(sum(demo_http_requests_total))**
- Type : scalaire
- Explication : 
  - `sum(...)` est une fonction d'agrégation qui additionne toutes les séries ensemble pour n'en faire qu'une seule
  - `scalar(...)` prend ce vecteur (qui n'a plus qu'une seule ligne) et en retire tous les labels (job, instance, etc.) pour ne gearder que le nombre pur
  - On obtient un chiffre unique, sans aucune étiquette

---
## Exercice 8 : PromQL - agrégations et jointures
Objectif : calculer ces requêtes :
a) taux de requêtes total par endpoint
b) ratio d'erreurs par endpoint
c) taux de requêtes par pod, ordonné (utiliser topk).
&nbsp;
**Le trafic total par endpoint** 
- Requête : `sum by (endpoint) (rate(demo_http_requests_total[5m]))`
- Explication : `sum by (endpoint)` indique à Prometheus d'additionner toutes les séries qui partagent le même label endpoint et de supprimer les autres labels (status, method, etc.) du résultat final.

![Trafic by endpoint](../img/module_1/exercice_08_1.png)

&nbsp;
**Ratio d'erreur par endpoint** 
- Requête : `sum(rate(demo_http_requests_total{status=~"5.."}[5m])) / sum(rate(demo_http_requests_total[5m]))`
- Explication : on divise la somme des erreurs par la somme totale du trafic. Comme nous n'avons pas mis de by (...), le résultat est un vecteur unique représentant le taux d'erreur global de toute l'infrastructure.
![Ratio d'erreur by endpoint](../img/module_1/exercice_08_2.png)

&nbsp;
**Taux de requêtes par pod, ordonné**
On veut classer les serveurs (instances) par volume de trafic et ne garder que les plus chargés.
- Requête : `topk(3, sum by (instance) (rate(demo_http_requests_total[5m])))`
- Explication :
  - On calcule le trafic total par serveur avec sum by (instance).
  - L'opérateur topk(3, ...) filtre ce résultat pour ne conserver que les 3 meilleures valeurs.
  - Note sur le résultat : comme je n'ai qu'une seule instance de demo-api dans mon lab (172.17.0.4:8000), Prometheus n'affiche qu'une seule ligne, car il n'y a pas de 2ème ou 3ème place à attribuer.
![Taux de requetes par pod](../img/module_1/exercice_08_3.png)

---
## Exercice 9 : PromQL avancé : histogrammes et quantiles
