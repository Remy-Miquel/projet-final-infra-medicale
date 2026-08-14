# Déploiement de l'agent Wazuh sur WEB-01

Supervision (SIEM) — collecte des journaux et détection d'intégrité.

## 1. Objectif

Wazuh assure la supervision de sécurité de l'infrastructure. Un agent est installé sur
chaque machine à surveiller ; il pousse ses journaux vers le serveur Wazuh central
(172.16.30.2), qui les analyse et génère des alertes. Ce document couvre le déploiement
sur WEB-01, la machine exposée en DMZ.

Le modèle est **agent → serveur** : l'agent initie la connexion (port 1514 pour les logs,
1515 pour l'enregistrement initial). Le serveur n'accède jamais aux machines surveillées.
Ce sens unique préserve l'isolation des segments, en particulier le verrouillage de DATA-01.

## 2. Installation

L'agent doit être dans la même version que le serveur — ici 4.12.0. La version est donc
fixée explicitement à l'installation, puis figée pour éviter une mise à jour automatique
qui désalignerait l'ensemble.

```bash
sudo apt install -y wazuh-agent=4.12.0-1
sudo apt-mark hold wazuh-agent      # fige la version
```

## 3. Configuration

Le fichier `/var/ossec/etc/ossec.conf` définit le serveur cible et un nom d'agent unique.

```xml
<client>
  <server>
    <address>172.16.30.2</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
  <enrollment>
    <agent_name>WEB-01</agent_name>
  </enrollment>
</client>
```

```bash
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

## 4. Problèmes rencontrés

Trois blocages avant que l'agent s'enregistre correctement.

**Version d'agent incompatible.** Le serveur renvoyait `ERROR: Incompatible version for
new agent` et l'enregistrement échouait en boucle. L'agent installé par défaut était en
4.14.7, donc plus récent que le manager en 4.12.0 — Wazuh refuse ce cas de figure.
Réinstallation en version exacte, puis `apt-mark hold` pour la figer.

**Adresse du serveur non configurée.** Au démarrage :
`ERROR (4112): Invalid server address found: 'MANAGER_IP'`. La réinstallation avait remis
le fichier de configuration par défaut, avec le placeholder à la place de l'adresse réelle.
Corrigé avec `sudo sed -i 's/MANAGER_IP/172.16.30.2/' ossec.conf`.

**Nom d'agent refusé.** `ERROR: Invalid agent name debian (same as manager)` — toutes les
machines du lab portaient le nom d'hôte `debian` par défaut, y compris le serveur Wazuh,
qui refuse un agent homonyme. Résolu en attribuant un nom explicite via
`<enrollment><agent_name>`, en supprimant l'ancienne clé dans `client.keys`, puis en
redémarrant. L'enregistrement passe alors : `Agent key generated for WEB-01`.

## 5. Journaux collectés

WEB-01 étant le frontend exposé, la couverture retenue est maximale.

| Source | Ce qu'on cherche à voir |
|---|---|
| nginx `access.log` / `error.log` | scans, requêtes anormales, erreurs HTTP |
| `gunicorn.log` | tentatives de connexion, erreurs applicatives |
| journald (ssh, sudo) | force brute, élévations de privilèges |
| FIM temps réel sur le répertoire applicatif | modification du code = signe de compromission |

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/nginx/access.log</location>
</localfile>

<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
  <filter field="_SYSTEMD_UNIT">^(ssh|sshd|sudo)</filter>
</localfile>

<!-- dans <syscheck> -->
<directories check_all="yes" report_changes="yes" realtime="yes">
  /home/debian/cabinet-medical/web-01-frontend
</directories>
```

**Spécificité Debian 12 :** `/var/log/auth.log` n'existe plus, les journaux
d'authentification passent par journald. La collecte utilise donc le format `journald`
avec un filtre sur les unités `ssh` et `sudo`, et non un chemin de fichier.

## 6. Résultat

L'agent est enregistré et actif, le serveur reçoit ses journaux. Le redémarrage confirme
la prise en compte des sources :

```
Analyzing file: '/var/log/nginx/access.log'.
Analyzing file: '/home/debian/gunicorn.log'.
Monitoring journal entries.
sca: Loaded policy 'cis_debian12.yml'
```

Ce qu'on y gagne : les journaux du portail, de l'application et du système sont centralisés
sur le SIEM ; toute modification du code applicatif déclenche une alerte en temps réel ;
un audit de conformité CIS Debian 12 tourne automatiquement sur l'agent ; et la traçabilité
des accès couvre l'exigence de journalisation du RGPD (article 5).

**Étape suivante :** déployer les agents sur APP-01 et DATA-01, avec la collecte des
journaux PostgreSQL sur DATA-01 — la traçabilité des accès aux données de santé est
l'exigence centrale du RGPD sur ce projet.
