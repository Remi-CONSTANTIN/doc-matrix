# Guide de Sécurisation : Protéger une Stack Matrix / Docker avec CrowdSec

Ce guide détaille la mise en place de CrowdSec sur un hôte Debian pour surveiller, détecter et bloquer en temps réel les attaques dirigées contre votre infrastructure Matrix auto-hébergée sous Docker (Traefik, Synapse, Element, LiveKit).

# Disclaimer
Tutoriel rédigé à l'aide de Gemini puis revu + testé + corrigé par Rémi CONSTANTIN

---

# Mise en place

## Étape 0 : Installation Crowdsec

1. Installer le dépôt CrowdSec sur la machine à protéger
```
curl -s https://install.crowdsec.net | sudo sh
```

2. Installer CrowdSec
```
apt install crowdsec
```

3. Si vous aviez déjà des services sur la machine lors de l'installation de CrowdSec, alors il est possible que des collections aient automatiquement été téléchargées.
Pour vérifier cela : 
```
sudo cscli collections list
```

4. Pour finir l'installation , je vous recommande vivement d'installer tout de suite le "Bouncer" iptables afin que CrowdSec puisse bannir automatiquement s'il détecte une activité suspecte
```
sudo apt install crowdsec-firewall-bouncer-iptables -y
```

## Étape 1 : Activation des logs d'accès Traefik via Ansible

Pour que CrowdSec puisse analyser le trafic HTTP(S), Traefik doit consigner chaque requête dans un fichier de log accessible sur l'hôte.

1. Préparer le dossier sur l'hôte
Le conteneur Traefik tourne sous un utilisateur non-root. Attribuez les permissions requises pour lui permettre de créer et d'écrire dans le fichier :
```
mkdir -p /matrix/traefik/logs
chmod -R 777 /matrix/traefik/logs
```

2. Configurer vars.yml
Ouvrez le fichier de configuration Ansible :
```
nano /opt/matrix-docker-ansible-deploy/inventory/host_vars/matrix.votredomaine.com/vars.yml
```

Ajoutez ces variables à la fin du fichier :
```
# --- GESTION DES LOGS D'ACCÈS TRAEFIK (CrowdSec) ---
traefik_config_accessLog_filePath: "/logs/access.log"
traefik_config_accessLog_format: "json"
traefik_config_accessLog_bufferingSize: 100
traefik_container_extra_arguments:
  - "--volume=/matrix/traefik/logs:/logs:rw"
```

3. Appliquer la configuration

Déployez les modifications avec Ansible :
```
cd /opt/matrix-docker-ansible-deploy
LC_ALL=C.UTF-8 ansible-playbook -i inventory/hosts setup.yml --tags=setup-traefik,start
```

4. Vérifier la génération des logs

Générez du trafic en ouvrant [https://element.votredomaine.com](https://element.votredomaine.com) dans votre navigateur, puis vérifiez sur le serveur :
```
tail -f /matrix/traefik/logs/access.log
```
(Appuyez sur Ctrl + C pour quitter dès que vous voyez défiler les lignes au format JSON).

---

## Étape 2 : Configuration du moteur CrowdSec (Hôte)
1. Installer les collections et scénarios de sécurité

Installez les parseurs et scénarios de détection pour Traefik et les attaques web :
```
# Parseur Traefik et règles de base
cscli collections install crowdsecurity/traefik
cscli collections install crowdsecurity/http-cve
cscli collections install crowdsecurity/whitelist-good-actors

# Détection proactive des scans d'URL et probes
cscli scenarios install crowdsecurity/http-probing
cscli scenarios install crowdsecurity/http-path-traversal-probing
```

2. Déclarer la source de logs Traefik

Indiquez à CrowdSec de lire le fichier partagé :
```
nano /etc/crowdsec/acquis.yaml
```

Ajoutez ce bloc à la fin du fichier :
```
filenames:
  - /matrix/traefik/logs/access.log
labels:
  type: traefik
```

3. Redémarrer CrowdSec
```
systemctl restart crowdsec
```

4. Contrôler l'acquisition

Vérifiez que CrowdSec lit et comprend les logs :
```
cscli metrics
```
(Dans la section Acquisition Metrics et Parser Metrics, le compteur traefik doit s'incrémenter).

---

## Étape 3 : Configuration du Bouncer Pare-feu (Protection Docker)

Le moteur CrowdSec détecte les attaques, mais c'est le Firewall Bouncer qui applique la sanction au niveau du noyau Linux.

1. Installation du Bouncer
Si ce n'est pas déjà fait :
```
apt install -y crowdsec-firewall-bouncer-nftables
```

2. Protéger les conteneurs Docker (DOCKER-USER)
Par défaut, le bouncer ne filtre que la chaîne INPUT. Pour intercepter et bloquer le trafic vers vos conteneurs Docker, éditez sa configuration :
```
nano /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```
Assurez-vous que les chaînes FORWARD et DOCKER-USER sont bien décommentées :
```
iptables_chains:
  - INPUT
  - FORWARD
  - DOCKER-USER
```

3. Redémarrer le bouncer
```
systemctl restart crowdsec-firewall-bouncer
```

4. Vérification de la chaîne pare-feu

Vérifiez que la règle CrowdSec est activement branchée sur Docker :
```
iptables -L DOCKER-USER -n -v
```
(Vous devez voir la cible CROWDSEC_CHAIN avec des paquets traités).

---

## Étape 4 : Tests & Validation en Conditions Réelles
A. Test de blocage manuel (Simulation)

Récupérez l'IP de votre smartphone en 4G/5G sur mon-ip.com.
Bannissez temporairement cette IP :
```
cscli decisions add --ip VOTRE_IP_TEST --duration 5m --reason "Test manuel"
```

Essayez d'accéder à [https://element.votredomaine.com](https://element.votredomaine.com) depuis le smartphone : la connexion doit immédiatement tomber en Timeout / Échec de connexion.

Débannissez l'IP :
```
cscli decisions delete --ip VOTRE_IP_TEST
```

B. Test d'attaque automatisée (Scan de vulnérabilités)

Depuis une machine distante (ou votre PC en partage de connexion 4G, jamais depuis le serveur) :
```
for i in {1..20}; do curl -s -o /dev/null -w "%{http_code}\n" "https://element.votredomaine.com/test-faille-$i.php"; done
```

Sur le serveur :
**Vérifiez l'alerte générée** : `cscli alerts list`
**Vérifiez le ban automatique** : `cscli decisions list`
**Débannissez votre machine de test une fois validé** : `cscli decisions delete --ip <IP>`

## Piste d'amélioration
Ajouter cette machine au dashboard Crowdsec Web officiel
