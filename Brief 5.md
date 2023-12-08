# DNS et TLS

## Description de l'infrastructure

Dans un resource groupe on déploie un vnet, un subnet, un keyvault, une application gateway et une machine virtuel. La machine Virtuel héberge une application "Nextcloud" et elle sera désigné comme backend dans l'application gateway. Il y a une adresse IP publique attachée à la gateway.

## Création du sous domaine

Un nom de domaine nous a été fournie auprés de Gandi.net ( simplon-lion.space)

- On relève l'adresse IP publique de la gateway
- on se rend dans l'interface de gestion de notre domaine sur Gandi.net.
- Dans la section DNS, on va ajouter un enregistrement de type A avec le sous domaine nextcloud.simplon-lion.space pour l'Ip publique qui nous a été attribuée

On reste dans l'interface de gestion et on va générer notre API key qui nous sera nécessaire pour le certificat TLS dans la section suivante.

## Création du certificat

1. installation du plugin cerbot pour Gandi

        ```bash
         sudo apt-get update
         sudo apt-get upgrade
         sudo apt install software-properties-common -y
         sudo add-apt-repository ppa:deadsnakes/ppa -y
         sudo apt install python3.10
         sudo apt-get update
         sudo apt-get upgrade
         sudo pip3 install certbot-plugin-gandi
        ```

2. Génération du certificat

   - dans un fichier gandi.ini stocker la clef d'API

    ```bash
    vim ~/gandi.ini # copier la clef et la coller dans le fichier
    :wq # enregistrer et quitter
    sudo chmod 600 gandi. ini # donner les droit d'execution sur le fichier.
    ```

    ```bash
    sudo certbot certonly --authenticator dns-gandi --dns-gandi-credentials ~/gandi.ini -d nextcloud.simplon-lion.space
    cd /etc/letsencrypt/live/nextcloud.simplon-lion.space
    cat cert.pem privkey.pem > certificat.pem
    mv certificat.pem ~/paul
    openssl pkcs12 -export -out ~/certificat.pfx -in ~/certificat.pem
    ```

3. Ajout du Listerner pour le TLS

    - Nom de l'écouteur -> TLS
    - Adresse ip du front-end -> Gateway Public IP
    - Protocole -> HTTPS
    - Port -> 443
    - Sélectionner un certificat -> Créer
    - Paramèters HTTPS -> Télécharger un certificat ->  Nom du certificat: nextcloud ->  Fichier du certificat PFX -> ~/certificat.pfx -> password: "6Euqj%5ùwow"

4. Ajouter des règles -> Règles d'acheminement

    - rule-1
       - Nom de la règle : rule-1
       - Priorité** : 100
       - Cibles de back-en
       - Type de cible : Redirection
       - Type de rediction : Permanent
       - Cible de rediction : HTTPS
       - Inclure la chaîne de requête : OUI
       - Inclure le chemin : OUI

    - rule-2
       - Nom de la règle : rule-2
       - Priorité** : 103
       - Listerner -> HTTPS
       - Cibles de back-end
       - Type de cible : Pool principal
       - Cible de back-end : backend-pool
       - Paramètres de back-end : https-setting

5. Charger le certificat dans Azure KeyVault

    - Importer le fichier contenant le certificat et la clé dans le KeyVault en utilisant la CLI

    ```bash
    az keyvault certificate import --vault-name keyvaultrajach -n certif -f ~/certificat.pfx --password 6Euqj%5%wow
    ```

    - Vérifier que le certificat est dans le KeyVault en utilisant la CLI :
  
    ```bash
    az keyvault certificate show -v <vault-name> -n <certificate-name>
    ```
