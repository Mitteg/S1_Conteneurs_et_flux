# Matthieu Grenier - Master 2 YNOV Campus Montpellier - Séance 1

## Contexte du poste

Poste Windows, 3,9 Go de RAM, Docker Desktop (WSL 2) limité à 2,5 Go / 3 CPU / swap 6 Go via `.wslconfig`.

## Versions

| Outil          | Version relevée sur mon poste                                         |
| -------------- | --------------------------------------------------------------------- |
| Docker Engine  | 27.5.1                                                                |
| Docker Compose | v2.32.4-desktop.1                                                     |
| Python         | À COMPLÉTER (3.12.x pour la partie 4.4 ; le poste avait aussi 3.14.0) |

## Partie 1

### Question 1.1

Non, le second conteneur ne voit pas le fichier, car chaque conteneur a sa propre couche inscriptible au-dessus de l'image en lecture seule, et celle de c1 a disparu avec lui.

### Mesure 1 - Image et couches

| Grandeur                                          | Valeur relevée                                |
| ------------------------------------------------- | --------------------------------------------- |
| Taille de `python:3.12-slim`                      | 177 MB                                        |
| Taille de mon image finale                        | 193 MB                                        |
| Nombre de couches non vides dans `docker history` | 9                                             |
| Couche la plus volumineuse                        | Base Debian (`debian.sh ... trixie`), 87,7 MB |

### Mesure 2 - Cache de build

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

Après recréation du conteneur, le producteur a démarré à 09:11:11.314 UTC. Le premier contrôle de santé réussi s'est terminé à 09:11:18.580 UTC, soit environ 7,27 secondes après le démarrage. Les deux contrôles visibles dans l'historique ont retourné ExitCode: 0, indiquant leur réussite. Ce délai correspond à la fin du premier contrôle réussi et non à une mesure exacte de l'instant de transition vers l'état healthy.

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

Après l'arrêt du conteneur, Kafka a été redémarré avec `docker compose start kafka`. Le délai total entre le début de la procédure de redémarrage et la détection de l'état `healthy` a été de **89,49 secondes**. Cette durée inclut le temps d'exécution de la commande de démarrage et la vérification périodique de l'état du conteneur. Le résultat peut varier selon la charge de la machine et l'état du broker.

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

| Partition | Offset relevé |
| --------: | -------------: |
|         0 |              19003 |
|         1 |              12668 |
|         2 |              6334 |

Ces valeurs correspondent aux offsets relevés avant la mesure de débit. Elles ne permettent pas d'identifier la partition et l'offset exacts des messages P001, P002 et P003.

### Q3.2

La lecture des cinq premiers messages du topic mimic-vitals montre que les cinq messages se trouvent dans la partition 1, aux offsets 0 à 4. Deux messages concernent le patient P002, mais aucun ne concerne P001.

Cette lecture ne permet donc pas de vérifier si les trois messages de test ayant pour clé P001 sont dans la même partition et si leurs offsets se suivent. Une lecture ciblée des messages de test est nécessaire.

## Partie 4

### Mesure 6

| Configuration                | Erreurs au démarrage ?                        |                                      Nombre de lignes d'erreur avant stabilisation |
| ---------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------: |
| `depends_on: [kafka]`        | Oui, `Connection refused`                     | 3 lignes d'erreur observées, dont une signale 88 occurrences identiques supprimées |
| `condition: service_healthy` | Oui, `Coordinator load in progress: retrying` |                                                          1 ligne d'erreur observée |


### Q4.1

Avec depends_on: [kafka], Docker Compose démarre le producteur dès que le conteneur Kafka a démarré, sans attendre que le broker soit prêt. Des erreurs Connection refused ont donc été observées.

Avec condition: service_healthy, le producteur attend que le healthcheck de Kafka réussisse avant de démarrer. Un message temporaire Coordinator load in progress: retrying a néanmoins été observé, car le démarrage du broker ne garantit pas que toutes ses opérations soient immédiatement disponibles.

Le scénario avec condition: service_healthy limite donc les erreurs liées au démarrage prématuré du producteur.

## Partie 5

### Mesure 7 — Débit mesuré

| Grandeur                                             |    Valeur |
| ---------------------------------------------------- | --------: |
| Somme des offsets de fin avant (t0)                  | 60733 |
| Somme des offsets de fin après 60 s (t1)             | 62485 |
| Débit calculé `(t1 - t0) / 60`                       | 29,2 messages/s |
| Débit annoncé par le producteur à l'arrêt            |  |
| Valeur de `INTERVAL_S` et de `NB_PATIENTS` utilisées | NTERVAL_S=1.0, NB_PATIENTS=12 |

### Q5.1

Lorsque INTERVAL_S diminue de 1.0 à 0.1 seconde, le débit mesuré passe de 29,2 à 219,4 messages/s. La diminution de l'intervalle semble donc augmenter le débit de production. Toutefois, le second résultat dépasse le débit théorique de 120 messages/s attendu pour 12 patients avec un intervalle de 0,1 seconde. Cette mesure doit être vérifiée avant de conclure définitivement.

### Deuxième mesure avec `INTERVAL_S=0.1`

| Grandeur                           |     Valeur |
| ---------------------------------- | ---------: |
| Débit mesuré avec `INTERVAL_S=0.1` |  219,4 messages/s |
| Écart avec la valeur théorique     | 219,4 messages/s |

### Mesure 8 — Coupure du broker

| Observation                                                         | Valeur relevée |
| ------------------------------------------------------------------- | -------------- |
| Le processus producteur s'arrête-t-il ?                             | Non, le conteneur reste Up et healthy. |
| Message d'erreur exact affiché (première ligne)                     | Disconnected (after 1365316ms in state UP) |
| Délai entre `stop` et la première erreur                            | Première erreur observée juste après l'arrêt du broker ; délai exact non mesuré séparément. |
| Après `start`, la production reprend-elle sans relancer le script ? | Non confirmé : des erreurs Connection refused sont encore présentes dans les logs après le redémarrage. |
| Des messages produits pendant la coupure ont-ils été perdus ?       | À vérifier en comparant les offsets et les messages livrés avant et après la coupure.  |

### Q5.2

Lorsque Kafka est arrêté, le producteur ne s'arrête pas immédiatement et les tentatives d'envoi rencontrent des erreurs de connexion. Le paramètre retries=5 autorise plusieurs tentatives de livraison en cas d'échec, tandis que acks=all demande une confirmation du broker avant de considérer un message comme livré. Le paramètre enable.idempotence=True limite les doublons liés aux nouvelles tentatives de livraison. Le producteur ne peut toutefois pas garantir une attente illimitée : les délais de livraison et la capacité de sa file d'attente limitent la durée pendant laquelle les messages peuvent rester en attente.

### Q6.1

La commande docker compose down supprime les conteneurs et le réseau du projet, mais conserve les volumes nommés, dont kafka-data. Au prochain docker compose up, Kafka réutilise donc les données persistées dans ce volume, notamment les données des topics et leurs messages. À l'inverse, docker compose down -v supprime également les volumes nommés déclarés dans le fichier Compose. Au prochain démarrage, Kafka recrée un volume vide et les anciennes données du broker ne sont plus disponibles.
