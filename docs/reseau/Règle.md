## Règles de pare-feu LAN


| Protocole | Source | Port Source | Destination | Port Destination | Action |
| :---: | :---: | :---: | :---: | :---: | :---: |
| any | Internet | any | LAN2FW | any | Bloquer |
| TCP | Serv web DMZ | 443 | BDD DMZ Privé | 3306 | Autoriser |
| any | Internet | any | DMZ Privé | any | Bloquer |
| any | interne | any | internet | any | Autoriser |
| any | Interne | any | DMZ | any | Bloquer |
| TCP | BDD DMZ Privé | 3306 | Serv web DMZ | 3306 | autoriser |
| any | Internet | any | DMZ | any | Bloquer |
| TCP | Interne | 143 | Serv mail | 143 | Autoriser |
| TCP | Interne | 587 | Serv mail | 583 | Autoriser |


## Règles de pare-feu WAN

| Protocole | Source | Port Source | Destination | Port Destination | Action |
| ----- | ----- | ----- | ----- | ----- | ----- |
| Any | Internet | any | LAN2FW | Any | Bloquer |
| TCP | Internet | 443 | Serv web | 443 | Autoriser |
| TCP | Internet | 80 | Serv web | 80 | Autoriser |
| TCP | Internet | 25 | Serv Mail | 25 | Autoriser |
| TCP | Internet | 587 | Serv Mail | 587 | Autoriser |
| TCP | Internet | 143 | Serv Mail | 143 | Autoriser |



