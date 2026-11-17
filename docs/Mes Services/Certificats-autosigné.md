# Comment créer un certificat autosigné

Un **certificat autosigné** est un certificat numérique signé par sa propre clé privée, sans passer par une Autorité de Certification (CA) tierce. Il est idéal pour les environnements de test, de développement ou internes, bien qu'il affiche une alerte de sécurité dans les navigateurs.

---

### Prérequis

* Disposer d'un accès terminal sur le serveur (Linux).
* Avoir l'outil **OpenSSL** installé (`openssl version`).
* Connaître le **nom de domaine (FQDN)** ou l'adresse IP du serveur (ex. : `www.sportludique.fr`).

---

## Méthode 1 : Création rapide en une seule commande

Cette méthode génère simultanément la **clé privée** et le **certificat autosigné** (valable 365 jours) avec l'extension **SAN** (*Subject Alternative Name*), indispensable pour les navigateurs modernes.

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout serveur.key \
  -out serveur.crt \
  -subj "/C=FR/ST=Centre-Val de Loire/L=Chartres/O=SportLudique/CN=www.sportludique.fr" \
  -addext "subjectAltName=DNS:www.sportludique.fr,DNS:sportludique.fr"
```

### Détail des options :
* `-x509` : Génère un certificat autosigné plutôt qu'une demande de signature (CSR).
* `-newkey rsa:4096` : Crée une nouvelle clé privée RSA de 4096 bits.
* `-sha256` : Utilise l'algorithme de hachage SHA-256.
* `-days 365` : Définit la durée de validité à 1 an.
* `-nodes` : (*No DES*) Ne chiffre pas la clé privée avec un mot de passe (évite de devoir le saisir à chaque redémarrage d'Apache/Nginx).
* `-keyout` / `-out` : Noms des fichiers de sortie (clé privée et certificat public).
* `-subj` : Informations d'identité du serveur (`CN` = *Common Name*).
* `-addext` : Ajoute les noms DNS autorisés dans l'extension **SAN**.

---

## Méthode 2 : Création étape par étape (Clé -> CSR -> Certificat)

Cette approche reproduit le cycle classique d'une PKI (*De la clé privée au certificat signé*).

### 1. Générer la clé privée
```bash
openssl genrsa -out serveur.key 4096
```

### 2. Créer la demande de signature de certificat (CSR)
```bash
openssl req -new -key serveur.key -out serveur.csr \
  -subj "/C=FR/ST=Centre-Val de Loire/L=Chartres/O=SportLudique/CN=www.sportludique.fr"
```

### 3. Signer le certificat avec sa propre clé privée
Créer d'abord un fichier d'extensions `san.ext` pour inclure les noms DNS :
```bash
echo "subjectAltName=DNS:www.sportludique.fr,DNS:sportludique.fr" > san.ext
```

Générer ensuite le certificat final :
```bash
openssl x509 -req -in serveur.csr -signkey serveur.key -out serveur.crt \
  -days 365 -sha256 -extfile san.ext
```

---

## Vérification et sécurisation

### Vérifier le contenu du certificat
Pour contrôler les dates de validité, l'émetteur et les noms SAN :
```bash
openssl x509 -in serveur.crt -text -noout
```

### Sécuriser les permissions (Important)
La **clé privée** ne doit jamais être lisible par d'autres utilisateurs que `root` :
```bash
chmod 600 serveur.key
chmod 644 serveur.crt
```

---

## Limites à retenir

* **Pas de chaîne de confiance :** Le navigateur ne reconnaît pas l'émetteur et affiche un avertissement de sécurité.
* **Usage limité :** À réserver aux phases de maquettage. En production ou pour le projet *SportLudique*, il faudra faire signer le CSR par l'**Autorité de Certification (CA)** interne du groupe.
