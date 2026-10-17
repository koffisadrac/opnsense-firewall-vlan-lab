# 🔥 OPNsense Firewall & VLAN Lab

> Volet pare-feu du projet **NovaTech** — Projet d'intégration II, AEC Sécurité et réseaux informatiques
> **Kanga Koffi Sadrac (KKS)** — octobre 2026
>
> 📦 Projet complet (Active Directory, partages, VPN, sauvegarde, rapport) : **[novatech-infrastructure](https://github.com/koffisadrac/novatech-infrastructure)**

## 📌 Présentation du projet

Ce laboratoire présente la partie réseau et pare-feu de l'infrastructure NovaTech, une PME fictive de 24 employés et 6 consultants.
J'ai utilisé OPNsense (FW01) comme pare-feu et routeur central. Il segmente le réseau en VLAN selon le niveau de confiance et contrôle chaque flux entre les zones.

## 🎯 Objectifs

- Segmenter le réseau par niveau de confiance (serveurs, administration, employés, consultants, invités)
- Bloquer par défaut tout trafic entre les VLAN et n'autoriser que les flux nécessaires
- Relayer le DHCP vers le contrôleur de domaine
- Fournir l'accès Internet par NAT
- Offrir un accès distant sécurisé avec WireGuard
- Prouver l'isolement par des tests positifs et négatifs

## 🧰 Environnement du laboratoire

| Élément | Détail |
|---|---|
| Virtualisation | VMware Workstation |
| Pare-feu | OPNsense (FW01) |
| Serveur | Ubuntu Server 24.04 — Samba AD, BIND9, Kea DHCP (DC01) |
| Clients de test | Windows 11, Windows 10, Kali Linux, Ubuntu Desktop |
| Réseaux virtuels | VMnet8 (WAN/NAT), VMnet1 à 5 (segments internes) |

## 🗺️ Architecture réseau

Toutes les zones passent par OPNsense : aucun segment ne communique directement avec un autre.
Le WAN sort vers Internet par le NAT de VMware ; chaque VLAN interne a sa passerelle sur FW01.

## 🌐 Plan VLAN et adressage IP

| VLAN | Nom | Réseau | Passerelle | Accès |
|---|---|---|---|---|
| 10 | SERVEURS | 10.10.10.0/24 | 10.10.10.254 | DC01 (10.10.10.10) |
| 20 | ADMIN-IT | 10.10.20.0/24 | 10.10.20.1 | Administration |
| 60 | EMPLOYÉS | 10.10.60.0/24 | 10.10.60.1 | Domaine, partages, Web |
| 80 | CONSULTANTS | 10.10.80.0/24 | 10.10.80.1 | Web seulement |
| 90 | INVITÉS | 10.10.90.0/24 | 10.10.90.1 | Internet seulement |
| — | VPN | 10.10.200.0/24 | 10.10.200.1 | ICMP + SSH vers DC01 |

## ⚙️ Configuration OPNsense

- **Interfaces** : WAN en DHCP (VMnet8) ; SERVERS, ADMIN, EMPLOYES, CONSULTANTS et GUEST en adresses statiques.
- **DHCP** : le serveur intégré est désactivé. *Services › DHCRelay* relaie les requêtes des VLAN vers **Kea sur DC01 (10.10.10.10)**.
- **DNS** : les postes internes utilisent DC01 ; seul le VLAN INVITÉS utilise des résolveurs publics.
- **VPN** : instance WireGuard `wg-admin` (10.10.200.1/24, UDP 51820), un pair par utilisateur.

## 🧱 Règles de pare-feu

Les règles sont évaluées de haut en bas. Tout ce qui n'est pas autorisé est bloqué.

| Interface | Action | Source | Destination | Ports | But |
|---|---|---|---|---|---|
| EMPLOYES | Pass | EMPLOYES net | DC01 | AD TCP/UDP, 445 | Domaine et partages |
| EMPLOYES | Block | EMPLOYES net | INTERNAL_NETS | any | Bloquer le latéral |
| EMPLOYES | Pass | EMPLOYES net | any | 80/443 | Web |
| CONSULTANTS | Block | CONSULTANTS net | INTERNAL_NETS | any | Isolement |
| CONSULTANTS | Pass | CONSULTANTS net | any | 80/443 | Web |
| GUEST | Block | GUEST net | INTERNAL_NETS | any | Internet seulement |
| GUEST | Pass | GUEST net | any | 53/80/443 | Navigation |
| WAN | Pass | any | WAN address | UDP 51820 | WireGuard |
| WireGuard | Pass | 10.10.200.0/24 | DC01 | ICMP, TCP 22 | Accès VPN limité |

## 🌍 NAT et accès Internet

Le NAT sortant automatique d'OPNsense traduit les adresses internes vers l'adresse WAN.
Aucune redirection de port n'expose les services internes (AD, DNS, SMB). Seul le port UDP 51820 de WireGuard est ouvert sur le WAN.

Tests de sécurité

| Test | Résultat attendu | Résultat |
|---|---|---|
| Employé → DC01 (DNS, Kerberos, SMB) | Autorisé | ✅ |
| Employé → autre VLAN interne | Bloqué | ✅ |
| Consultant (Kali) → DC01 | Bloqué | ✅ |
| Consultant → Internet (Web) | Autorisé | ✅ |
| Invité → réseaux internes | Bloqué | ✅ |
| Client DHCP → bail Kea via le relais | Adresse reçue | ✅ |
| Client VPN → ping et SSH vers DC01 | Autorisé | ✅ |
| Client VPN → SMB | Bloqué | ✅ |

Les 14 tests du projet complet sont dans le [plan de tests NovaTech](https://github.com/koffisadrac/novatech-infrastructure/blob/main/tests/plan-de-tests.md).

Captures d'écran

Les captures de chaque test (interfaces, règles, relais DHCP, statut WireGuard, tests de blocage) se trouvent dans le
[rapport technique complet](https://github.com/koffisadrac/novatech-infrastructure/blob/main/docs/Rapport_NovaTech.pdf).

Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| Pare-feu contourné | Adaptateur hôte VMware connecté aux segments internes | Adaptateurs hôtes désactivés sur VMnet1 à 5 |
| VLAN EMPLOYÉS injoignable | Interface non assignée ou mal adressée | Réassignation et adresse statique 10.10.60.1/24 |
| Pas d'adresse DHCP | Relais non configuré vers DC01 | DHCRelay activé, destination 10.10.10.10 |
| VPN : 0 octet reçu | Clé publique du pair mal copiée | Vérification croisée des clés avec `wg show` |
| Pair WireGuard invisible | Instance non activée | Activation de l'instance et redémarrage du service |

Les 28 incidents du projet sont documentés dans l'annexe A du rapport.

Compétences acquises

- Conception d'un plan VLAN et d'un adressage IP par niveau de confiance
- Rédaction de règles de pare-feu selon le principe du moindre privilège
- Relais DHCP entre segments et intégration avec un serveur Kea
- Mise en place d'un VPN WireGuard et diagnostic des échanges de clés
- Vérification de l'isolement par des tests positifs et négatifs
- Diagnostic méthodique et documentation des incidents

---
*La sauvegarde XML d'OPNsense et les clés WireGuard ne sont pas publiées : elles contiennent des secrets.*
