# Guide de Sécurisation : Contrôler les domaines autorisés à se fédérer avec votre instance
Ce guide porte sur la mise en place du whitelisting de domaines précis dans le cas où votre instance ne peut être fédéré que par certains domaines.  

Utile dans le cas où vous souhaiteriez limiter la fédération aux instances de votre entourage ou de personnes en qui vous avez confiance.


# Mise en place

1. Connectez vous en SSH à votre machine Matrix puis allez dans le répertoire de travail  
Dans mon cas : `cd /opt/matrix-docker-ansible-deploy`

2. Entrez dans votre fichier de configuration  
`nano inventory/host_vars/matrix.tadaron.fr/vars.yml`  

3. Ajoutez la variable, votre domaine et le(s) potentiel(s) domaine(s) approuvé(s)
```
matrix_synapse_federation_domain_whitelist:
- votredomaine.com
- exemple.com
- exemple.com
```

4. Quittez le fichier et relancer le playbook pour appliquer les changements  
`ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start`

Si vous avez décidé d'approuver `matrix.org`, vous pouvez tester la fédération avec l'outil officiel : [Matrix Federation Tester](https://federationtester.matrix.org/)
