# Serveur WEB (Apache)

## 1. Objectif

L'objectif est d'installer et de configurer un serveur web Apache2 sur une machine Debian.

Apache permettra d'héberger et de publier des pages web accessibles depuis un navigateur à travers le protocole HTTP.

**Environnement :**
- Système d'exploitation : Debian
- Serveur web : Apache2
- Protocole : HTTP
- Port réseau : TCP 80
- Répertoire web par défaut : `/var/www/html`

## 2. Mise à jour du système

Avant l'installation, mettre à jour la liste des paquets :

```
sudo apt update
```

Installer les mises à jour disponibles :

```
sudo apt upgrade -y
```

## 3. Installation d'Apache2

Installer le serveur web Apache2 :

```
sudo apt install apache2 
```

Activer Apache pour qu'il démarre automatiquement au démarrage de Debian :

```
sudo systemctl enable apache2
```

Démarrer le service :

```
sudo systemctl start apache2
```

## 4. Vérification du service Apache

Vérifier que le service fonctionne correctement :

```
sudo systemctl status apache2
```

Le résultat attendu doit contenir :

```text
Active: active (running)
```

Cela confirme que le serveur Apache est démarré.

## 5. Vérification du port HTTP

Apache utilise par défaut le port TCP 80 pour les connexions HTTP.

Vérifier que le port est en écoute :

```bash
sudo ss -lntp | grep :80
```

Le résultat doit montrer qu'Apache écoute sur le port 80.

## 6. Accès à la page d'accueil Apache

Par défaut, Apache installe une page d'accueil dans le répertoire :

```
/var/www/html/
```

Depuis un navigateur web, saisir l'adresse IP du serveur :

```
http://ADRESSE_IP_DU_SERVEUR
```

Exemple :

```text
http://192.168.1.100
```

La page d'accueil par défaut d'Apache doit apparaître.

Il est également possible de tester directement depuis le serveur :

```
curl -I http://localhost
```

Une réponse HTTP `200 OK` indique que le serveur répond correctement.

## 7. Organisation des fichiers Apache

Sur Debian, les principaux fichiers et répertoires sont organisés de la manière suivante :

| Chemin | Description |
|---|---|
| `/etc/apache2/` | Répertoire principal de configuration |
| `/etc/apache2/apache2.conf` | Configuration générale d'Apache |
| `/etc/apache2/ports.conf` | Configuration des ports d'écoute |
| `/etc/apache2/sites-available/` | Configurations des sites disponibles |
| `/etc/apache2/sites-enabled/` | Configurations des sites activés |
| `/var/www/html/` | Répertoire web par défaut |
| `/var/log/apache2/` | Journaux du serveur Apache |

## 8. Création d'une page web de test

La page d'accueil par défaut peut être remplacée par une page HTML personnalisée.

Ouvrir le fichier :

```
sudo nano /var/www/html/index.html
```

Ajouter le contenu suivant :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Mon serveur Apache</title>
</head>
<body>
    <h1>Bienvenue sur mon serveur web !</h1>
    <p>Apache fonctionne correctement sur Debian.</p>
</body>
</html>
```

Enregistrer le fichier.

Définir les permissions :

```bash
sudo chmod 644 /var/www/html/index.html
```

Depuis un navigateur, actualiser la page :

```text
http://ADRESSE_IP_DU_SERVEUR
```

La nouvelle page HTML doit maintenant apparaître.

## 9. Configuration du pare-feu

Si un pare-feu est installé et actif sur le serveur ou sur le réseau, il faut autoriser les connexions HTTP.

Le trafic à autoriser est :

| Source | Destination | Protocole | Port |
|---|---|---|---|
| Clients autorisés | Serveur Debian | TCP | 80 |

Si UFW est utilisé, autoriser HTTP avec :

```bash
sudo ufw allow 80/tcp
```

Vérifier les règles :

```bash
sudo ufw status
```

## 10. Consultation des journaux Apache

Apache conserve des journaux permettant de suivre les connexions et les erreurs.

**Journal des accès :**

```bash
sudo tail -f /var/log/apache2/access.log
```

**Journal des erreurs :**

```bash
sudo tail -f /var/log/apache2/error.log
```

Ces fichiers permettent de diagnostiquer les problèmes et de consulter les requêtes reçues par le serveur.


