# Documentation PKI — Gestion des certificats TLS

Cette documentation rassemble les procédures de création et de gestion des certificats TLS pour l'infrastructure **SportLudique**, couvrant deux usages :
1. **Certificat autosigné :** Pour le maquettage rapide et les tests isolés.
2. **Autorité de Certification (CA) interne :** Pour le déploiement sécurisé et la chaîne de confiance en production (site d'Orléans `sportludique.fr`).

---

## Prérequis communs

* Disposer d'un accès terminal sur un serveur Linux.
* Avoir l'outil **OpenSSL** installé (`openssl version`).
* Connaître le **nom de domaine complet (FQDN)** ou l'adresse IP du serveur cible.

---

## Partie 1 : Création d'un certificat autosigné (Tests & Maquettage)

Un **certificat autosigné** est signé par sa propre clé privée, sans passer par une Autorité de Certification. Il permet de chiffrer rapidement les flux en environnement de développement, mais déclenche une alerte de sécurité dans les navigateurs car l'émetteur n'est pas reconnu.

### 1. Génération en une seule commande
Cette commande génère simultanément la **clé privée** et le **certificat autosigné** (valable 365 jours) avec l'extension **SAN** (*Subject Alternative Name*), indispensable pour les navigateurs modernes :

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout serveur.key \
  -out serveur.crt \
  -subj "/C=FR/ST=Centre-Val de Loire/L=Chartres/O=SportLudique/CN=www.sportludique.fr" \
  -addext "subjectAltName=DNS:www.sportludique.fr,DNS:sportludique.fr"
```

* **`-x509`** : Génère directement un certificat autosigné plutôt qu'une demande de signature (CSR).
* **`-newkey rsa:4096`** : Crée une nouvelle clé privée RSA de 4096 bits.
* **`-sha256` / `-days 365`** : Utilise le hachage SHA-256 pour une validité de 1 an.
* **`-nodes`** : (*No DES*) Ne chiffre pas la clé privée par mot de passe (permet le redémarrage automatique d'Apache/Nginx).
* **`-subj` / `-addext`** : Définit l'identité (`CN`) et les noms DNS autorisés dans l'extension **SAN**.

### 2. Sécurisation des permissions
La clé privée ne doit jamais être lisible par d'autres utilisateurs que `root` :

```bash
chmod 600 serveur.key
chmod 644 serveur.crt
```

> **Limite :** À réserver exclusivement aux phases de maquettage. En production, utilisez la **CA interne** détaillée ci-dessous.

---

## Partie 2 : Déploiement d'une PKI interne (Production)

Cette architecture déploie l'**Autorité de Certification (CA)** interne du site d'Orléans (`sportludique.fr`) afin de délivrer des certificats reconnus sans alerte de sécurité.

### 1. Architecture et politique de sécurité
L'ensemble des éléments cryptographiques est centralisé dans l'arborescence `~/pki-groupe` afin d'isoler la CA du serveur web applicatif :

```text
pki-groupe/
├── autorite/
│   ├── certificats/
│   │   └── ca-orleans.crt      # Certificat racine autosigné de la CA
│   ├── cles/
│   │   └── ca-orleans.key      # Clé privée de la CA (chiffrée AES-256)
│   └── ca.srl                  # Compteur des numéros de série émis
└── serveur/
    ├── certificats/
    │   └── web.crt             # Certificat final du serveur web
    ├── cles/
    │   └── web.key             # Clé privée du serveur web (non chiffrée)
    └── demandes/
        └── web.csr             # Demande de signature de certificat
```

* **Droits d'accès :** Les répertoires `autorite/cles/` et `serveur/cles/` sont restreints au seul propriétaire (`chmod 700`).
* **Protection des clés :** `ca-orleans.key` est protégée par une phrase de passe (AES-256) contre toute émission frauduleuse, tandis que `web.key` reste non chiffrée pour autoriser le redémarrage automatique d'Apache.

---

### 2. Création de l'Autorité de Certification (CA)

1. **Générer la clé privée racine (RSA 4096 bits chiffrée) :**
   ```bash
   openssl genpkey -algorithm RSA -aes-256-cbc \
     -pkeyopt rsa_keygen_bits:4096 \
     -out autorite/cles/ca-orleans.key
   ```

2. **Émettre le certificat racine autosigné (valable 365 jours) :**
   ```bash
   openssl req -new -x509 -days 365 \
     -key autorite/cles/ca-orleans.key \
     -out autorite/certificats/ca-orleans.crt
   ```

3. **Déploiement de la confiance :** Importez `ca-orleans.crt` dans le magasin de confiance des postes clients (ou du reverse proxy) afin que la chaîne de confiance soit validée automatiquement.

---

### 3. Production de la demande de certificat serveur (CSR)

Le serveur web d'Orléans répond aux domaines `www.orleans.sportludique.fr` et `orleans.sportludique.fr`.

1. **Générer la clé privée du serveur (RSA 3072 bits) :**
   ```bash
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 \
     -out serveur/cles/web.key
   ```

2. **Générer la CSR avec l'extension SAN :**
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

### 4. Signature du certificat par la CA

La CA signe la demande `web.csr` et transfère explicitement les extensions SAN grâce à `-copy_extensions copy` :

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

---

## Partie 3 : Commandes de vérification et d'audit

Quelle que soit la méthode utilisée (autosigné ou CA interne), contrôlez systématiquement le certificat produit avant sa mise en service :

1. **Afficher les détails complets d'un certificat (dates, émetteur, sujet, SAN) :**
   ```bash
   openssl x509 -in serveur/certificats/web.crt \
     -noout -subject -issuer -dates -ext subjectAltName
   ```

2. **Valider cryptographiquement la chaîne de confiance (PKI interne) :**
   ```bash
   openssl verify -CAfile autorite/certificats/ca-orleans.crt serveur/certificats/web.crt
   ```
   *Résultat attendu :* `serveur/certificats/web.crt: OK`
