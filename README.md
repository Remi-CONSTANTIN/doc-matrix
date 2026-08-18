# Matrix Ansible - Guides et Documentation

Bienvenue sur ce dépôt de documentation dédié à l'auto-hébergement de l'écosystème **Matrix** (Synapse, Element, LiveKit, Ketesa) via Docker et Ansible.

Vous y trouverez des tutoriels pas-à-pas, des guides de sécurisation avancée et des astuces d'optimisation pour déployer, protéger et maintenir une instance de messagerie souveraine, robuste et performante.

## Sommaire et Navigation

### Déploiement Initial
* [Tutoriel de Déploiement Matrix via Ansible](Tutoriel_Deploiement_Matrix_via_Ansible.md) : Le guide de démarrage rapide (Quick Start) pour installer la stack de base de A à Z (Chat textuel + Appels vidéo complets) avec un hardening léger.

### Sécurisation (Hardening)
* [[HARDENING] CrowdSec]([HARDENING]%20crowdsec_matrix.md) : Détecter et bloquer en temps réel les attaques dirigées contre les conteneurs Matrix grâce à CrowdSec et son bouncer pare-feu.
* [[HARDENING] Whitelist de Fédération]([HARDENING]%20domain_whitelist_federation_matrix.md) : Restreindre la fédération pour contrôler précisément avec quels serveurs externes votre instance est autorisée à communiquer.

### Améliorations & Confort (Quality of Life - QoL)
* [[QOL] Auto Clean Matrix]([QOL]%20auto_clean_matrix.md) : Stratégies de maintenance pour purger automatiquement l'historique et les médias afin d'économiser de l'espace disque.
* [[QOL] URL Preview]([QOL]%20url_preview_matrix.md) : Activer et configurer proprement la prévisualisation des liens web dans les salons de discussion.

---

## Prérequis Techniques

Pour suivre ces guides, il est recommandé de disposer de :
* Un serveur (ou VM) sous Linux (de préférence Debian)
* Un nom de domaine avec accès à la gestion de la zone DNS
* Un accès à son routeur pour la redirection de ports
* Des notions de base en ligne de commande Linux, Docker et Ansible

## Sources et Crédits

* Le socle technique repose sur l'excellent playbook de la communauté : [matrix-docker-ansible-deploy](https://github.com/spantaleev/matrix-docker-ansible-deploy).
* Certaines parties de la documentation rédigée à l'aide de Gemini mais elle reste testée, corrigée et maintenue par **Rémi CONSTANTIN**.
