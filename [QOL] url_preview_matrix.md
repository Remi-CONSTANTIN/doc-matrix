# Guide de mise en place des preview (miniatures) pour les liens youtube, Github etc...
Ce rapide guide porte sur l'activation des miniatures sous les liens menant vers des plateformes externes.

# Sécurité
Cette fonctionnalité peut représentée une porte d'entrée pour de potentiels pirates à cause du fonctionnement de celle-ci et même si elle vient accompagnée d'une liste noir d'URL (192.168.x.x, 10.x.x.x, localhost et plus)
En effet, c'est le serveur Synapse lui-même qui va aller se connecter à l'URL envoyée dans la discussion.  
Les risques portent sur le pistage, l'épuisement des ressources et l'[attaque SSRF](https://www.wandesk.fr/attaque-ssrf-definition-et-prevention/)

**Il est donc conseillé de n'activer cette fonctionnalité que si vous avez confiance en vos utilisateurs et que vous maîtrisez les inscriptions**

---

# Mise en place

## Configuration du serveur
1. Connectez vous en SSH à votre machine Matrix puis allez dans le répertoire de travail  
Dans mon cas : `cd /opt/matrix-docker-ansible-deploy`

2. Entrez dans votre fichier de configuration  
`nano inventory/host_vars/matrix.votredomaine.com/vars.yml`  

3. Ajoutez la variable
```
matrix_synapse_url_preview_enabled: true
```

4. Quittez le fichier et relancer le playbook pour appliquer les changements  
`ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start`

5. Pour vérifier que la blacklist ai bien été mise en place, vous pouvez exécutez cette commande
```
grep -A 15 "url_preview_ip_range_blacklist" /matrix/synapse/config/homeserver.yaml
```

## Configuration des clients
Par défaut, l'aperçus sur les liens n'est pas activée dans les salons chiffrés, il vous faudra donc l'activer en fonction de votre type d'appareil

### Client Element sur Ordinateur (testé sur Fedora)
1. Cliquez sur votre photo de profil en haut à gauche
2. Allez dans "Tout les paramètres"
3. Dans "Préférences"
4. Descendez dans la partie "Aperçus des liens" et activez "Activez les aperçus dans les salons chiffrés"

### Application Element X sur téléphone (testé sur Android)
A date du 17/08/2026, cette fonctionnalité n'est pas disponible car l'application est encore jeune.  
Elle devrait être disponible sur l'application d'origine "Element Classique" ou sur les clients alternatifs (FluffyChat, SchildiChat etc...)

---

# Sources
[Documentation Officielle Matrix Ansible](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/roles/custom/matrix-synapse/defaults/main.yml)
