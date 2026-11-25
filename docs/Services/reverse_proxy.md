# Documentation Technique — Reverse Proxy Nginx (`orleans.sportludique.fr`)

## 1. Architecture et Flux Réseau

Le reverse proxy **Nginx** (`10.140.0.61`) centralise les requêtes entrantes HTTP/HTTPS pour le domaine `*.orleans.sportludique.fr`, assure la terminaison SSL via un certificat wildcard, et redirige le trafic vers les serveurs internes.

| Service / Site | Nom de domaine (FQDN) | Port Externe | Serveur Backend | Port Backend | Fichier de conf Nginx |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Site principal** | `www.orleans.sportludique.fr` | 80 → 443 | Apache2 (`192.168.149.5`) | `443` (HTTPS) | `site1.fr.conf` |
| **iTop (ITSM)** | `itop.orleans.sportludique.fr` | 80 → 443 | Apache2 (`192.168.149.5`) | `4443` (HTTPS) | `site2.fr.conf` |
| *(À venir)* **App Docker** | `app.orleans.sportludique.fr` | 80 → 443 | Hôte Docker | `<PORT_CONTENEUR>` | `site3-docker.conf` |
| *(À venir)* **Site IIS** | `iis.orleans.sportludique.fr` | 80 → 443 | Windows Server (IIS) | `80` ou `443` | `site4-iis.conf` |

---

## 2. Gestion des Certificats SSL/TLS (Wildcard)

Les certificats sont stockés dans `/etc/nginx/ssl/` :
* **Chaîne complète :** `/etc/nginx/ssl/wildcard-fullchain.pem`
* **Clé privée :** `/etc/nginx/ssl/wildcard.key` (droits restreints : `chmod 600`)

### Points de vigilance sur le certificat Wildcard
1. **Extension SAN (Subject Alternative Name) :** Les navigateurs modernes ignorent le champ *Common Name (CN)*. Le certificat doit impérativement contenir `DNS:*.orleans.sportludique.fr` (et `DNS:orleans.sportludique.fr` si le domaine racine est utilisé) dans les **SAN**.
2. **Ordre de la chaîne (`fullchain`) :** Le fichier `wildcard-fullchain.pem` doit contenir dans l'ordre :
   1. Le certificat du serveur (`*.orleans.sportludique.fr`)
   2. Le certificat de l'autorité intermédiaire (et racine interne si PKI privée)
3. **Cohérence clé/certificat :** Vérifier que l'empreinte de la clé correspond au certificat :
   ```bash
   openssl x509 -noout -modulus -in /etc/nginx/ssl/wildcard-fullchain.pem | openssl md5
   openssl rsa -noout -modulus -in /etc/nginx/ssl/wildcard.key | openssl md5
   ```

---

## 3. Configurations Nginx Actuelles

### Site 1 — `www.orleans.sportludique.fr` (`/etc/nginx/sites-available/site1.fr.conf`)

```nginx
server {
    listen 443 ssl;
    server_name www.orleans.sportludique.fr;

    ssl_certificate     /etc/nginx/ssl/wildcard-fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/wildcard.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass https://192.168.149.5:443;

        proxy_ssl_server_name on;
        proxy_ssl_name www.orleans.sportludique.fr;
        proxy_ssl_verify off;
        proxy_ssl_trusted_certificate /etc/nginx/ssl/wildcard-fullchain.pem;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name www.orleans.sportludique.fr;
    return 301 https://$host$request_uri;
}
```

### Site 2 — `itop.orleans.sportludique.fr` (`/etc/nginx/sites-available/site2.fr.conf`)

```nginx
server {
    listen 443 ssl;
    server_name itop.orleans.sportludique.fr;

    ssl_certificate     /etc/nginx/ssl/wildcard-fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/wildcard.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass https://192.168.149.5:4443;

        proxy_ssl_server_name on;
        proxy_ssl_name itop.orleans.sportludique.fr;
        proxy_ssl_verify off;
        proxy_ssl_trusted_certificate /etc/nginx/ssl/wildcard-fullchain.pem;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name itop.orleans.sportludique.fr;
    return 301 https://$host$request_uri;
}
```

---

## 4. Incidents Rencontrés et Résolutions

### Incident 1 : Erreurs TLS liées au certificat Wildcard
* **Symptôme :** Alerte de sécurité navigateur ou rejet de connexion TLS lors de l'accès aux sous-domaines.
* **Cause :** Certificat wildcard généré sans le SAN adéquat ou chaîne de certification incomplète.
* **Résolution :** Regénération du certificat avec `*.orleans.sportludique.fr` dans les SAN et assemblage du fichier `wildcard-fullchain.pem`.

### Incident 2 : Un seul site accessible sur les deux (conflit VirtualHost Apache2)
* **Symptôme :** Seul `www.orleans.sportludique.fr` répondait correctement ; `itop.orleans.sportludique.fr` renvoyait vers le premier site ou échouait.
* **Cause :** Les deux sites étaient hébergés sur la même machine (`192.168.149.5`). Sans transmission du SNI (*Server Name Indication*) entre Nginx et Apache2, ou avec deux VirtualHosts HTTPS concurrents mal isolés, Apache2 servait uniquement le VirtualHost par défaut.
* **Résolution étape par étape :**
  1. **Isolation en HTTP (port 80) :** Bascule temporaire du VirtualHost `itop` sur le port `80` côté Apache2 et `proxy_pass http://192.168.149.5:80;` côté Nginx pour valider le routage par en-tête `Host`.
  2. **Passage en HTTPS sur port distinct (`4443`) :** Activation du SSL sur Apache2 sur le port `4443` pour iTop, puis mise à jour du `proxy_pass` vers `https://192.168.149.5:4443`.
  3. **Activation du SNI côté Nginx :** Ajout de `proxy_ssl_server_name on;` et `proxy_ssl_name <fqdn>;` dans chaque bloc `location /` afin que Nginx transmette le nom d'hôte exact lors du *handshake* TLS avec Apache2.

---

## 5. Procédure d'Ajout des Futurs Sites (Docker & IIS)

Pour chaque nouveau sous-domaine, créer un fichier dans `/etc/nginx/sites-available/`, l'activer via lien symbolique (`ln -s`), puis recharger Nginx.

### Modèle A : Conteneur Docker (HTTP interne)
Un conteneur Docker expose généralement son service en HTTP sur un port mappé (ex. `8080`). Le chiffrement HTTPS est géré uniquement par Nginx (terminaison SSL).

```nginx
server {
    listen 443 ssl;
    server_name docker.orleans.sportludique.fr;

    ssl_certificate     /etc/nginx/ssl/wildcard-fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/wildcard.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://<IP_SERVEUR_DOCKER>:<PORT_CONTENEUR>;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name docker.orleans.sportludique.fr;
    return 301 https://$host$request_uri;
}
```

### Modèle B : Serveur Windows IIS
Si IIS écoute en **HTTPS (port 443)** avec liaison SNI (*Require Server Name Indication* coché dans IIS), conserver les directives `proxy_ssl_server_name` :

```nginx
server {
    listen 443 ssl;
    server_name iis.orleans.sportludique.fr;

    ssl_certificate     /etc/nginx/ssl/wildcard-fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/wildcard.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass https://<IP_SERVEUR_IIS>:443;

        proxy_ssl_server_name on;
        proxy_ssl_name iis.orleans.sportludique.fr;
        proxy_ssl_verify off;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name iis.orleans.sportludique.fr;
    return 301 https://$host$request_uri;
}
```
*(Note : si le site IIS est configuré uniquement en HTTP sur le port 80, remplacer `proxy_pass` par `http://<IP_SERVEUR_IIS>:80;` et retirer les directives `proxy_ssl_*`.)*

---

## 6. Commandes d'Exploitation et Vérification

1. **Activer un nouveau fichier de configuration :**
   ```bash
   sudo ln -s /etc/nginx/sites-available/nouveau-site.conf /etc/nginx/sites-enabled/
   ```
2. **Tester la syntaxe avant application :**
   ```bash
   sudo nginx -t
   ```
   *Validation :* La commande doit retourner `syntax is ok` et `test is successful`.
3. **Recharger Nginx sans coupure de service :**
   ```bash
   sudo systemctl reload nginx
   ```
4. **Vérifier le certificat présenté et les en-têtes HTTP :**
   ```bash
   curl -I https://itop.orleans.sportludique.fr
   ```
   *Validation :* Retour `HTTP/1.1 200 OK` (ou `302` vers la page de login) sans erreur de certificat.