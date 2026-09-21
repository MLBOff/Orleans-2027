# Configuration d'un DHCP 

## I - Mise à jour de la machine 
```
sudo apt update
sudo apt upgrade -y
```

## II - Mettre son IP en mode static
Dans un premier temps accéder au fichier de conf pour modifier les IP.

```
sudo nano /etc/network/interfaces
```

Puis le modifier en remplacant `dhcp` par `static` puis y ajouter l'adresse IP choisi.

## III Activation et mise en place du DHCP
Ouvrir le fichier `/etc/default/isc-dhcp-server`.

Recherchez la ligne `INTERFACESv4=` et inscrivez le nom de votre interface réseau entre guillemets.

Exemple du rendu :
```
INTERFACESv4="ens33"
```

Ensuite ouvrir le fichier `/etc/dhcp/dhcpd.conf`

Faites défiler le fichier pour ajouter ou décommenter un bloc de sous-réseau (subnet) adapté à votre réseau

Exemple du rendu :
```
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;              # Plage d'IP distribuées
  option routers 192.168.1.1;                     # Passerelle par défaut
  option domain-name-servers 8.8.8.8, 1.1.1.1;    # Serveurs DNS
  option domain-name "monreseau.local";           # Nom de domaine (optionnel)
  default-lease-time 600;                         #durée du bail par défaut
  max-lease-time 7200;                            #durée maximale du bail DHCP
}
```

Ensuite entré les commandes suivantes pour démarre le service DHCP.
```
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```

## IV Vérification du bon fonctionnement du service DHCP
Pour vérifier si le service dhcp fonctionne et qu'il n'y a aucune erreur de conf il faut entrer cette commande : 
```
sudo systemctl status isc-dhcp-server
```

En cas d'erreur le service ne sera pas démarre et un ou plusieurs code(s) d'erreur(s) apparaitront.

Si l'erreur persiste aller voir les logs du dhcp `journalctl -u isc-dhcp-server -f`.
