# Configuration d'un routeur  

## Création d'un utilisateur pour SSH

Pour créer un utilisateur local permettant de se connecter au switch en SSH :

```
username NOM privilege 15 secret MDP
```
### Explication

| Élément        | Description                              |
| -------------- | ---------------------------------------- |
| `username NOM` | Crée l'utilisateur                       |
| `privilege 15` | Attribue le niveau de privilège maximal  |
| `secret MDP`   | Définit le mot de passe de l'utilisateur |


Il est recommandé d'utiliser `secret` plutôt que `password`, car le mot de passe est stocké de manière plus sécurisée dans la configuration.

---

## Configuration de SSH

Une configuration SSH complète peut être réalisée avec les commandes suivantes :

```bash
hostname RO-01

ip domain-name exemple.local

username NOM privilege 15 secret MDP

crypto key generate rsa modulus 2048

ip ssh version 2

line vty 0 15
 login local
 transport input ssh
```

### Explication

| Commande                               | Description                           |
| -------------------------------------- | ------------------------------------- |
| `hostname RO-01`                       | Définit le nom du switch              |
| `ip domain-name exemple.local`         | Définit le nom de domaine             |
| `username NOM privilege 15 secret MDP` | Crée l'utilisateur local              |
| `crypto key generate rsa modulus 2048` | Génère les clés RSA nécessaires à SSH |
| `ip ssh version 2`                     | Active SSH version 2                  |
| `line vty 0 15`                        | Sélectionne les lignes VTY            |
| `login local`                          | Utilise les utilisateurs locaux       |
| `transport input ssh`                  | Autorise les connexions SSH           |

---

### Attribution d'une IP

Pour pouvoir administrer le routeur à distance, une adresse IP doit être configurée sur une interface accessible depuis le réseau d'administration.


```
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```


### Test connexion SSH

Une fois l'utilisateur crée et une IP attribuée sur une interface du routeur dans le même réseau. On essaye de se connecter en SSH
 
```
ssh NOM@ip 
```

Si en essayant de vous connecter la reponse est : 
```
Unable to negotiate with 'ip' port 22: no matching key exchange method found. Their offer: diffie-hellman-group1-sha1
```

voir [Cypher](Problème/Cypher.md)

Après avoir mit le script la connexion sera possible avec le nom mit en "host"

```
ssh ****(host)
```
### Encapsulation port
Sur le port LAN il est possible de mettre plusieurs sous interface pour les VLAN

Pour mettre une interface en sous interface :
```
conf t
interface GigabitEthernet0/0.10
encapsulation dot1q 10
ip address 192.168.10.1 255.255.255.0
end
```

### Explication

| Commande                               | Description                           |
| -------------------------------------- | ------------------------------------- |
| `conf t`                       | Permet de se mettre en mode configuration             |
| `encapsulation dot1q 10`         | Active l'encapsulation 802.1Q et associe la sous-interface au VLAN 10             |
---
