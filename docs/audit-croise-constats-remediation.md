# Audit croisé — constats et plan de remédiation

## Le principe de l'exercice

En fin de projet, les équipes s'auditent mutuellement : chacune monte son infrastructure,
puis passe en boîte noire sur celle d'en face, depuis un poste Kali externe et selon un
périmètre défini par lettre de mission. On récupère donc un rapport d'attaque sur sa propre
infra, écrit par des gens qui n'ont pas participé à sa construction.

C'est l'exercice le plus utile du projet : il montre l'écart entre ce qu'on croit avoir
configuré et ce qui est réellement joignable.

**Les constats ci-dessous proviennent du rapport rédigé par l'équipe adverse** sur notre
infrastructure. Le plan de remédiation et les chiffrages sont les nôtres.

## Ce qui a été trouvé

Cinq findings, dont deux de niveau haut. Vecteurs en CVSS 3.1.

| ID | Constat | Sévérité | CVSS | Vecteur |
|---|---|---|---|---|
| F1 | GUI pfSense + XML-RPC exposés | HAUTE | 8.1 | `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` |
| F8 | nginx 1.22.1 — CVE-2026-42533 | HAUTE | 8.1 | `AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| F4 | SSH — authentification par mot de passe | MOYENNE | 6.5 | `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N` |
| F2 | PostgreSQL exposé (pg_hba refuse) | MOYENNE | 5.3 | `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` |
| F7 | Médecin2 → BDD + pfSense joignables (ICMP) | MOYENNE | 5.3 | `AV:N/AC:L/PR:L/UI:N/S:C/C:L/I:N/A:N` |

**F1 — GUI pfSense et XML-RPC exposés.** Le plus grave. XML-RPC est l'API de gestion à
distance de pfSense : elle répond sur `/xmlrpc.php` et permet d'exécuter des commandes
d'administration en HTTP. Accessible depuis la zone d'attaque, avec un mot de passe faible
ou connu, elle permettrait de modifier les règles firewall, monter des tunnels, couper des
services ou exfiltrer toute la configuration réseau. Les mots de passe par défaut ont été
testés et refusés — mais la surface d'attaque, elle, existe bel et bien.

**F8 — nginx 1.22.1.** Version vulnérable à une CVE publiée en 2026, dont un PoC public
existe. L'équipe adverse a confirmé le débordement sur nos workers.

**F4 — SSH par mot de passe.** Un compte avait un mot de passe faible, ce qui a suffi à
prendre la main sur la machine.

**F2 — PostgreSQL exposé.** Le port 5432 répondait au réseau, même si `pg_hba.conf`
refusait effectivement la connexion. La seconde couche a tenu, mais le service n'aurait pas
dû être joignable.

**F7 — cloisonnement inter-sites incomplet.** Depuis le poste Médecin2 du site 2
(192.168.200.11), la base de données (172.16.20.2) et l'interface d'administration pfSense
(192.168.30.1) répondaient au ping. Une règle de blocage manquait sur pfSense-2 entre le
segment du site 2 et les zones DATA et ADMIN. La segmentation était pensée, mais pas
appliquée jusqu'au bout — exactement le genre d'écart qu'un pentest sert à révéler.

## Ce qui a tenu

L'audit n'a trouvé aucune faille applicative : ni CSRF, ni injection SQL, ni XSS, ni
contournement du contrôle d'accès. sqlmap a passé 909 charges d'injection sur le frontend
sans qu'une seule aboutisse, et les routes protégées ont bien renvoyé `403 Forbidden` aux
comptes non autorisés.

Les cinq findings portent tous sur la configuration réseau et système, pas sur la
conception applicative.

## Plan de remédiation

| Priorité | ID | Action | Temps | Coût interne |
|:---:|---|---|---|---|
| P0 | F1 | Restreindre la GUI pfSense au réseau Admin et XML-RPC au seul flux de synchronisation nécessaire | 2 h 30 | 125 € |
| P0 | F8 | Monter nginx en 1.30.4+ et faire les tests de non-régression | 4 h | 200 € |
| P1 | F4 | Vérifier les clés SSH puis désactiver l'authentification par mot de passe | 2 h | 100 € |
| P1 | F7 | Durcir les règles pfSense inter-sites et tester les flux autorisés et interdits | 4 h | 200 € |
| P2 | F2 | Limiter PostgreSQL 5432 au réseau applicatif et vérifier `pg_hba.conf` | 1 h 30 | 75 € |

**Total estimé : 14 h, soit environ 2 jours — 700 €.**

Hypothèse de calcul : coût interne chargé de 350 € par jour. Les estimations incluent la
sauvegarde préalable, la mise en œuvre, les tests et la validation. Aucun achat de licence
n'est nécessaire.

## État réel des corrections

Il faut être clair là-dessus : le projet s'est terminé peu après la réception du rapport.

**F4 a été corrigée et vérifiée.** L'authentification SSH est passée en clés Ed25519, le
mot de passe et la connexion root ont été désactivés, et la correction a été validée depuis
la machine d'attaque — `Permission denied (publickey)`. La procédure complète est dans
[ssh-par-cle.md](ssh-par-cle.md).

**Les quatre autres n'ont pas été appliquées.** Elles sont analysées, priorisées et
chiffrées, mais pas mises en œuvre faute de temps. Le plan ci-dessus est un livrable
d'audit, pas un compte rendu de travaux réalisés.
