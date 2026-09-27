
# Documentation Déploiement Domotique : Home Assistant & Chauffage Zigbee

Ce document récapitule l'ensemble des étapes techniques, des choix d'architecture et des configurations pour réinstaller ou reproduire le système domotique à partir de zéro.

---

## 1. Matériel & Spécifications

* **Hôte :** Raspberry Pi 4 Model B (Rev 1.1) - 4 Go RAM.
* **Système d'exploitation :** Raspbian GNU/Linux 11 (Bullseye) 64-bit (`aarch64` kernel, userland `armhf`).
* **Coordinateur Zigbee :** Sonoff Zigbee 3.0 USB Dongle Plus V2 (Modèle E - puce Silicon Labs EFR32MG21).
* Port physique hôte identifié : `/dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_d2aa11fe1d3ff1118da57533b6df0e24-if00-port0`.


* **Capteurs & Actionneurs :**
* Capteurs de température/humidité LCD : Sonoff SNZB-02D (Salon, Chambre Aya, Chambre Nour, Chambre parentale).
* Capteur de température étanche : Sonoff SNZB-02LD (Extérieur).
* Capteurs de température d'appoint : eWeLink SNZB-02P.
* Détecteur de présence / mouvements micro-ondes : Sonoff SNZB-06P (Salon).
* Capteur d'ouverture : eWeLink SNZB-04P (Salon).
* Actionneurs de radiateurs : Modules NodOn fil pilote (SIN-4-FP-21).



---

## 2. Préparation du Système & Docker

### 2.1 Installation de Docker

```bash
# Mise à jour système et installation officielle de Docker
curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh
sudo sh get-docker.sh

# Ajout de l'utilisateur au groupe Docker
sudo usermod -aG docker $USER
newgrp docker

# Test de confirmation
docker run --rm hello-world

```

---

## 3. Déploiement de Home Assistant

Le conteneur utilise le réseau de l'hôte (`--network=host`) pour la détection mDNS/UPnP et la communication locale, monte le socket D-Bus (stabilité Bluetooth) et mappe le port série de la clé Zigbee.

L'architecture est forcée en 64 bits (`--platform linux/arm64`) pour correspondre au noyau du Raspberry Pi 4 sans conflit de manifeste.

### 3.1 Création des répertoires persistants

```bash
mkdir -p ~/homeassistant/www

```

### 3.2 Lancement du conteneur

```bash
docker run -d \
  --name homeassistant \
  --platform linux/arm64 \
  --privileged \
  --restart=unless-stopped \
  -e TZ=Europe/Paris \
  -v ~/homeassistant:/config \
  -v /run/dbus:/run/dbus:ro \
  --device /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_d2aa11fe1d3ff1118da57533b6df0e24-if00-port0:/dev/ttyUSB0 \
  --network=host \
  ghcr.io/home-assistant/home-assistant:stable

```

### 3.3 Commandes d'administration courantes

* Suivre les logs de démarrage : `docker logs -f homeassistant`
* Redémarrer le service : `docker restart homeassistant`
* Stopper et supprimer le conteneur : `docker stop homeassistant && docker rm homeassistant`

---

## 4. Configuration Zigbee (ZHA)

1. Accéder à l'interface web : `http://:8123`.
2. Compléter l'assistant de premier démarrage (création compte administrateur).
3. Aller dans **Paramètres** > **Appareils et services**.
4. Valider l'intégration **Zigbee Home Automation (ZHA)** détectée :
* **Port série :** `/dev/ttyUSB0`
* **Type radio :** `Silicon Labs EmberZNet` (ou `EZSP`)
* **Vitesse :** `115200` bauds
* **Option :** Créer un nouveau réseau.


5. Procéder à l'appairage successif des capteurs Sonoff et modules NodOn fil pilote via le bouton **Ajouter un appareil Zigbee**.

---

## 5. Tableau de bord : Plan 2D & Sondes

Le plan de l'appartement est importé dans Home Assistant et sert de fond pour afficher l'emplacement réel de chaque sonde thermique et détecteur.

### 5.1 Configuration de la carte Lovelace (Picture Elements)

NB: this part isn't yet test and is under working (WIP)

Dans le tableau de bord Home Assistant, ajouter une carte en mode **Manuel (YAML)** :

```yaml
type: picture-elements
image:
  media_content_id: media-source://image_upload/916ff6587e2dfe982532fbd470fbdd4a
  media_content_type: image/png
  metadata:
    title: apprt_chauffours.png
    thumbnail: /api/image/serve/916ff6587e2dfe982532fbd470fbdd4a/256x256
    media_class: image
    navigateIds:
      - {}
      - media_content_type: app
        media_content_id: media-source://image_upload
elements:
  # Capteur Salon (SNZB-02D)
  - type: state-label
    entity: sensor.heath_sensor_lcd_1_temperature
    style:
      top: 25%
      left: 50%
      font-size: 13px
      font-weight: bold
      color: "#111111"
      background-color: "rgba(255, 255, 255, 0.85)"
      padding: 3px 6px
      border-radius: 4px
      box-shadow: 0 1px 3px rgba(0,0,0,0.2)

  # Capteur Chambre Aya (SNZB-02D)
  - type: state-label
    entity: sensor.heath_sensor_lcd_2_temperature
    style:
      top: 75%
      left: 35%
      font-size: 13px
      font-weight: bold
      color: "#111111"
      background-color: "rgba(255, 255, 255, 0.85)"
      padding: 3px 6px
      border-radius: 4px
      box-shadow: 0 1px 3px rgba(0,0,0,0.2)

  # Capteur Chambre Nour (SNZB-02D)
  - type: state-label
    entity: sensor.heath_sensor_lcd_3_temperature
    style:
      top: 75%
      left: 65%
      font-size: 13px
      font-weight: bold
      color: "#111111"
      background-color: "rgba(255, 255, 255, 0.85)"
      padding: 3px 6px
      border-radius: 4px
      box-shadow: 0 1px 3px rgba(0,0,0,0.2)

  # Capteur Chambre parentale (SNZB-02D)
  - type: state-label
    entity: sensor.heath_sensor_lcd_4_temperature
    style:
      top: 25%
      left: 70%
      font-size: 13px
      font-weight: bold
      color: "#111111"
      background-color: "rgba(255, 255, 255, 0.85)"
      padding: 3px 6px
      border-radius: 4px
      box-shadow: 0 1px 3px rgba(0,0,0,0.2)

  # Capteur de présence Salon (SNZB-06P)
  - type: state-icon
    entity: binary_sensor.sonoff_snzb_06p_occupancy
    style:
      top: 32%
      left: 42%
      transform: scale(0.85)

  # Capteur ouverture fenêtre Salon (SNZB-04P)
  - type: state-icon
    entity: binary_sensor.open_close_sensor_1_door
    style:
      top: 8%
      left: 30%
      transform: scale(0.8)

```

---

## 6. Procédure de Sauvegarde & Migration

Toutes les données (clés Zigbee, appareils appairés, automatisations, dashboards) résident dans le répertoire `~/homeassistant`.

### 6.1 Exporter une sauvegarde complète

```bash
tar -czvf ~/backup_ha_$(date +%F).tar.gz -C ~ homeassistant

```

### 6.2 Restaurer sur une nouvelle machine

```bash
# 1. Décompresser l'archive à la racine du home
tar -xzvf backup_ha_AAAA-MM-JJ.tar.gz -C ~

# 2. Brancher le dongle USB Zigbee et vérifier son chemin (/dev/serial/by-id/...)
ls -l /dev/serial/by-id/

# 3. Relancer la commande docker run (Section 3.2)

```
