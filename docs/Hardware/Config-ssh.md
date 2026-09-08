# Guide de Configuration et Réinitialisation d'un Switch Cisco

Ce guide récapitule les procédures essentielles pour la gestion, la réinitialisation et la sécurisation d'un switch Cisco IOS.

---

## 🔄 1. Réinitialisation complète du Switch

Pour remettre un switch à sa configuration d'usine lorsqu'il est bloqué ou configuré précédemment.

    La suppression de `config.text` et `vlan.dat` efface définitivement la configuration existante ainsi que la base de données des VLANs.

### Procédure étape par étape

1. Maintenez enfoncé le bouton **MODE** en façade tout en allumant/branchant le switch jusqu'à ce que la LED `SYST` clignote ou reste fixe en vert/ambre.
2. Une fois dans le prompt d'amorçage `switch:`, saisissez les commandes suivantes :

```
switch: flash_init
switch: del flash:config.text
switch: del flash:vlan.dat
switch: reset
```

=== "Description des commandes"
`flash_init`= Initialise le système de fichiers mémoire FLASH |
`del flash:config.text` = Supprime le fichier de configuration de démarrage |
`del flash:vlan.dat` = Supprime la base de données des VLANs enregistrés |
`reset` =  Redémarre le switch avec les paramètres d'usine |
    
    
### Configuration pas à pas

```cisco title="Configuration SSH & Compte Utilisateur"
!-- 1. Nom de l'équipement et Domaine
conf t
hostname SW-ACCESS-01
ip domain-name domaine.local

!-- 2. Génération de la clé de chiffrement RSA
crypto key generate rsa
# Indiquer la taille de clé souhaitée lors du prompt (ex: 1024 ou 2048)

!-- 3. Création de l'utilisateur Administrateur
username admin privilege 15 secret MonMotDePasseSecurise!

!-- 4. Restriction des lignes d'accès VTY au protocole SSH uniquement
line vty 0 15
 transport input ssh
 login local
 exit
```

---

## 🌐 3. Configuration Réseau (Adresse IP de Management)

Attribution d'une adresse IP d'administration sur une interface VLAN.

```cisco title="Interface VLAN de Management"
conf t
interface vlan 140
 description VLAN Management
 ip address 10.140.0.1 255.255.255.128
 no shutdown
 exit
```

!!! info "Rappel de sous-réseau"
    Le masque `/25` (`255.255.255.128`) permet d'avoir 126 adresses hôtes utilisables dans le sous-réseau (de `10.140.0.1` à `10.140.0.126`).

---

## 🔌 4. Affectation d'un VLAN sur un Port d'Accès

Procédure pour associer un port du switch à un VLAN spécifique en mode `access`.

```cisco title="Configuration Interface d'Accès"
conf t
interface FastEthernet 0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
```

=== "Vérification"
    Pour vérifier l'état des VLANs et de l'interface :
    
    ```cisco
    show vlan brief
    show interface FastEthernet 0/1 switchport
    ```
