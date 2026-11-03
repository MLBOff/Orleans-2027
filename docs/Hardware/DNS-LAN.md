# Documentation Technique : Mise en place et vérification du Résolveur DNS Interne (LAN)

Ce guide détaille la configuration et la validation du résolveur DNS interne basé sur **Unbound** pour le réseau local (LAN) du projet SportLudique.

## 1. Objectifs de l'architecture
Le résolveur interne a pour missions de :
* Résoudre et mettre en cache les requêtes des postes du LAN.
* Rediriger les requêtes de la zone locale (`orleans.sportludique.fr.`) vers le DNS d'autorité en DMZ.
* Transmettre les requêtes globales et de la zone parente (`sportludique.fr`) vers le DNS de l'enseignant.
* Sécuriser l'accès en restreignant les requêtes au seul réseau local autorisé.

## 2. Configuration d'Unbound (`/etc/unbound/unbound.conf.d/conf.conf`)

Voici le fichier de configuration implémenté sur le serveur `ORL-SRV-DNS-LAN` :

```text
server:
    val-permissive-mode: yes

    hide-version: yes
    hide-identity: yes

    do-ip4: yes
    do-ip6: no

    do-tcp: yes
    do-udp: yes

    verbosity: 1

    interface: 172.28.141.4
    interface: 127.0.0.1

    access-control: 172.28.128.0/19 allow
    access-control: 127.0.0.1 allow

stub-zone:
    name: "orleans.sportludique.fr."
    stub-addr: 192.168.149.3

forward-zone:
    name: "sportludique.fr"
    forward-addr: 121.183.90.205
```

## 3. Explications détaillées de la configuration

### Section `server:` (Paramètres Généraux et Sécurité)
* **`val-permissive-mode: yes`** : Active le mode permissif pour la validation DNSSEC. Si la validation échoue, le résolveur renvoie tout de même la réponse au client au lieu de la bloquer, ce qui évite d'interrompre les flux en phase de test.
* **`hide-version: yes` et `hide-identity: yes`** : Mesure de durcissement (sécurité). Empêche un utilisateur malveillant de connaître la version exacte d'Unbound ou l'identité du serveur via des requêtes spécifiques, limitant ainsi la recherche de vulnérabilités connues.
* **`do-ip4: yes` / `do-ip6: no`** : Active exclusivement le traitement des requêtes en IPv4 et désactive l'IPv6, conformément au plan d'adressage du projet.
* **`do-tcp: yes` / `do-udp: yes`** : Permet à Unbound de répondre à la fois en UDP (requêtes standards de moins de 512 octets) et en TCP (port 53), indispensable pour les réponses volumineuses ou les transferts de zone.
* **`verbosity: 1`** : Niveau de journalisation standard. Il enregistre les alertes et les erreurs sans surcharger les fichiers de logs du serveur.
* **`interface:`** : Spécifie les adresses IP sur lesquelles le service Unbound écoute. Le serveur écoute sur son IP locale (`172.28.141.4`) pour servir le LAN, et sur la boucle locale (`127.0.0.1`).
* **`access-control:`** : Définit les listes de contrôle d'accès (ACL). Seuls le serveur lui-même et les machines appartenant au sous-réseau LAN (`172.28.128.0/19`) sont autorisés à interroger ce résolveur. Toutes les autres requêtes externes sont rejetées pour éviter que le serveur ne devienne un *open resolver* (vecteur d'attaques par amplification DNS).

### Section `stub-zone:` (Redirection locale d'autorité)
* **`name: "orleans.sportludique.fr."`** : Indique que pour toutes les requêtes se terminant par ce domaine, Unbound ne doit pas faire de résolution récursive classique sur Internet.
* **`stub-addr: 192.168.149.3`** : Redirige ces requêtes spécifiquement vers l'IP du serveur DNS d'autorité (BIND9) situé dans la DMZ. Une *stub-zone* est privilégiée ici car elle s'attend à interroger directement un serveur d'autorité.

### Section `forward-zone:` (Redirection conditionnelle globale)
* **`name: "sportludique.fr"`** : Concerne le domaine parent du projet.
* **`forward-addr: 121.183.90.205`** : Transmet toutes les requêtes de cette zone (ainsi que le reste des requêtes Internet externes) vers l'IP du résolveur DNS de l'enseignant.

---

## 4. Procédure de Vérification

### Validation de la syntaxe
Avant de redémarrer le service, validez l'absence d'erreurs de frappe dans le fichier de configuration :
```bash
unbound-checkconf
```

### Redémarrage du service
Si la syntaxe est correcte, appliquez les modifications :
```bash
sudo systemctl restart unbound
sudo systemctl status unbound
```

### Tests depuis un poste client du LAN
Depuis une machine du réseau `172.28.128.0/19`, configurez les paramètres réseau pour utiliser le résolveur `172.28.141.4`, puis effectuez les tests suivants :

1. **Test de la zone locale (DMZ)** :
   ```bash
   dig www.orleans.sportludique.fr
   ```
   *Vérification : La réponse doit correspondre à l'enregistrement configuré dans la vue interne de votre BIND9 en DMZ (IP privée).*

2. **Test de la zone pédagogique (Enseignant)** :
   ```bash
   dig sportludique.fr
   ```
   *Vérification : La requête doit être transmise avec succès au serveur `121.183.90.205` et retourner une réponse valide.*
