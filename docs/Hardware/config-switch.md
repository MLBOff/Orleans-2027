# Configuration d'un switch Cisco

## Configuration d'un port trunk

Pour configurer un port en **trunk** et autoriser les VLAN `140` et `142` :

```bash
interface FastEthernet1/0/1
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 140,142
 switchport mode trunk
```

### Explication

| Commande                                | Description                               |
| --------------------------------------- | ----------------------------------------- |
| `interface FastEthernet1/0/1`           | Sélectionne l'interface à configurer      |
| `switchport trunk encapsulation dot1q`  | Utilise l'encapsulation **802.1Q**        |
| `switchport trunk allowed vlan 140,142` | Autorise les VLAN 140 et 142 sur le trunk |
| `switchport mode trunk`                 | Configure le port en mode trunk           |


Un port **trunk** permet de transporter plusieurs VLAN sur une même liaison, par exemple entre deux switches ou entre un switch et un routeur.

---

## Configuration d'un port dans un VLAN

Pour placer un port dans un VLAN spécifique, il faut utiliser le mode **access**.

Par exemple, pour placer le port `FastEthernet1/0/1` dans le VLAN `140` :

```bash
interface FastEthernet1/0/1
 switchport mode access
 switchport access vlan 140
```

### Explication

| Commande                      | Description                          |
| ----------------------------- | ------------------------------------ |
| `interface FastEthernet1/0/1` | Sélectionne l'interface à configurer |
| `switchport mode access`      | Configure le port en mode access     |
| `switchport access vlan 140`  | Affecte le port au VLAN 140          |


Un port **access** appartient généralement à un seul VLAN. Il est utilisé pour connecter des équipements tels que des PC, imprimantes ou autres périphériques réseau.

---

## Configuration d'une interface VLAN

Une interface VLAN, également appelée **SVI (Switch Virtual Interface)**, permet d'attribuer une adresse IP au switch dans un VLAN.

Exemple avec le VLAN 149 :

```bash
interface Vlan149
 ip address 192.168.149.254 255.255.255.0
```

### Explication

| Commande                                   | Description                                          |
| ------------------------------------------ | ---------------------------------------------------- |
| `interface Vlan149`                        | Sélectionne l'interface virtuelle du VLAN 149        |
| `ip address 192.168.149.254 255.255.255.0` | Attribue l'adresse IP `192.168.149.254/24` au switch |

Le réseau correspondant est :

```text
Réseau      : 192.168.149.0/24
Adresse IP  : 192.168.149.254
Masque      : 255.255.255.0
```

Cette adresse peut notamment être utilisée comme **passerelle par défaut** pour les équipements présents dans le VLAN 149.

Par exemple :

```
PC
IP          : 192.168.149.10
Masque      : 255.255.255.0
Passerelle  : 192.168.149.254
```


L'interface `Vlan149` doit être active pour pouvoir être utilisée. Le VLAN 149 doit également exister sur le switch.

---

## Configuration d'une route statique

La commande `ip route` permet de configurer une **route statique** vers un autre réseau.

Exemple :

```bash
ip route 192.168.140.0 255.255.255.0 192.168.149.1
```

Cette configuration signifie :

> Pour atteindre le réseau `192.168.140.0/24`, le switch doit envoyer les paquets vers `192.168.149.1`.

### Explication

| Élément         | Description                                    |
| --------------- | ---------------------------------------------- |
| `ip route`      | Crée une route statique                        |
| `192.168.140.0` | Réseau de destination                          |
| `255.255.255.0` | Masque du réseau de destination                |
| `192.168.149.1` | Adresse IP du prochain équipement (*next-hop*) |

---

## Route par défaut

Une route par défaut permet d'indiquer au switch où envoyer les paquets lorsqu'il ne possède aucune route spécifique vers leur destination.

```bash
ip route 0.0.0.0 0.0.0.0 192.168.149.1
```

Cette commande signifie :

> Si aucune route connue ne correspond à la destination, envoyer le trafic vers `192.168.149.1`.

### Explication

| Élément         | Description                |
| --------------- | -------------------------- |
| `0.0.0.0`       | Toutes les destinations    |
| `0.0.0.0`       | Tous les masques           |
| `192.168.149.1` | Passerelle / prochain saut |


La route `0.0.0.0/0` est appelée **route par défaut**.

---

## Création d'un utilisateur pour SSH

Pour créer un utilisateur local permettant de se connecter au switch en SSH :

```bash
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
hostname SW-01

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
| `hostname SW-01`                       | Définit le nom du switch              |
| `ip domain-name exemple.local`         | Définit le nom de domaine             |
| `username NOM privilege 15 secret MDP` | Crée l'utilisateur local              |
| `crypto key generate rsa modulus 2048` | Génère les clés RSA nécessaires à SSH |
| `ip ssh version 2`                     | Active SSH version 2                  |
| `line vty 0 15`                        | Sélectionne les lignes VTY            |
| `login local`                          | Utilise les utilisateurs locaux       |
| `transport input ssh`                  | Autorise les connexions SSH           |

---

## Vérification de la configuration

### Vérifier les VLAN

```bash
show vlan brief
```

### Vérifier les trunks

```bash
show interfaces trunk
```

### Vérifier une interface

```bash
show running-config interface FastEthernet1/0/1
```

### Vérifier les interfaces VLAN

```bash
show ip interface brief
```

Cette commande permet notamment de vérifier l'état de `Vlan149` et son adresse IP.

### Vérifier la table de routage

```bash
show ip route
```

Cette commande permet de voir les réseaux directement connectés, les routes statiques et la route par défaut.

### Vérifier SSH

```bash
show ip ssh
```

### Vérifier les utilisateurs

```bash
show running-config | include username
```

---

## Sauvegarde de la configuration

Après avoir terminé la configuration, sauvegarder la configuration courante :

```bash
copy running-config startup-config
```

ou :

```bash
write memory
```


La configuration présente dans `running-config` est active immédiatement, mais elle sera perdue après un redémarrage si elle n'est pas sauvegardée dans `startup-config`.

---

## Récapitulatif

| Élément                                  | Fonction                                                                |
| ---------------------------------------- | ----------------------------------------------------------------------- |
| **Trunk**                                | Transporte plusieurs VLAN                                               |
| **Access**                               | Connecte un équipement à un VLAN                                        |
| **Interface VLAN / SVI**                 | Donne une adresse IP au switch dans un VLAN                             |
| **`ip route`**                           | Ajoute une route vers un réseau                                         |
| **Route par défaut**                     | Définit la passerelle utilisée lorsque aucune route spécifique n'existe |
| **Utilisateur SSH**                      | Permet l'authentification sur le switch                                 |
| **SSH**                                  | Permet l'administration distante sécurisée                              |
| **`show ip route`**                      | Affiche la table de routage                                             |
| **`copy running-config startup-config`** | Sauvegarde la configuration                                             |
