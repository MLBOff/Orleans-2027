## Contexte 

Dans un environnement StackWise, chaque switch possède un niveau de priorité. Ce niveau permet de déterminer quel switch doit avoir le rôle principal dans la pile.

## Attribution de la priorité

En mode de configuration, attribuer la priorité maximale `15` au switch 1 :

```text
switch(config)# switch 1 priority 15
```

Vérifier le niveau de priorité avec :

```text
switch# show switch
```

Vérifier le numéro `15` dans la colonne **Priority** du switch 1.
## Explication 

Les anciens équipements Cisco peuvent utiliser des algorithmes SSH qui ne sont plus activés par défaut dans les versions récentes d'OpenSSH.

Dans ce cas, il est possible de créer une configuration SSH spécifique afin d'autoriser les anciens algorithmes nécessaires à la connexion au switch.

## Création du script pour les anciens algorithmes SSH

Création du fichier de configuration dans `~/.ssh/` :

```bash
nano ~/.ssh/config
```

Ajouter le script :

```text
Host switch
    HostName 10.140.0.1
    User NAME
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
```

Attribution des permissions au fichier :

```bash
chmod 600 ~/.ssh/config
```

Pour se connecter au switch en SSH :

```bash
ssh ****(host)
```

Grace a ce script l'utilisation de ssh sur le routeur est possible.