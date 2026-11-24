## Guide d'installation et de configuration d'iTop sur Debian 

Ce document détaille l'installation complète d'iTop sur Debian dans une architecture séparée en deux serveurs distincts :
* **Serveur Web (Apache / PHP / iTop)** : hébergé dans le réseau **DMZ Public**.
* **Serveur de Base de Données (MariaDB)** : hébergé dans le réseau **DMZ Privé**.

Il couvre également la configuration initiale de l'environnement (organisation, utilisateurs, catalogue de services, modèles de support) ainsi que les solutions aux problèmes rencontrés lors de la mise en place.

---

## Partie 1 : Installation du Serveur de Base de Données (DMZ Privée)

Les opérations suivantes sont à exécuter avec `sudo`sur le serveur Debian situé dans le réseau **DMZ Privé**.

### 1. Prérequis système et installation de MariaDB

Mettez à jour le système et installez le serveur MariaDB :

```bash
apt update && apt upgrade -y
apt install -y mariadb-server
systemctl enable --now mariadb
```

### 2. Autoriser les connexions distantes depuis la DMZ Publique

Par défaut, MariaDB n'écoute que sur l'interface locale (`127.0.0.1`). Pour que le serveur web de la **DMZ Publique** puisse s'y connecter, il faut modifier l'adresse d'écoute :

1. Ouvrez le fichier de configuration de MariaDB :
```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

2. Repérez la ligne `bind-address` et remplacez `127.0.0.1` par l'adresse privée :
```ini
bind-address = "ip privée"
```

3. Redémarrez MariaDB pour appliquer le changement :
```bash
systemctl restart mariadb
```

> **Note réseau / Pare-feu :** Assurez-vous que les règles de routage et de pare-feu entre les deux zones autorisent le flux **TCP/3306** uniquement depuis l'IP du serveur Web (**DMZ Publique**) vers l'IP du serveur DB (**DMZ Privée**).

### 3. Création de la base de données et de l'utilisateur distant

Connectez-vous à MariaDB (`mysql -u root -p`) et créez la base `itop` ainsi que l'utilisateur autorisé à se connecter depuis l'adresse IP du serveur Web (remplacez `<IP_SERVEUR_WEB_DMZ_PUBLIC>` par l'IP exacte de votre serveur web, ou utilisez le sous-réseau ex. `192.168.10.%`) :

```sql
CREATE DATABASE itop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'itop_user'@'<IP_SERVEUR_WEB_DMZ_PUBLIC>' IDENTIFIED BY 'MotDePasseSolide123!';
GRANT ALL PRIVILEGES ON itop.* TO 'itop_user'@'<IP_SERVEUR_WEB_DMZ_PUBLIC>';
FLUSH PRIVILEGES;
EXIT;
```

---

## Partie 2 : Installation du Serveur Web iTop (DMZ Publique)

Les opérations suivantes sont à exécuter avec `sudo` sur le serveur Debian situé dans le réseau **DMZ Publique**.

### 1. Installation d'Apache, PHP et du client MariaDB

Installez le serveur web Apache, PHP avec les extensions requises par iTop (`gd`, `mbstring`, `xml`, `zip`, `mysql`, `ldap`, `soap`, `intl`, `apcu`), `graphviz`, ainsi que le client MariaDB (`mariadb-client`) pour pouvoir tester la connexion réseau vers la DMZ Privée :

```bash
apt update && apt upgrade -y
apt install -y wget unzip curl vim apache2 mariadb-client php libapache2-mod-php \
php-mysql php-gd php-xml php-mbstring php-zip php-curl \
php-ldap php-soap php-intl php-apcu graphviz
```

Activez et démarrez Apache :

```bash
systemctl enable --now apache2
```

*(Optionnel)* Vérifiez depuis le serveur Web que la connexion réseau et SQL vers la DMZ Privée fonctionne avant de déployer iTop :
```bash
mysql -h <IP_SERVEUR_DB_DMZ_PRIVEE> -u itop_user -p itop
```

### 2. Téléchargement et déploiement d'iTop

Téléchargez la dernière archive d'iTop et extrayez-la dans `/var/www/html/itop` :

```bash
cd /tmp
wget -O itop.zip "https://sourceforge.net/projects/itop/files/latest/download"
unzip itop.zip -d /tmp/itop_extract
mkdir -p /var/www/html/itop
mv /tmp/itop_extract/web/* /var/www/html/itop/
```

Appliquez les permissions appropriées pour l'utilisateur Apache (`www-data`) :

```bash
chown -R www-data:www-data /var/www/html/itop
chmod -R 755 /var/www/html/itop
```

### 3. Configuration du VirtualHost Apache

Si vous utilisez un nom de domaine dédié (ex. `itop.orleans.sportludique.fr` sur le port `8080`) :

1. Ajoutez `Listen 8080` dans `/etc/apache2/ports.conf`.

2. Créez `/etc/apache2/sites-available/itop.conf` :

```apache
<VirtualHost *:8080>
    ServerName itop.orleans.sportludique.fr
    DocumentRoot /var/www/html/itop

    <Directory /var/www/html/itop>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/itop_error.log
    CustomLog ${APACHE_LOG_DIR}/itop_access.log combined
</VirtualHost>
```

3. Activez le site et redémarrez Apache :

```bash
a2ensite itop.conf
a2enmod rewrite
systemctl reload apache2
```

### 4. Finalisation via l'assistant web

1. Ouvrez votre navigateur sur `http://itop.orleans.sportludique.fr:8080/setup`.
2. Vérifiez que tous les prérequis PHP sont au vert.
3. Choisissez **Installer un nouvel iTop**.
4. Renseignez les paramètres de connexion vers le serveur de la **DMZ Privée** :
   * **Nom du serveur :** `<IP_SERVEUR_DB_DMZ_PRIVEE>` (ne pas mettre `localhost`)
   * **Login :** `itop_user`
   * **Mot de passe :** `MotDePasseSolide123!`
   * **Base de données :** Sélectionnez la base existante `itop`
5. Créez le compte administrateur principal (`admin`).
6. Choisissez les modules (Gestion des services, Gestion des incidents, Gestion des demandes utilisateur, Portail utilisateur standard).
7. Lancez l'installation.

---

## Partie 3 : Configuration initiale de l'environnement

Pour qu'iTop soit opérationnel en production avec une organisation propre (exemple : **Orleans Sportludique**), respectez l'ordre de création suivant :

1. **Organisation :**
   * Allez dans **Administration des données > Organisations** et créez votre organisation principale (`Orleans Sportludique`).

2. **Personnes et Comptes Utilisateurs :**
   * Créez les fiches **Personne** (via *Administration des données > Contacts > Personnes*) rattachées à l'organisation `Orleans Sportludique` (ex. `Matheo` pour l'admin, `client test` pour l'utilisateur portail).
   * Créez les **Comptes utilisateurs** (via *Administration > Comptes utilisateurs* > **Utilisateur iTop**) :
     * Compte admin : profil `Administrator`.
     * Compte utilisateur standard : profil **uniquement `Portal user`**.

3. **Équipes et Techniciens :**
   * Créez une **Équipe** (ex. *Support IT*) rattachée à `Orleans Sportludique`.
   * Dans l'onglet **Membres** de l'équipe, ajoutez les fiches Personne des techniciens habilités (`Matheo`).

4. **Modèle de support (Delivery Model) :**
   * Créez un **Modèle de support** (dans *Gestion des services > Modèles de support*) pour `Orleans Sportludique`.
   * Ajoutez votre **Équipe** dans l'onglet **Contacts** du modèle de support.
   * Retournez sur la fiche de l'organisation `Orleans Sportludique` et associez-lui ce **Modèle de support**.

5. **Catalogue de services et Contrat client :**
   * Créez une **Famille de services** (ex. *Informatique*).
   * Créez un **Service** (ex. *Support IT*) en statut **En production**.
   * Dans l'onglet **Sous-catégories de service** du service, créez :
     * Une sous-catégorie de type **Demande utilisateur** (statut **En production**).
     * Une sous-catégorie de type **Incident** (statut **En production**).
   * Créez un **Contrat client** (*Gestion des services > Contrats clients*) avec `Client : Orleans Sportludique`, `Fournisseur : Orleans Sportludique`, `État : En production`, et ajoutez le service dans son onglet **Services**.

---

## Partie 4 : Problèmes rencontrés et solutions (Dépannage)

Voici le récapitulatif des blocages rencontrés durant la configuration et la manière de les résoudre :

### 1. Impossible de supprimer l'organisation ou la personne par défaut
* **Symptôme :** iTop bloque la suppression de l'organisation ou de la personne d'exemple installée par défaut.
* **Cause :** Le compte `admin` initial est rattaché à cette personne par défaut (dépendance bloquante en base de données).
* **Solution :** Créer d'abord la nouvelle organisation (`Orleans Sportludique`) et votre nouvelle fiche Personne (`Matheo`), modifier l'utilisateur `admin` pour le rattacher à cette nouvelle Personne, puis supprimer les objets par défaut devenus orphelins.

### 2. Impossible de se connecter avec l'utilisateur portail (`test`) ou erreur `Missing argument 'exec_module'`
* **Symptôme :**
  * Tentative de connexion sur `/pages/UI.php` refusée ou bouclée.
  * Accès direct à `/pages/exec.php` affichant l'erreur `Missing argument 'exec_module'`.
* **Causes :**
  * `/pages/UI.php` est réservé aux profils back-office (administrateurs/agents).
  * `/pages/exec.php` nécessite le paramètre spécifiant le module du portail.
  * Le profil `Portal power user` seul ne suffit pas toujours sans le profil de base `Portal user`.
* **Solutions :**
  * Attribuer **uniquement le profil `Portal user`** (ou `Portal user` + `Portal power user`) à l'utilisateur `test`, en vérifiant que son compte est de type **Utilisateur iTop** et à l'état **Actif**.
  * Utiliser l'URL complète du portail : `http://<DOMAINE>:8080/pages/exec.php?exec_module=itop-portal&exec_page=index.php` (ou la racine `/` en navigation privée).

### 3. Erreur portail : `Could not load contact related to connected user`
* **Symptôme :** Page d'erreur *« Oups ! Une erreur est survenue »* mentionnant : `Make sure the contact's organization is among the user's allowed organizations`.
* **Cause :** L'utilisateur `test` avait une restriction dans son onglet **Organisations permises** qui ne contenait pas l'organisation de sa fiche Personne (`Orleans Sportludique`).
* **Solution :** Sur la fiche de l'utilisateur `test`, ouvrir l'onglet **Organisations permises** et **le laisser totalement vide** (aucune restriction) ou y ajouter explicitement `Orleans Sportludique`.

### 4. Erreur fatale : `No result for the single row query` lors de la connexion
* **Symptôme :** Écran blanc *« iTop - Erreur Fatale : No result for the single row query »* sur `/pages/UI.php` après avoir tenté de se connecter avec `test`, bloquant même le retour à l'écran de login.
* **Causes :**
  * Le cookie de session du navigateur est resté bloqué sur la tentative de connexion de `test` vers l'interface d'administration (`/pages/UI.php`).
  * Ou la fiche Personne liée à l'utilisateur a été supprimée/mal rattachée.
* **Solutions :**
  * Forcer la déconnexion en appelant l'URL `/pages/logoff.php` ou supprimer les cookies du site dans le navigateur (et tester le compte portail dans une fenêtre de navigation privée).
  * Vérifier que la fiche Personne (`client test`) liée à l'utilisateur `test` existe bien, est **Active** et rattachée à `Orleans Sportludique`.
  * *(En cas de blocage sur le compte `admin` lui-même, corriger le `contactid` directement dans la table `priv_user` sur le serveur MariaDB de la DMZ Privée).*

### 5. Page « Nouvelle demande » vide (`Aucun élément`) sur le portail
* **Symptôme :** L'utilisateur connecté au portail voit le menu *Nouvelle demande*, mais la liste affiche *« Aucun élément »*.
* **Cause :** iTop masque les services tant que toute la chaîne contractuelle n'est pas publiée.
* **Solution :** Vérifier que **tous** les maillons suivants sont présents et à l'état **En production** :
  1. Le **Service**.
  2. Au moins une **Sous-catégorie de service** rattachée à ce service.
  3. Un **Contrat client** liant le `Client` (organisation du demandeur) au `Service`, sans date de début future.

### 6. Listes déroulantes « Équipe » et « Agent » vides lors de l'assignation d'un ticket (`R-000002`)
* **Symptôme :** En cliquant sur **Assigner** sur un ticket, le menu **Équipe** ne propose aucun choix (`-- choisir une valeur --`).
* **Cause :** iTop utilise le **Modèle de support** (*Delivery Model*) de l'organisation cliente pour déterminer quelles équipes et quels agents sont autorisés à traiter les tickets.
* **Solution :**
  1. Créer une **Équipe** et ajouter les techniciens dans son onglet **Membres**.
  2. Créer un **Modèle de support** et ajouter cette équipe dans son onglet **Contacts**.
  3. Rattacher ce **Modèle de support** à l'organisation cliente (`Orleans Sportludique`).

### 7. Différence d'icône/pastille verte entre Incident (`I-000003`) et Demande utilisateur (`R-000002`)
* **Symptôme :** Dans la liste des demandes résolues sur le portail, une pastille verte apparaît à côté de l'état *Résolue* sur `R-000002` mais pas sur `I-000003`.
* **Explication :** Le cycle de vie et le rendu visuel des états dans le portail diffèrent par défaut entre la classe `UserRequest` (Demande utilisateur) et la classe `Incident` selon les transitions (stimuli) disponibles pour l'utilisateur portail (comme la validation/fermeture). Cela n'empêche pas le bon fonctionnement ni la clôture du ticket.