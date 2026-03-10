# LoRa Stack (Node-RED / InfluxDB 3 / Grafana behind HAProxy)

Ce dépôt installe une stack Docker complète pour recevoir les données LoRa (depuis TTN) :

- **HAProxy** est le point d’entrée unique côté public (port 80). Il redirige `http://<VM_IP>/nodered` vers Node-RED, `/grafana` vers Grafana et `/api` vers InfluxDB.  
- **Node-RED** (custom `lora-nodered:latest` dérivé de `nodered/node-red`) gère les flux LoRa sur `/nodered` et embarque `node-red-contrib-influxdb` pour écrire directement dans InfluxDB 3.
- **InfluxDB 3 Core** (`influxdb:3-core`) stocke les données suite à un service lancé avec `--object-store file` et écoute sur le backend HTTP 8181 exposé sous `/api`.  
- **Grafana** (`grafana/grafana`) tourne avec `GF_SERVER_ROOT_URL=.../grafana` et `GF_SERVER_SERVE_FROM_SUB_PATH=true`, donc toutes les URLs restent préfixées par `/grafana` (ex. `/grafana/login`).

## URLs exposées (via HAProxy)

- Node-RED : `http://<public-ip>/nodered`  
- Grafana : `http://<public-ip>/grafana`  
- API InfluxDB : `http://<public-ip>/api`

## Broker MQTT (Mosquitto)

- Le service `mosquitto` écoute sur le port TCP `1883` de la VM : `mqtt://<public-ip>:1883`.
- Les identifiants sont définis dans `.env` via `MOSQUITTO_USERNAME` et `MOSQUITTO_PASSWORD`; changez-les avant déploiement.
- Pour générer ou mettre à jour le fichier de mots de passe, exécutez :

  ```sh
  docker compose run --rm mosquitto \
    mosquitto_passwd -b /mosquitto/config/password ${MOSQUITTO_USERNAME} ${MOSQUITTO_PASSWORD}
  ```

  La commande est lancée une fois au démarrage du conteneur via `docker-compose` mais il est bon de la rerun chaque fois que vous modifiez les identifiants.

`config/mosquitto/mosquitto.conf` désactive l’accès anonyme et persiste les données/logs dans les volumes `mosquitto_data` et `mosquitto_log`.

## Authentification HAProxy

HAProxy force désormais une authentification HTTP basique sur toutes les URLs (identifiants définis dans `config/haproxy.cfg`). Le frontend utilise un `userlist` `protected` et `http-request auth` : vous devez remplacer `admin`/`ChangeMeClient123!` par vos propres identifiants (idéalement stockés dans un mot de passe haché via `password` ou `password-file`). Les navigateurs demanderont ces identifiants dès qu’ils atteignent `http://<public-ip>/...`.

*(Remplacez `<public-ip>` par l’adresse publique Azure de la VM. L’interface est sécurisée par les secrets définis dans `.env`.)*

## Démarrage

1. Copier `.env.example` en `.env` et mettre vos propres mots de passe/tokens (`NODE_RED_PASSWORD`, `INFLUX_TOKEN`, `INFLUX_PASSWORD`, `GRAFANA_PASSWORD`, `INFLUXD_NODE_ID`, `INFLUXD_LOG_FILTER`, `INFLUXD_DATA_DIR`).  
2. Lancer `docker compose up -d`.  
3. Les volumes `nodered_data`, `influxdb_data`, `grafana_data` persistent l’état (flows, buckets, dashboards).  
4. Le routage HAProxy `/nodered`, `/grafana`, `/api` garantit qu’aucun autre service n’est exposé directement.

## Notes de sécurité / production

- Limitez l’accès Azure/OS (NSG, `ufw`) au port 80 uniquement.  
- Changez les identifiants par défaut dans `.env` avant toute mise en production.  
- Le volume InfluxDB est écrit en tant que `root` pour éviter les erreurs de permission sur les fichiers catalogues.
