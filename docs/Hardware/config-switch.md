# Configuration d'un switch  


Conf port mode trunk
```
interface FastEthernet1/0/1
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 140,142
 switchport mode trunk
```

Conf port vlan 
```
interface FastEthernet1/0/1
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 140,142
 switchport mode trunk
```

Création user pour ssh 
```
username matheo privilege 15 secret 5 $1$RAVI$YbDlx75U2cvGpgNM5xWnm.
username nolan privilege 15 secret 5 $1$Wtx5$Kk/V32oKDW78iQss9AdiT0
username Lorenzo privilege 15 secret 5 $1$J6OA$7ZHjwW4uzqftOz0qA/C4r0
```