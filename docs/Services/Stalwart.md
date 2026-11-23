# Stalwart

## Préalable

Tout d'abord on va préparer la zone DNS du domaine.On va ajouter un enregistrement A pour le serveur et un enregistrement MX pointant vers ce nom :

```
@       IN MX 10 mail.orleans.sportludique.fr.
mail    IN A     <IP_Serveur_Mail>
```

Et on vérifie que l'enregistrement est bien pris en compte

```
dig mail.ville.sportludique.fr A +short
dig ville.sportludique.fr MX +short
```

## Installation Stalwart

Stalwart ne s'installe pas avec la commande ***sudo apt install***. Alors on l'installe avec un script fournit par Stalwart et sera enregistré dans un fichier nommé ***install.sh***.

```
curl --proto '=https' --tlsv1.2 -sSf https://get.stalw.art/install.sh -o install.sh
```

Ensuite on peut exécuter le fichier

```
sudo sh install.sh
```

On verifie l'installation de Stalwart :

```
sudo systemctl status stalwart
```

Et on regarde les ports ouverts :

```
sudo ss -lntp
```

## Configuration Stalwart

Une fois Stalwart installé, la configuration ce fait via une interface web accessible avec l'ip de la machine et vous devez arriver sur cette page :

![page_de_connexion](../image_stal/connexion_stal.png)

Vous retrouverez l'identifiant et le mot de passe pour la première connexion dans les logs de stalwart 
**!Mot de passe de l'user admin à modifier!**

```
journalctl -u stalwart -n 200
```
![journal](../image_stal/journalctl_stal.png)

Une fois connecté en **admin** vous pouvez commencer la configuration.

## Server Identity

La première étape est de :
- mettre le nom du serveur (mail.orleans.sportludique.fr)

- mettre le nom de domaine (orleans.sportludique.fr)

![Server_Identity](../image_stal/Server_identity_stal.png)

## Stockage

La seconde étape est de choisir un type de stockage. Dans notre cas on va utiliser SQLite.

SQLite va chercher un fichier où il peut écrire dedans, pour cela on va créer un fichier dans le répertoire /var/lib/stalwart/.

![logs](../image_stal/db_stal.png)

Lors de l'installation de Stalwart, un user et un groupe se nommant **Stalwart**. Il faut donc attribuer la propriété a l'user et au groupe Stalwart au chemin /var/lib/stalwart/"nom".db

![droit](../image_stal/droit_stal.png)

Sinon vous aurez ce message d'erreur :

![fail](../image_stal/fail_droit_stal.png)

## Account Directory

Par défaut
![account](../image_stal/account_directory_stal.png)

## Logs

Cette étape sert à choisir le type de logs que vous souhaitez avoir. Dans notre cas on veut un fichier de logs qui sera nommé **stalwart.log** dans le chemin ***/var/log/stalwart/*** avec une frequence journalier.

![log](../image_stal/logs_stal.png)

## DNS

Etant donné que nous avons notre propre serveur DNS on choisit un management manuel.

![dns](../image_stal/dns_stal.png)

Après la configuration faite, redemarrez le service :
```
sudo systemctl restart stalwart
```

## Administration

Maintenant que Stalwart est configuré on peut commencer a l'administrer et le parametrer a nos besoins.

Si votre DNS a bien été configuré avec les bons enregistrements, (voir [Préalable](#préalable)) vous pourrez avoir accès a l'interface web en https, dans notre cas : 
```
https://mail.orleans.sportludique.fr
```

Une fois connecté, vérifiez le domaine dans :
**Management > Domains > Domains** 
 
 et verifiez que c'est bien **ENABLED**

 ![domains](../image_stal/domains_stal.png)


 ## Ouverture des ports

 Dans `Settings › Network › Listeners`, créer (ou modifier) deux écouteurs sans chiffrement :

| Champ | Soumission SMTP | Consultation IMAP |
| :--- | :--- | :--- |
| **Listener Id** | `submission` | `imap` |
| **Protocol** | `SMTP (smtp)` | `IMAP (imap)` |
| **Bind addresses** | `ipv4-service:587` | `ipv4-service:143` |
| **Enable TLS (`useTls`)** | Désactivé | Désactivé |
| **Implicit TLS (`implicitTls`)** | Désactivé | Désactivé |