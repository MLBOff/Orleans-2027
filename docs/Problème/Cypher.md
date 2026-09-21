## Contexte 

Dans un environnement Cisco StackWise, les différents commutateurs constituant la pile doivent normalement disposer d'un niveau de sécurité et d'une configuration homogènes, notamment concernant les paramètres d'accès SSH.

Cependant, certains équipements Cisco anciens utilisent des algorithmes cryptographiques SSH obsolètes. Ces algorithmes, considérés aujourd'hui comme insuffisamment sécurisés, sont désactivés par défaut dans les versions récentes d'OpenSSH.

Par conséquent, une tentative de connexion SSH classique peut échouer, même si le service SSH est correctement configuré sur le switch. Le client OpenSSH refuse alors les algorithmes proposés par l'équipement Cisco.
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
    HostName 'ip'
    User NAME
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
```
Script pour le routeur :
```
Host routeur
    HostName 'ip'
    User NAME
    KexAlgorithms +diffie-hellman-group14-sha1
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

Grace a ce script l'utilisation de ssh sur le switch et/ou le routeur est possible.