# Guide de nettoyage automatique de votre instance Matrix
Il est vu ici la mise en place de deux processus de nettoyage automatique dans l'optique d'économiser du stockage sur votre machine.
- Le premier concerne la rétention des médias locaux et distants
- Le deuxième vient compresser la base de données

En effet, dans l'optique d'éviter au maximum de surcharger le stockage de votre machine, il est conseillé de ne pas garder indéfiniment les photos et fichiers stockés dans les salons/discussions.
Attention, le nettoyage des médias **locaux** les supprime définitivement des disques alors que les médias distants seront re-téléchargés depuis le Matrix distant lors qu'ils seront re-consultés.

# Mise en place
Comme toujours, tout se passe dans le dossier `/opt/matrix-docker-ansible-deploy/`  
Et dans le fichier `inventory/host_vars/matrix.votredomaine.com/vars.yml`

## 1. Médias
Ajoutez la nouvelle variables et adaptez la rétention à votre convenance
```
matrix_synapse_configuration_extension_yaml: |
  media_retention:
    local_media_lifetime: 180d # 6 mois
    remote_media_lifetime: 30d # 1 mois
```

## 2. Base de données
Ajoutez la nouvelle variables et adaptez la planification à votre contexte
```
matrix_synapse_auto_compressor_enabled: true
matrix_synapse_auto_compressor_schedule: "*-*-* 14:00:00" # Heure de base : tous les jours à 14h00
matrix_synapse_auto_compressor_schedule_randomized_delay_sec: "1800" # Délai aléatoire court : 1800 secondes
```
Dans cet exemple, le nettoyage sera déclenché entre `14:00` et `14:30` grâce au `randomized_delay`


## 3. Application des changements
Pour appliquer les changements, c'est toujours la même chose  
`ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start`

---

# Sources
[Media Retention](https://element-hq.github.io/synapse/latest/usage/configuration/config_documentation.html?highlight=media_reten#media_retention)
[synapse-auto-compressor](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-synapse-auto-compressor.md)
