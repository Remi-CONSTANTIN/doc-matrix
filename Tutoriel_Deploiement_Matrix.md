# Tutoriel Complet : Auto-héberger Matrix, Element & LiveKit avec Ansible (Édition 2026)

Ce guide permet de déployer un serveur Matrix complet sur une machine Debian en utilisant Docker et Ansible. Il comprend :
*   **Synapse** : Le cœur du serveur de messagerie
*   **Element Web** : L'interface web de messagerie
*   **Ketesa** : L'interface graphique d'administration (utilisateurs, salons, fédération etc...)
*   **LiveKit** : Le serveur de visioconférence WebRTC (Element Call)

# Disclaimer
Tutoriel rédigé à l'aide de Gemini puis revu et corrigé par moi-même

---
<br><br>

## Phase 1 : Préparation du Réseau et des DNS (L'étape cruciale)

La majorité des erreurs d'installation proviennent d'une mauvaise configuration réseau. Avant de toucher au serveur Debian, préparez le terrain :

### 1. Gestion de l'IP (Spécial Freebox / CGNAT)
Vérifiez que vous disposez d'une IP publique complète (tous les ports disponibles). Chez Free, vous devez impérativement demander une **"Adresse IP fixe V4 full-stack"** depuis votre espace abonné, sinon les ports d'entrée seront bloqués par l'opérateur.

### 2. Configuration des DNS (Chez votre hébergeur)
Vous devez créer **3 enregistrements** pour lier votre domaine à votre IP publique :
*   **Enregistrement A** pour `@` (le domaine racine, ex: `tadaron.fr`) ➔ *Nécessaire pour l'authentification des appels vidéo*
*   **Enregistrement A** (ou CNAME) pour `matrix` (ex: `matrix.tadaron.fr`) ➔ *Le serveur principal*
*   **Enregistrement A** (ou CNAME) pour `element` (ex: `element.tadaron.fr`) ➔ *L'interface web*

> [!warning]
> Une modification DNS peut prendre de 15 minutes à quelques heures pour se propager. vérifiez avec la commande `ping votre-domaine.fr` que c'est bien votre IP publique qui vous retourne le ping


### 3. Ouverture des ports (Redirection NAT sur la box)
Dans l'interface de votre box internet, redirigez les ports suivants vers **l'adresse IP locale** de votre serveur Debian (ex: `10.168.90.231`) :

**Ports standards (Web & Fédération) :**
*   `80` (TCP) : Trafic web initial et validation des certificats Let's Encrypt.
*   `443` (TCP) : Trafic web sécurisé HTTPS.
*   `8448` (TCP) : Communication entre les différents serveurs Matrix (Fédération).

**Ports LiveKit (Visioconférence) :**
*   `7881` (TCP) & **7882** (UDP) : Gestion des flux WebRTC.
*   `3479` (UDP) & **5350** (TCP) : Négociation STUN/TURN.
*   `30000` à `30020` (UDP) : Plage pour la transmission directe de l'audio et de la vidéo.

### 4. Contournement du blocage réseau local (Hairpin NAT)

Si vous hébergez votre serveur chez vous derrière une box internet grand public, la box détruira les requêtes de vos conteneurs (LiveKit) tentant de s'authentifier via votre propre IP publique (Erreur OPEN_ID_ERROR / Timeout).
Pour forcer le serveur à traiter ces requêtes en interne sans passer par la box :

Attribuez virtuellement votre IP publique à la boucle locale de votre serveur (remplacez 82.67.x.x par votre VRAIE IP publique) :
```
ip addr add 82.67.x.x/32 dev lo
```
Rendez cette modification permanente après chaque redémarrage de la machine avec une tâche planifiée (cron) :
```
(crontab -l 2>/dev/null; echo "@reboot /sbin/ip addr add 82.67.x.x/32 dev lo") | crontab -
```

---
<br><br>

## Phase 2 : Préparation du serveur Debian

Connectez-vous en `root` sur votre machine Debian.

1.  Installez les prérequis :
    ```
    apt update && apt install -y git ansible
    ```
2.  Clonez le projet Ansible :
    ```
    git clone [https://github.com/spantaleev/matrix-docker-ansible-deploy.git](https://github.com/spantaleev/matrix-docker-ansible-deploy.git)
    cd matrix-docker-ansible-deploy
    ```
3.  Configurez l'inventaire local :
    A. Nous exécutons Ansible directement sur la machine pour éviter les problèmes de droits SSH.
    ```
    mkdir -p inventory/host_vars/matrix.votre-domaine.fr
    nano inventory/hosts
    ```
    B. Insérez ce texte en adaptant le nom de domaine :
    ```
    [matrix_servers]
    matrix.votre-domaine.fr ansible_connection=local
    ```

---
<br><br>

## Phase 3 : Le fichier de configuration (`vars.yml`)

1. C'est le cœur de votre serveur, là où l'on définit son comportement.

```
nano inventory/host_vars/matrix.votre-domaine.fr/vars.yml
```

2. Copiez-collez cette configuration type en remplaçant les valeurs :

```
# --- 1. CONFIGURATION DE BASE ---
matrix_domain: votre-domaine.fr
matrix_playbook_reverse_proxy_type: playbook-managed-traefik

# --- 2. SÉCURITÉ ---
# Générez des mots de passe complexes et uniques pour ces deux variables
matrix_homeserver_generic_secret_key: "UNE_CLE_TRES_LONGUE_ET_SECRETE"
postgres_connection_password: "UN_MOT_DE_PASSE_BDD_COMPLEXE"

# --- 3. VALIDATION DE L'INSTALLATION ---
matrix_playbook_migration_validated_version: "v2026.05.18.0"

# --- 4. INSCRIPTIONS (Ouverture temporaire pour la création du 1er compte) ---
matrix_synapse_enable_registration: true
matrix_synapse_enable_registration_without_verification: true

# --- 5. INTERFACE D'ADMINISTRATION GRAPHIQUE ---
matrix_ketesa_enabled: true

# --- 6. VISIOCONFÉRENCE (LiveKit) & DÉLÉGATION ---
matrix_rtc_enabled: true
# Indispensable pour éviter l'erreur OPEN_ID_ERROR lors des appels (gère le /.well-known/)
matrix_static_files_container_labels_base_domain_enabled: true
```

---
<br><br>

## Phase 4 : Installation et Déploiement

Toujours depuis le dossier `/opt/matrix-docker-ansible-deploy`, lancez l'installation :

1. Téléchargement des dépendances Ansible :
```
ansible-galaxy install -r requirements.yml --roles-path ./roles/galaxy
```
Lancement du déploiement général :
```
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```
(Laissez tourner, le script télécharge les images Docker, configure Traefik, génère les certificats et lance les services. Cela prend plusieurs minutes et dépendra de votre connexion internet)

---
<br><br>

## Phase 5 : Création de l'Administrateur & Verrouillage

Une fois l'installation terminée, créez votre compte "maître" en ligne de commande :
```
ansible-playbook -i inventory/hosts setup.yml --tags=register-user \
-e "username=VOTRE_PSEUDO" \
-e "password=VOTRE_MOT_DE_PASSE" \
-e "admin=yes"
```

🔒 Verrouillage du serveur :
Il faut immédiatement fermer les inscriptions pour protéger votre serveur des spams et robots. Vous gérez vos utilisateurs via l'interface Web Ketesa.

Éditez votre fichier vars.yml et passez les deux lignes d'inscription sur false :
```
matrix_synapse_enable_registration: false
matrix_synapse_enable_registration_without_verification: false
```

Appliquez le verrouillage sur le composant Synapse :
```
ansible-playbook -i inventory/hosts setup.yml --tags=setup-synapse,start
```

---
<br><br>

## Phase 6 : (Optionnel) Sécuriser l'accès à l'administration par IP

Par défaut, l'interface d'administration Ketesa (/synapse-admin/) est accessible publiquement avec identifiant/mot de passe. Pour bloquer l'accès à toute personne extérieure et restreindre cette page à vos seules adresses IP de confiance (votre box, votre VPN, votre réseau local) :

1. Ouvrez le fichier de configuration de Traefik :
```
nano /matrix/traefik/config/provider.yml
```

2. Remplacez l'intégralité du contenu par ce bloc (adaptez les adresses IP dans sourceRange) :
```
http:
  middlewares:
    compression:
      compress:
        encodings: zstd,br,gzip
        minResponseBodyBytes: 1024
    ketesa-ip-guard:
      ipAllowList:
        sourceRange:
          - "82.15.86.153"       # Votre IP publique fixe
          - "10.0.0.0/8"         # Exemple Réseau local 1
          - "192.168.0.0/16"     # Exemple Réseau local 2
          - "172.16.0.0/12"      # Exemple Réseau local 3

  routers:
    ketesa-secure-override:
      rule: "Host(`matrix.votre-domaine.fr`) && PathPrefix(`/synapse-admin`)"
      entryPoints:
        - "web-secure"
      service: "matrix-ketesa@docker"
      tls: {}
      priority: 1000
      middlewares:
        - "ketesa-ip-guard"
        - "matrix-ketesa-slashless-redirect@docker"
        - "matrix-ketesa-strip-prefix@docker"
        - "matrix-ketesa-add-headers@docker"

tcp:
  serversTransports:
    proxy:
      proxyProtocol:
        version: 1
```

3. Redémarrez Traefik pour appliquer la restriction immédiatement :
```
docker restart matrix-traefik
```

---
<br><br>

## Phase 7 : Accès et Utilisation

Votre infrastructure est opérationnelle. Les certificats HTTPS sont gérés automatiquement par Traefik.

💬 Pour discuter (L'application de messagerie) : Allez sur https://element.votre-domaine.fr.

⚙️ Pour administrer (Ketesa) : Allez sur https://matrix.votre-domaine.fr/synapse-admin/ pour ajouter vos amis, modifier des mots de passe ou gérer vos salons graphiquement. (Note : L'URL du "Homeserver" à renseigner lors du login est https://matrix.votre-domaine.fr).

📞 Pour les appels vidéo : Ils fonctionnent nativement dans les salons. Pour trouver un ami et démarrer une conversation, utilisez son identifiant complet (ex: @jean:votre-domaine.fr).

---
<br><br>

## Phase 8 : (Fortement conseillée) Sécurisation de la machine
- Placer cette machine dans la DMZ (grand minimum)
- Déployer crowdsec sur la machine pour profiter des listes noirs d'IP publiques
- Restreindre les risques de déplacement latéraux sur le réseau local à l'aide d'un pare-feux local comme Proxmox firewall (UFW n'est pas conseillé car, par défaut, docker bypass ses règles) dans le cas d'une infection de la machine
