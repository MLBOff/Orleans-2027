# Documentation PKI — Autorité de Certification et certificat web

Cette documentation décrit l'architecture et les procédures mises en œuvre pour déployer l'**Autorité de Certification (CA)** interne du site d'Orléans sportludique.fr et délivrer le certificat TLS du serveur web Apache.

---

## 1. Architecture et organisation des fichiers

L'ensemble des éléments cryptographiques est centralisé dans l'arborescence `~/pki-groupe`. Cette structure isole strictement les composants de l'autorité de certification de ceux du serveur web applicatif.

```text
pki-groupe/
├── autorite/
│   ├── certificats/
│   │   └── ca-orleans.crt      # Certificat racine autosigné de la CA
│   ├── cles/
│   │   └── ca-orleans.key      # Clé privée de la CA (chiffrée)
│   └── ca.srl                  # Compteur des numéros de série émis
└── serveur/
    ├── certificats/
    │   └── web.crt             # Certificat final du serveur web
    ├── cles/
    │   └── web.key             # Clé privée du serveur web
    └── demandes/
        └── web.csr             # Demande de signature de certificat
```

### Politique de sécurité et droits d'accès

Les répertoires hébergeant les clés privées (`autorite/cles/` et `serveur/cles/`) sont restreints au seul propriétaire (`chmod 700`). 

* **Clé privée de la CA (`ca-orleans.key`) :** Protégée par une phrase de passe (chiffrement AES-256). Cette double sécurité empêche toute émission frauduleuse de certificats en cas de compromission du fichier.
* **Clé privée du serveur (`web.key`) :** Non chiffrée afin de permettre le redémarrage automatique du service Apache sans intervention manuelle.

---

## 2. Déploiement de l'Autorité de Certification (CA)

L'autorité racine du site d'Orléans repose sur une paire de clés asymétriques **RSA 4096 bits**, dimensionnée pour garantir une robustesse cryptographique sur le long terme.

### Génération de la clé privée racine
```bash
openssl genpkey -algorithm RSA -aes-256-cbc \
  -pkeyopt rsa_keygen_bits:4096 \
  -out autorite/cles/ca-orleans.key
```

### Émission du certificat racine autosigné
En tant qu'autorité racine, la CA signe son propre certificat pour une durée de validité de 365 jours :
```bash
openssl req -new -x509 -days 365 \
  -key autorite/cles/ca-orleans.key \
  -out autorite/certificats/ca-orleans.crt
```

### Caractéristiques du certificat racine
Le contrôle via `openssl x509 -in autorite/certificats/ca-orleans.crt -noout -text` valide les propriétés suivantes :
* **Subject et Issuer :** Identiques (certificat autosigné).
* **Extension `CA:TRUE` :** Habilite ce certificat à signer des certificats tiers.

> **Déploiement client :** Ce certificat racine (`ca-orleans.crt`) doit être importé dans le magasin de confiance des postes clients du domaine afin que les navigateurs reconnaissent la chaîne de confiance sans afficher d'alerte de sécurité.

---

## 3. Production du certificat du serveur web

Le serveur web d'Orléans est accessible via deux enregistrements DNS distincts : `www.orleans.sportludique.fr` (nom principal) et `orleans.sportludique.fr` (nom alternatif).

### Génération de la clé privée et de la CSR
Une clé **RSA 3072 bits** est générée pour le serveur, offrant un équilibre optimal entre sécurité et performances lors des négociations TLS :
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 \
  -out serveur/cles/web.key
```

La demande de signature de certificat (**CSR**) intègre obligatoirement l'extension **SAN** (*Subject Alternative Name*). Les navigateurs modernes s'appuient exclusivement sur ce champ — et non sur le seul *Common Name* (CN) — pour valider l'identité du domaine consulté :
```bash
openssl req -new -sha256 \
  -key serveur/cles/web.key \
  -out serveur/demandes/web.csr \
  -addext "subjectAltName=DNS:www.orleans.sportludique.fr,DNS:orleans.sportludique.fr"
```

| Attribut X.509 | Valeur configurée |
| :--- | :--- |
| **Country Name (C)** | `FR` |
| **Organization Name (O)** | `SportLudique` |
| **Common Name (CN)** | `www.orleans.sportludique.fr` |
| **Subject Alternative Name (SAN)** | `DNS:www.orleans.sportludique.fr`, `DNS:orleans.sportludique.fr` |

---

## 4. Signature et validation du certificat serveur

### Signature par la CA d'Orléans
La CA signe la demande `web.csr` et génère le certificat final `web.crt` :
```bash
openssl x509 -req -days 365 \
  -in serveur/demandes/web.csr \
  -CA autorite/certificats/ca-orleans.crt \
  -CAkey autorite/cles/ca-orleans.key \
  -CAserial autorite/ca.srl \
  -CAcreateserial \
  -out serveur/certificats/web.crt \
  -copy_extensions copy
```
* **Conservation des extensions (`-copy_extensions copy`) :** Transfère explicitement les noms DNS déclarés dans les SAN de la CSR vers le certificat signé.
* **Suivi des séries (`ca.srl`) :** Maintient l'unicité des numéros de série attribués par la CA.

### Contrôle d'intégrité et de chaîne de confiance
La conformité du certificat final est validée par deux vérifications :

1. **Inspection des métadonnées et des SAN :**
   ```bash
   openssl x509 -in serveur/certificats/web.crt \
     -noout -subject -issuer -dates -ext subjectAltName
   ```
2. **Validation cryptographique de la chaîne :**
   ```bash
   openssl verify -CAfile autorite/certificats/ca-orleans.crt serveur/certificats/web.crt
   ```
   *Résultat obtenu :* `serveur/certificats/web.crt: OK`
