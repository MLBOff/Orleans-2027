# Documentation PKI — Autorité de Certification Intermédiaire (Site d'Orléans)

Cette documentation détaille la migration de l'Autorité de Certification (CA) d'**Orléans** vers une **CA intermédiaire** intégrée à la hiérarchie de confiance globale de **SportLudique**, ainsi que la mise en place d'un registre d'émission (`openssl ca`) et la génération de la chaîne complète (`fullchain.pem`) pour le serveur Apache.

---

## 1. Architecture de la chaîne de confiance

Contrairement à une CA autonome (autosignée), l'autorité du site d'Orléans est désormais certifiée par la **CA racine SportLudique** (détenue par le prestataire). La clé privée existante (`ca-orleans.key`) est conservée, mais un nouveau certificat intermédiaire (`ca-intermediaire.crt`) remplace l'ancien certificat racine local pour toute nouvelle signature.

```text
[ CA Racine SportLudique ]  (sportludique-root.crt.pem — de confiance sur les clients)
            │
            ▼  Signe

[ CA Intermédiaire Orléans ] (ca-intermediaire.crt — clé privée ca-orleans.key)
            │
            ▼  Signe

[ Certificat Serveur Web ]   (web-intermediaire.crt — www.orleans.sportludique.fr)
```

---

## 2. Demande de signature de la CA intermédiaire (CSR)

Une demande de certificat (CSR) est générée à partir de la clé privée existante de l'autorité d'Orléans pour être soumise à l'autorité racine :

```bash
mkdir -p autorite/demandes

openssl req -new -sha256 \
  -key autorite/cles/ca-orleans.key \
  -out autorite/demandes/ca-groupe.csr \
  -subj "/C=FR/O=SportLudique/CN=CA Orléans SportLudique"
```

### Contrôle de la demande avant transmission
```bash
openssl req -in autorite/demandes/ca-groupe.csr -noout -verify -text
```
* **Transmission :** Seul le fichier public `ca-groupe.csr` est transmis au prestataire pour le périmètre DNS `orleans.sportludique.fr`. La clé privée `ca-orleans.key` ne quitte jamais le serveur.

---

## 3. Intégration des certificats et validation de la confiance

Une fois signés et récupérés, les certificats sont déposés dans `autorite/certificats/` :
* `sportludique-root.crt.pem` : Certificat public de la CA racine SportLudique.
* `ca-intermediaire.crt` : Certificat de la CA intermédiaire d'Orléans signé par la racine.

### 1. Contrôle de l'empreinte de la CA racine
Avant tout déploiement, l'intégrité du certificat racine est validée en comparant son empreinte SHA-256 à celle communiquée par le prestataire :

```bash
openssl x509 -in autorite/certificats/sportludique-root.crt.pem -noout -fingerprint -sha256
```
* **Empreinte de référence validée :** `3C:8F:51:57:6E:C0:EF:51:94:9C:D5:12:E0:C8:BD:8A:CC:BF:DD:9B:FD:C0:50:00:9D:CE:FC:D6:AE:08:5F:0B`

### 2. Validation de la signature intermédiaire
```bash
openssl verify \
  -CAfile autorite/certificats/sportludique-root.crt.pem \
  autorite/certificats/ca-intermediaire.crt
```
* **Résultat :** `autorite/certificats/ca-intermediaire.crt: OK`
* **Contraintes techniques présentes :** `CA:TRUE`, `pathlen:0` (interdit de créer une sous-CA de niveau 3), usages `keyCertSign` et `cRLSign`, et restriction au domaine `orleans.sportludique.fr`.

### 3. Vérification de la correspondance des clés
Les empreintes SHA-256 des clés publiques extraites respectivement de la clé privée et du certificat intermédiaire sont strictement identiques, confirmant que la paire de clés n'a pas été altérée :

```bash
openssl pkey -in autorite/cles/ca-orleans.key -pubout -outform DER | openssl dgst -sha256
openssl x509 -in autorite/certificats/ca-intermediaire.crt -pubkey -noout | \
  openssl pkey -pubin -outform DER | openssl dgst -sha256
```

---

## 4. Configuration du registre d'émission (`openssl ca`)

Pour assurer la traçabilité, le suivi des numéros de série et la gestion future des révocations (CRL), l'autorité utilise désormais un registre complet géré par `openssl ca`.

### Initialisation de la base de données de la CA
```bash
mkdir -p autorite/newcerts autorite/crl
touch autorite/index.txt
openssl rand -hex 16 > autorite/serial
printf '1000\n' > autorite/crlnumber
```

| Fichier / Dossier | Rôle dans le registre |
| :--- | :--- |
| `autorite/index.txt` | Base de données listant tous les certificats émis et leur statut (Valide / Révoqué) |
| `autorite/serial` | Compteur hexadécimal attribuant un numéro de série unique à chaque certificat |
| `autorite/newcerts/` | Archive automatique de chaque certificat signé |
| `autorite/crl/` & `crlnumber` | Répertoire et compteur pour les listes de révocation (CRL) |

### Fichier de configuration (`autorite/openssl.cnf`)
```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir = ./autorite
database = \$dir/index.txt
new_certs_dir = \$dir/newcerts
serial = \$dir/serial
crlnumber = \$dir/crlnumber
certificate = \$dir/certificats/ca-intermediaire.crt
private_key = \$dir/cles/ca-orleans.key
default_md = sha256
default_days = 90
default_crl_days = 7
policy = policy_server
copy_extensions = none
unique_subject = no

[ policy_server ]
countryName = optional
organizationName = optional
commonName = supplied
```

> **Sécurité des extensions (`copy_extensions = none`) :** La CA ne recopie jamais aveuglément les extensions demandées dans une CSR. C'est l'administrateur de la CA qui impose les extensions autorisées via un fichier dédié lors de la signature.