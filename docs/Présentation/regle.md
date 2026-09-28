## Règles de pare-feu

| Source | Destination | Service (Port) | Action |
|---|---|---|---|
| Internet | Serveur WEB | HTTPS (443) | Autoriser |
| Internet | Serveur Mail | SMTP (25) | Autoriser |
| Internet | Serveur DNS | DNS (53 UDP/TCP) | Autoriser |
| Mana | Pare-feu | Tous | Autoriser |

## Table de NAT

| Adresse de destination | Port de destination | Identifiant / IP NAT |
|---|---:|---|
| IP Pub | 443 | IP Srv |
| IP Pub | 25 | IP Srv |
| IP Pub | 53 | IP Srv |
