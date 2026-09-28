## Règles de pare-feu

| Source | Destination | Service (Port) | Sens | Action |
|---|---|---|---|---|
| Internet | DMZ | HTTPS (443) | Entrant | Autoriser |
| Internet | VLAN Clients | Tous | Entrant | Refuser |
| Internet | VLAN Serveurs | Tous | Entrant | Refuser |
| Internet | VLAN Management | Tous | Entrant | Refuser |
| VLAN Clients | Internet | Tous | Sortant | Autoriser |
| VLAN Clients | VLAN Serveurs | Ports applicatifs nécessaires | Interne | Autoriser |
| VLAN Clients | VLAN Management | Tous | Interne | Refuser |
| VLAN Management | Pare-feu / équipements réseau | SSH (22), RDP (3389) | Interne | Autoriser |
| VLAN Management | VLAN Serveurs | SSH, RDP, HTTPS selon besoin | Interne | Autoriser |
| VLAN Serveurs | Internet | HTTP/HTTPS, DNS | Sortant | Autoriser au cas par cas |
| VLAN Serveurs | VLAN Clients | Tous | Interne | Refuser |
| DMZ | VLAN Clients | Tous | Interne | Refuser |
| DMZ | VLAN Serveurs | Ports applicatifs nécessaires | Interne | Autoriser au cas par cas |
| LAN | DMZ | Ports nécessaires | Interne | Autoriser au cas par cas |
| Tous | Tous | ICMP | Selon besoin | Limiter / autoriser selon besoin |
| Tous | Tous | Tous | Tous | Refuser |


## Matrice de Flux

| Adresse de destination | Port de destination | Identifiant / IP NAT |
|---|---:|---|
| IP Pub DMZ | 443 | IP Serveur WEB |
| IP Pub DMZ | Ports applicatifs nécessaires | IP Serveur concerné |
| IP VLAN Clients | Tous | Internet |
| IP VLAN Management | 22, 3389 | Pare-feu / équipements réseau |
| IP VLAN Management | 22, 3389, 443 | Serveurs concernés |
| IP VLAN Serveurs | 80, 443, 53 | Internet |
| IP DMZ | Ports applicatifs nécessaires | Serveurs internes concernés |
| IP LAN | Ports nécessaires | Services DMZ concernés |


