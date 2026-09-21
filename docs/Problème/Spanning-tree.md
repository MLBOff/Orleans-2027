### Contexte
Il est possible que s'il y a eu une boucle sur un port, spanning-tree va bloquer se port et restera down en physique mais UP en faisant un ```show interface``` 


## Diagnostic
D'abord on verifie si le port est bloqué:
```
show spanning-tree interface [port]
```

Cela va afficher le VLAN affecté :

```
Vlan                Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
VLAN10              Desg BKN*19        128.7    P2p *TYPE_Inc
```

| Champ        | Valeur     | Signification                                          |
| ------------ | ---------- | ------------------------------------------------------ |
| **VLAN**     | `VLAN10` | Le VLAN concerné est le **VLAN 10**                   |
| **Role**     | `Desg`     | **Designated Port**                                    |
| **Sts**      | `BKN`      | **Broken** : STP considère le port comme problématique |
| **Cost**     | `19`       | Coût STP du lien                                       |
| **Prio.Nbr** | `128.7`    | Priorité du port = 128, numéro de port = 7             |
| **Type**     | `P2p`      | Le lien est considéré comme **point-à-point**          |
| `*TYPE_Inc`  | `TYPE_Inc` | Type de port STP incohérent                            |



## Résolution du problème
Avant de désactiver STP sur tout les VLAN, on va d'abord essayer de le desactiver sur le port bloqué

```
interface FastEthernet1/0/1
spanning-tree portfast
```
Si même avec la ligne ```portfast``` c'est toujours bloqué. On va désactiver STP sur le VLAN concerné

```
no spanning-tree vlan 10
```
ou alors si vous voulez le faire pour tout les VLAN

```
no spanning-tree vlan 1-4094
```


