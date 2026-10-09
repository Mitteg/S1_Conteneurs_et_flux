# Matthieu Grenier - Master 2 YNOV Campus Montpellier - Séance 1

## Contexte du poste

Poste Windows, 3,9 Go de RAM, Docker Desktop (WSL 2) limité à 2,5 Go / 3 CPU / swap 6 Go via `.wslconfig`.

Les mesures de débit (partie 5) sont donc faites sur une machine contrainte.

## Versions

| Outil          | Version relevée sur mon poste                                         |
| -------------- | --------------------------------------------------------------------- |
| Docker Engine  | 27.5.1                                                                |
| Docker Compose | v2.32.4-desktop.1                                                     |
| Python         | À COMPLÉTER (3.12.x pour la partie 4.4 ; le poste avait aussi 3.14.0) |

## Partie 1

### Question 1.1

Non, le second conteneur ne voit pas le fichier, car chaque conteneur a sa propre couche inscriptible au-dessus de l'image en lecture seule, et celle de c1 a disparu avec lui.

### Mesure 1

| Grandeur                                          | Valeur relevée                                |
| ------------------------------------------------- | --------------------------------------------- |
| Taille de `python:3.12-slim`                      | 177 MB                                        |
| Taille de mon image finale                        | 193 MB                                        |
| Nombre de couches non vides dans `docker history` | 9                                             |
| Couche la plus volumineuse                        | Base Debian (`debian.sh ... trixie`), 87,7 MB |

### Mesure 2

| Scénario de rebuild                | Durée / étapes CACHED                     |
| ---------------------------------- | ----------------------------------------- |
| Aucune modification                | 4,9 s (build Docker : 1,8 s) / 5 CACHED   |
| Modification de `producer.py`      | 5,2 s (build Docker : 4,2 s) / 3 CACHED   |
| Modification de `requirements.txt` | 14,2 s (build Docker : 13,2 s) / 1 CACHED |

### Q1.2

Avec `COPY . ./` avant le `RUN pip install`, le scénario « modification de producer.py » changerait : la copie du code invaliderait le cache, donc le `pip install` (environ 9 s ici) serait refait à chaque édition, avec moins d'étapes en CACHED. Les deux autres scénarios resteraient identiques.

## Décisions

| Décision                          | Valeur retenue                  |
| --------------------------------- | ------------------------------- |
| Image                             | `vitals-producer:0.1`           |
| Utilisateur du conteneur          | UID 10001 (`appuser`), non-root |
| Nom du volume des journaux        | `vitals-logs`                   |
| Nom du réseau applicatif          | `vitals-net`                    |
| VITALS_TOPIC                      | `mimic-vitals`                  |
| interval / retries du healthcheck | `10s / 3`                       |
| Port hôte publié                  | `29092`                         |
| acks / linger.ms / partitioner    | À compléter                     |
| INTERVAL_S                        | À compléter                     |

## Partie 2

### Q2.1 — Volume nommé vs bind mount

Un volume nommé est préférable à un bind mount `./logs:/app/logs` pour deux raisons :

1. Le volume est géré par Docker et est indépendant de l'arborescence du projet sur l'hôte, ce qui rend le stockage plus portable entre environnements.

2. Le volume permet de conserver les données même si le conteneur est supprimé ou recréé, sans dépendre directement d'un dossier présent sur la machine hôte.

### Mesure 3 — Réseau et volume

Réseau créé : `s1-vitals-stream_vitals-net`

Type : `bridge`

Volume créé : `s1-vitals-stream_vitals-logs`

Point de montage Docker : `/var/lib/docker/volumes/s1-vitals-stream_vitals-logs/_data`

### Healthcheck

* `interval` : `10s`
* `timeout` : `5s`
* `retries` : `3`
* `start_period` : `10s`

Justification : un intervalle de 10 secondes permet de vérifier régulièrement l'état du conteneur sans effectuer des contrôles trop fréquents. Trois tentatives permettent de tolérer des échecs temporaires avant de considérer le conteneur comme non sain.

### Mesure 3 — Passage à healthy

À compléter avec le délai réellement observé.

## Partie 3

### Décisions retenues — Kafka

| Décision                                   | Valeur retenue      |
| ------------------------------------------ | ------------------- |
| Port hôte publié en face de `29092`        | `29092`             |
| `KAFKA_PROCESS_ROLES`                      | `broker,controller` |
| Facteur de réplication des topics internes | `1`                 |
| `KAFKA_AUTO_CREATE_TOPICS_ENABLE`          | `false`             |
| Nom du volume nommé des journaux Kafka     | `kafka-data`        |
| Listener annoncé interne                   | `kafka:9092`        |

### Mesure 4 — Démarrage de Kafka

Temps écoulé entre `docker compose up -d kafka` et le passage à `healthy` :

**À compléter avec la mesure réellement observée.**

### Création du topic

Topic : `mimic-vitals`

Nombre de partitions : `3`

Facteur de réplication : `1`

Le topic a été créé avec succès.

### Description du topic

| Partition | Leader | Réplicas | ISR |
| --------: | -----: | -------: | --: |
|         0 |      1 |        1 |   1 |
|         1 |      1 |        1 |   1 |
|         2 |      1 |        1 |   1 |

### Q3.1

Le facteur de réplication ne peut pas être `2` car il n'y a qu'un seul broker Kafka dans notre configuration. Kafka ne dispose donc pas de deux brokers distincts sur lesquels placer les deux copies de chaque partition et refuserait la création du topic avec une erreur indiquant que le facteur de réplication demandé est supérieur au nombre de brokers disponibles.

### Mesure 5 — Répartition des messages

| Clé  | Partition(s) et offsets observés |
| ---- | -------------------------------- |
| P001 | À compléter                      |
| P002 | À compléter                      |
| P003 | À compléter                      |

### Bornes actuellement relevées

La commande équivalente disponible dans l'image utilisée (`kafka-get-offsets`) a retourné :

```text
mimic-vitals:0:1
mimic-vitals:1:0
mimic-vitals:2:0
```

| Partition | Offset suivant |
| --------: | -------------: |
|         0 |              1 |
|         1 |              0 |
|         2 |              0 |

Soit actuellement **1 message présent au total**.

### Q3.2

À compléter après la lecture des cinq messages demandés dans la partie 3.4. Il faut vérifier expérimentalement si les trois messages ayant pour clé `P001` sont dans la même partition et si leurs offsets se suivent.

## Partie 4

### Mesure 6

| Forme de `depends_on`        | Le producteur affiche-t-il des erreurs de connexion au démarrage ? | Nombre de lignes d'erreur avant stabilisation |
| ---------------------------- | ------------------------------------------------------------------ | --------------------------------------------: |
| `depends_on: [kafka]`        | À mesurer                                                          |                                     À mesurer |
| `condition: service_healthy` | À mesurer                                                          |                                     À mesurer |

La forme `condition: service_healthy` sera conservée dans le rendu final.

## Partie 5

### Mesure 7 — Débit mesuré

| Grandeur                                             |    Valeur |
| ---------------------------------------------------- | --------: |
| Somme des offsets de fin avant (t0)                  | À mesurer |
| Somme des offsets de fin après 60 s (t1)             | À mesurer |
| Débit calculé `(t1 - t0) / 60`                       | À mesurer |
| Débit annoncé par le producteur à l'arrêt            | À mesurer |
| Valeur de `INTERVAL_S` et de `NB_PATIENTS` utilisées | À mesurer |

### Q5.1

À compléter après les deux mesures de débit.

### Deuxième mesure avec `INTERVAL_S=0.1`

| Grandeur                           |     Valeur |
| ---------------------------------- | ---------: |
| Débit mesuré avec `INTERVAL_S=0.1` |  À mesurer |
| Écart avec la valeur théorique     | À calculer |

### Mesure 8 — Coupure du broker

| Observation                                                         | Valeur relevée |
| ------------------------------------------------------------------- | -------------- |
| Le processus producteur s'arrête-t-il ?                             | À mesurer      |
| Message d'erreur exact affiché (première ligne)                     | À relever      |
| Délai entre `stop` et la première erreur                            | À mesurer      |
| Après `start`, la production reprend-elle sans relancer le script ? | À mesurer      |
| Des messages produits pendant la coupure ont-ils été perdus ?       | À mesurer      |

### Q5.2

À compléter après l'expérience de coupure du broker, en s'appuyant sur les paramètres `retries`, `acks` et `enable.idempotence` de la configuration du producteur.

### Q6.1

À compléter.
