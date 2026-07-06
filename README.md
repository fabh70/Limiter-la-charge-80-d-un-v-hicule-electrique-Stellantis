# Estimation & coupure de charge EV en temps réel (sans dépendre du cloud constructeur)

Beaucoup d'intégrations véhicule (Stellantis, et d'autres marques) ne remontent
le niveau de batterie que toutes les 15 à 30 minutes. Ce projet permet de :

- estimer le niveau de batterie en **temps réel** en combinant le dernier % connu
  du cloud avec l'énergie réellement injectée par votre borne/prise locale
- calculer une **heure de fin de charge** fiable et un temps restant
- **couper automatiquement** la charge à l'heure prévue, ou en sécurité si le
  niveau estimé dépasse la cible + marge

Développé et testé sur une Opel Mokka-e avec intégration Stellantis, mais
applicable à n'importe quel véhicule dont l'intégration cloud est lente,
tant que vous avez un moyen de mesurer la puissance de charge en local
(prise connectée, borne avec API locale, shelly, etc.).

## Fichiers

- **`ev_charge_estimation_sensors.yaml`** — Package de sensors template à
  copier dans votre configuration (via `!include` ou directement dans
  `configuration.yaml`). Contient des noms d'entités génériques à adapter à
  votre installation (voir les commentaires en tête de fichier).

- **`ev_charge_stop_estimated_time.yaml`** — Blueprint d'automation, à
  importer directement dans Home Assistant (Paramètres > Automatisations >
  Blueprints > Importer un blueprint). Entièrement paramétrable via l'UI,
  aucune édition YAML nécessaire une fois les sensors en place.

## Installation

### 1. Sensors (obligatoire, à adapter manuellement)

1. Copiez le contenu de `ev_charge_estimation_sensors.yaml`.
2. Remplacez les entités génériques par les vôtres :
   - `sensor.ma_borne_puissance_w` → votre sensor de puissance de charge (W)
   - `sensor.mon_niveau_batterie_cloud` → le % batterie remonté par le cloud
   - `switch.ma_prise_de_charge` → le switch qui pilote la charge
   - Ajustez `capacite_kwh` (capacité utile de la batterie) et `rendement`
     (perte de charge AC/DC, ~0.9 par défaut)
3. Collez dans `configuration.yaml` (ou un fichier inclus), redémarrez HA.

### 2. Blueprint d'automation

1. **Paramètres > Automatisations et scènes > Blueprints > Importer un
   blueprint**, collez l'URL de partage (GitHub/Gist) ou importez le fichier
   directement.
2. Créez une nouvelle automation à partir du blueprint, sélectionnez vos
   entités (switch de charge, sensor batterie estimée, sensor heure de fin,
   sélecteur de cible), ajustez les marges de sécurité.

## Pourquoi un sensor "refresh" intermédiaire ?

Le sensor `platform: integration` (qui calcule l'énergie en intégrant la
puissance dans le temps) ne recalcule que lorsqu'il reçoit un **nouvel état**
de sa source. Si votre capteur de puissance ne pousse pas de nouvel état
quand la valeur reste stable (comportement fréquent), l'intégration se fige
pendant toute une charge à puissance constante — et donc tout ce qui en
dépend (batterie estimée, heure de fin) se fige aussi.

Le sensor `Puissance charge refresh` (trigger `time_pattern: /30s`) force un
nouvel état régulier, même à valeur inchangée, ce qui débloque le calcul.

## Pourquoi ne pas utiliser le binary_sensor "en charge" du cloud comme condition ?

Même piège : ce binary_sensor peut rester bloqué sur `off` pendant la charge
réelle à cause du polling lent du cloud, ce qui empêcherait l'automation de
coupure de se déclencher. Le blueprint utilise à la place l'état du switch
qui pilote physiquement la charge — plus réactif et sous votre contrôle
direct.

## Précision observée

Sur un test réel (Opel Mokka-e, 51 kWh), l'écart entre le niveau estimé et
le niveau réel constructeur à l'arrêt de charge était de 0,3 point de
pourcentage (79,7 % estimé vs 80 % réel) — largement suffisant pour un
arrêt fiable à la cible souhaitée.

## Licence

Partagé librement pour la communauté Home Assistant — à adapter selon vos
besoins.
