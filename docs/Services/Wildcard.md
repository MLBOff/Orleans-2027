# Documentation PKI — Certificat Wildcard

Cette procédure décrit les étapes pour générer une demande de certificat Wildcard depuis le reverse proxy, la faire signer par l'Autorité de Certification (CA) interne, préparer les fichiers `.crt` / `.pem` et sécuriser les services situés derrière le reverse proxy.

---

## 1. Générer la clé privée et la demande de signature (CSR) sur le Reverse Proxy

Sur le reverse proxy, générez la clé privée (non chiffrée pour permettre le redémarrage automatique d'Apache/Nginx) ainsi que la CSR incluant l'extension **SAN** (*Subject Alternative Name*) pour le domaine racine et le wildcard.

```bash
# 1. Génération de la clé privée (RSA 3072 bits)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 \
  -out wildcard.orleans.key

# 2. Génération de la CSR avec le Wildcard dans le SAN
openssl req -new -sha256 \
  -key wildcard.orleans.key \
  -out wildcard.orleans.csr \
  -subj "/C=FR/ST=Centre-Val de Loire/L=Orleans/O=SportLudique/CN=*.orleans.sportludique.fr" \
  -addext "subjectAltName=DNS:*.orleans.sportludique.fr,DNS:orleans.sportludique.fr"
```

*Transférez ensuite uniquement le fichier `wildcard.orleans.csr` vers le serveur hébergeant votre Autorité de Certification (CA).*

---

## 2. Faire signer le certificat par la CA et récupérer le `.crt`

Sur **le serveur de l'Autorité de Certification** (`pki-groupe`), signez la demande en conservant l'extension SAN (`-copy_extensions copy`) :

```bash
openssl x509 -req -days 365 \
  -in wildcard.orleans.csr \
  -CA autorite/certificats/ca-orleans.crt \
  -CAkey autorite/cles/ca-orleans.key \
  -CAserial autorite/ca.srl \
  -CAcreateserial \
  -out wildcard.orleans.crt \
  -copy_extensions copy
```

*Rapatriez le certificat signé **`wildcard.orleans.crt`** sur le reverse proxy.*

---

## 3. Création des fichiers `.pem` (Chaîne complète et format combiné)

Selon le logiciel utilisé (Apache2, Nginx, HAProxy), il est nécessaire de regrouper le certificat signé, la CA racine et/ou la clé privée au format **`.pem`** :

* **Créer le bundle de chaîne complète (`fullchain.pem` — certificat + CA racine) :**
  ```bash
  cat wildcard.orleans.crt ca-orleans.crt > wildcard.orleans.fullchain.pem
  ```

* **Créer un fichier `.pem` combiné (certificat + clé privée, requis par certains services comme HAProxy) :**
  ```bash
  cat wildcard.orleans.crt wildcard.orleans.key > wildcard.orleans.bundle.pem
  chmod 600 wildcard.orleans.bundle.pem wildcard.orleans.key
  ```

---

## 4. Déploiement vers les services derrière le Reverse Proxy

> **⚠️ Point de sécurité important :**
> * **Si les services internes utilisent ce même Wildcard** (pour chiffrer le lien `Reverse Proxy ➔ Service`), vous devez leur envoyer le **certificat (`.crt` / `.fullchain.pem`) ET sa clé privée (`.key`)** via un canal chiffré (`scp`), puis activer le HTTPS sur chaque service.
> * **Si chaque service possède son propre certificat dédié** (architecture recommandée), **n'envoyez jamais la clé privée du Wildcard** aux services : envoyez uniquement le certificat public de la CA (`ca-orleans.crt`) au reverse proxy.

### Exemple d'envoi sécurisé (`scp`) vers un service interne
```bash
scp wildcard.orleans.crt wildcard.orleans.key wildcard.orleans.fullchain.pem user@ip-service-interne:/etc/ssl/sportludique/
```

### Configuration Apache2 côté Reverse Proxy (Re-chiffrement TLS)
Dans le VirtualHost du reverse proxy, activez le proxy SSL vers le service interne en vérifiant la signature de votre CA :

```apache
<VirtualHost *:443>
    ServerName app1.orleans.sportludique.fr

    # Certificat Wildcard présenté aux clients externes
    SSLEngine on
    SSLCertificateFile /etc/ssl/sportludique/wildcard.orleans.crt
    SSLCertificateKeyFile /etc/ssl/sportludique/wildcard.orleans.key
    SSLCertificateChainFile /etc/ssl/sportludique/ca-orleans.crt

    # Chiffrement TLS vers le service interne
    SSLProxyEngine on
    SSLProxyCACertificateFile /etc/ssl/sportludique/ca-orleans.crt
    SSLProxyVerify require
    SSLProxyCheckPeerName on

    ProxyPass / https://sportludique.fr
    ProxyPassReverse / https://sportludique.fr
</VirtualHost>
```
