# Guide de configuration : AJouter un proxy indépendant devant Matrix 

Ce guide détaille la configuration à mener sur Matrix après l'ajout d'un Proxy entre Matrix et le Web.

# Problématique
Si vous avez suivis mon tutoriel sur le déploiement de Matrix, vous vous serez peut-être rendu compte que les ports 80 et 443 de votre routeur sont accaprés par Matrix car on les redirige directement vers celui-ci.  
Or, ce n'est pas ce que nous voulons, car cela bloque la possible exposition de nouveaux services utilisants ces deux ports (un site web par exemple).  

# Objectif
L'objectif est donc d'adapter matrix et de configurer le nouveau proxy frontal

# Besoins techniques
- Un serveur Matrix déployé avec Ansible et la documentation officielle Matrix (ma procédure se base dessus) fonctionnant sur Debian 13
- Un serveur Nginx Proxy manager fonctionnant sur Debian 13
- Les deux machines sont dans la DMZ
- Un domaine à disposition (normalemenr déjà le cas si vous avez déjà un serveur Matrix)

---

# En pratique

- Snapshot (éteinte ou pas)
- Allumer machine Matric
- changer les redirectin de ports vers le Proxy (80 et 443)
- Configurer les redirections vers matrix  dans Nginx proxy manager
1. element.votredomaine.com --> IP:443 Matrix ?
2. 
- Changer la conf Matrix et relancer le playbook 
