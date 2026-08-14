# Authentification SSH par clé

Durcissement de l'accès SSH, administration centralisée depuis le poste ADMIN.

Cette mesure répond au finding **F4** de l'audit croisé (SSH en authentification par mot
de passe, CVSS 6.5). C'est la seule remédiation qui a été appliquée et vérifiée pendant
le projet — voir [audit-croise-constats-remediation.md](audit-croise-constats-remediation.md).

## 1. Objectif

Le test d'intrusion a identifié l'accès SSH comme surface d'attaque : un compte avait un
mot de passe faible, ce qui suffisait à prendre la main sur la machine. Après correction
des mots de passe, l'accès est passé en authentification par clé.

Le principe : seul le poste ADMIN, qui détient la clé privée, peut se connecter aux
serveurs. L'authentification par mot de passe est désactivée, donc le brute-force devient
impossible même avec un accès réseau complet. Une clé ne se devine pas.

## 2. Mise en place

**Génération de la paire, une seule fois sur le poste ADMIN.** Ed25519 retenu — moderne,
robuste, compact.

```bash
ssh-keygen -t ed25519 -C "admin-cabinet"
```

La clé privée (`~/.ssh/id_ed25519`) ne quitte jamais le poste ADMIN. La clé publique
(`~/.ssh/id_ed25519.pub`) est distribuée sur les serveurs ; seule, elle ne permet pas de
se connecter, elle sert à vérifier que le demandeur possède bien la privée.

**Distribution sur les serveurs :**

```bash
ssh-copy-id debian@192.168.30.2      # WEB-01
ssh-copy-id debian@172.16.10.2       # APP-01
ssh-copy-id debian@172.16.20.2       # DATA-01
ssh-copy-id <user>@172.16.30.2       # WAZUH
ssh-copy-id <user>@<ip-backup>       # BACKUP
```

Cela suppose que le poste ADMIN joigne chaque serveur sur le port 22 : les règles pfSense
autorisent ADMIN → serveurs en SSH, conformément au principe d'administration centralisée.

**Vérification avant de couper les mots de passe.** Étape à ne pas sauter : la connexion
doit s'établir sans rien demander.

```bash
ssh debian@192.168.30.2      # pas de mot de passe demandé = clé OK
```

**Désactivation de l'authentification par mot de passe**, dans `/etc/ssh/sshd_config` :

```
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

```bash
sudo sshd -t                     # valide la syntaxe avant de recharger
sudo systemctl restart ssh
```

**Méthode pour ne pas se verrouiller dehors :** chaque serveur est traité un par un, dans
l'ordre copie de clé → test de connexion → désactivation du mot de passe. Une session SSH
reste ouverte pendant la modification, en filet, jusqu'à ce qu'une nouvelle connexion
confirme que ça marche.

## 3. Validation côté attaquant

Depuis la Kali, toute tentative est rejetée immédiatement — l'accès est refusé avant même
qu'un mot de passe soit demandé :

```
$ ssh debian@192.168.30.2
Permission denied (publickey)
```

Le brute-force qui avait permis la compromission initiale n'est plus possible.

## 4. Cas de l'Active Directory

Le serveur AD étant sous Windows, l'authentification par clé façon Linux ne s'applique pas.
Son administration est sécurisée autrement : mots de passe robustes, accès RDP restreint au
seul poste ADMIN par les règles de pare-feu, et authentification NLA.

## 5. Bilan

Sur les serveurs Linux : authentification par clé uniquement, mot de passe désactivé,
connexion root interdite, administration centralisée sur le poste ADMIN.

Ce qui est intéressant ici, c'est que la protection ne repose pas sur un blocage réseau de
l'attaquant, mais sur la robustesse du service lui-même. Même en gardant un accès réseau
complet, la voie SSH est fermée.
