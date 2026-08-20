# Guide complet : Intercaler un Reverse Proxy (Nginx Proxy Manager) devant Matrix

# Problématique
Si vous avez suivi le tutoriel classique de déploiement de Matrix via Ansible, vous avez remarqué que les ports 80 et 443 de votre box Internet sont accaparés par la machine Matrix (ils sont redirigés directement vers elle).
Or, cela vous empêche d'exposer d'autres services web (comme un site personnel, un cloud, etc.) sur la même adresse IP publique, puisque la porte est monopolisée par Matrix.

# Objectif
L'objectif est de placer un Nginx Proxy Manager (NPM) en première ligne. C'est lui qui recevra tout le trafic web provenant d'internet (ports 80 et 443) et qui le distribuera intelligemment vers Matrix ou vers vos autres services en fonction du nom de domaine demandé.

# Besoins techniques & Prérequis
- Un serveur Matrix fonctionnel, déployé via le playbook Ansible officiel, sur Debian 13
- Un serveur Nginx Proxy Manager fonctionnel, sur Debian 13
- Les deux machines sont situées dans une DMZ (ou un réseau local sécurisé)
- Un nom de domaine (ex: votredomaine.com)
- Avoir fait un Snapshot (sauvegarde) de vos machines avant de commencer !

---

# Mise en place

## Étape 1 : Configuration du Routeur et du Pare-feu
Votre routeur et votre hyperviseur (Proxmox) doivent s'adapter au nouveau rôle du proxy.

1. Sur votre Box Internet / Routeur :
- Ports 80 et 443 : Redirigez-les désormais vers l'adresse IP de votre machine Nginx Proxy Manager (et non plus vers Matrix)
- Port 8448 (Fédération) : Supprimez la redirection. NPM gérera la fédération de manière transparente via le port 443.
- Ports Coturn (3478, 5349, 7881, 7882 et 35000+) : Gardez ces redirections directement vers l'adresse IP de votre VM Matrix (le proxy ne gère pas bien ce trafic UDP/WebRTC pour les appels).

2. Sur le Pare-feu Proxmox (de la VM Matrix) :
- Assurez-vous de créer une règle ACCEPT tout en haut de votre pare-feu pour autoriser l'adresse IP de votre machine NPM à communiquer avec les ports 80 et 8448 de la VM Matrix.

## Étape 2 : Adaptations sur le serveur Matrix

Puisque le proxy frontal va désormais gérer les connexions, Matrix doit arrêter de se comporter comme s'il était seul au monde

1. Liste blanche CrowdSec
Dans votre fichier `vars.yml` (Ansible), assurez-vous d'ajouter l'adresse IP de votre machine NPM à la liste blanche de CrowdSec. Sans cela, CrowdSec bannira votre proxy au premier transfert de requêtes suspectes.

2. Réparer le DNS de Docker (Fédération sortante)
Pour éviter l'erreur "Erreur de serveur inconnue" lors de l'invitation de membres de serveurs externes, forcez Docker à utiliser un DNS public :  
   a. Sur la VM Matrix, éditez le fichier : `sudo nano /etc/docker/daemon.json`  
   b. Ajoutez : `{"dns": ["1.1.1.1", "8.8.8.8"]}`  
   c. Redémarrez Docker : `sudo systemctl restart docker`  

## Étape 3 : Configuration de Nginx Proxy Manager
C'est ici que la magie opère. Dans l'interface web de NPM, vous devez créer trois Proxy Hosts.

**Pour les trois, activez "Force SSL" et générez un certificat Let's Encrypt)**

1. Le client web (Element)
- `Domain Names` : element.votredomaine.com
- `Scheme` : http
- `Forward Hostname / IP` : [IP_DE_VOTRE_VM_MATRIX]
- `Forward Port` : 80
- Rien à ajouter dans l'onglet `Advanced`

2. Le domaine racine (Le panneau indicateur)
Pour que la fédération vous trouve, le domaine racine doit héberger le fichier .well-known.

- `Domain Names` : votredomaine.com
- `Scheme` : http
- `Forward Hostname / IP` : L'adresse IP de votre site web principal (ou une IP bidon si vous n'avez pas de site).
- `Forward Port` : 80
- `Onglet "Advanced"` : Collez ce code pour déléguer la gestion Matrix au bon sous-domaine :

```
location /.well-known/matrix/server {
    add_header Access-Control-Allow-Origin *;
    default_type application/json;
    return 200 '{"m.server": "matrix.votredomaine.com:443"}';
}

location /.well-known/matrix/client {
    add_header Access-Control-Allow-Origin *;
    default_type application/json;
    return 200 '{"m.homeserver": {"base_url": "https://matrix.votredomaine.com"}}';
}
```

3. Le serveur Matrix (Trafic principal et Fédération)
C'est la règle la plus importante. Elle redirige le trafic classique vers le port 80, et trie le trafic de fédération pour l'envoyer au port 8448 de Traefik en texte clair.

- `Domain Names` : matrix.votredomaine.com
- `Scheme` : http
- `Forward Hostname / IP` : [IP_DE_VOTRE_VM_MATRIX]
- `Forward Port` : 80
- `Onglet "Advanced"` : Collez ce code :

```
client_max_body_size 50M;

location /_matrix/federation {
    proxy_pass http://[IP_DE_VOTRE_VM_MATRIX]:8448;
    proxy_set_header Host matrix.votredomaine.com;
}

location /_matrix/key {
    proxy_pass http://[IP_DE_VOTRE_VM_MATRIX]:8448;
    proxy_set_header Host matrix.votredomaine.com;
}
```

## Étape 4 : Validation

1. Ouvrez votre navigateur et allez sur [https://element.votredomaine.com](https://element.votredomaine.com). L'interface de connexion doit s'afficher
2. Allez sur le site officiel Matrix Federation Tester --> Entrez `votredomaine.com` et lancez le test --> Le résultat final doit afficher un beau `FederationOK: true` !
