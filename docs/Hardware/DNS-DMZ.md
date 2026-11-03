# Focus Technique : Rôle des fichiers et analyse approfondie de la configuration BIND9

Cette section détaille l'organisation des fichiers de configuration du serveur d'autorité `ORL-SRV-DNS-DMZ` ainsi que leur logique interne.

---

## 1. Cartographie et rôle des fichiers de l'infrastructure

Avant d'analyser le code, voici une vue d'ensemble des fichiers édictés sur le serveur d'autorité et la mission précise de chacun :

| Chemin du fichier | Nom du fichier | Rôle et responsabilité au sein de l'infrastructure |
| :--- | :--- | :--- |
| `/etc/bind/named.conf` | **Fichier d'ordonnancement** | Point d'entrée principal du service BIND9. Il ne contient aucune règle de zone, mais orchestre et charge les autres sous-fichiers de configuration via la directive `include`. |
| `/etc/bind/named.conf.local` | **Gestionnaire des Vues (Split DNS)** | C'est le cerveau du Split DNS. Il définit les listes de contrôle d'accès (ACL) pour identifier la provenance des clients et aiguille les requêtes vers la vue interne (`inside`) ou externe (`outside`). |
| `/etc/bind/db.orleans.sportludique.fr.interne` | **Base de zone Interne** | Base de données DNS exploitée par la vue `inside`. Elle répertorie les correspondances noms/IP privées (`192.168.x.x`) pour les utilisateurs du LAN et de la DMZ. |
| `/etc/bind/db.orleans.sportludique.fr.externe` | **Base de zone Externe** | Base de données DNS exploitée par la vue `outside`. Elle répertorie les correspondances noms/IP publiques (`221.x.x.x` / `183.x.x.x`) pour les requêtes provenant d'Internet. |

---

## 2. Décryptage du fichier d'ordonnancement principal (`/etc/bind/named.conf`)

Ce fichier est lu en premier par le démon BIND9 (`named`) à son démarrage.

### Contenu complet du fichier :
```text
// This is the primary configuration file for the BIND DNS server named.
//
// Please read /usr/share/doc/bind9/README.Debian for information on the
// structure of BIND configuration files in Debian, *BEFORE* you customize
// this configuration file.
//
// If you are just adding zones, please do that in /etc/bind/named.conf.local

include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
#include "/etc/bind/named.conf.default-zones";
```

### Explications détaillées :
*   **`include "/etc/bind/named.conf.options";`** : Cette directive charge les paramètres globaux et comportementaux du serveur DNS (les ports d'écoute, les ACL de récursion, les mécanismes de sécurité comme le masquage de version ou la protection contre les attaques par déni de service).
*   **`include "/etc/bind/named.conf.local";`** : Cette ligne permet de séparer la configuration d'usine des modifications de l'administrateur. Dans notre cas, elle charge toute la structure des vues (Split DNS).
*   **`#include "/etc/bind/named.conf.default-zones";` (Le piège du projet)** : Cette ligne est commentée avec un caractère `#`. Par défaut, ce fichier contient les zones de bouclage local indispensables à une installation standard (telles que `localhost`, `127.in-addr.arpa`, etc.).
    *   *Pourquoi l'avoir désactivé ?* BIND9 impose une règle de syntaxe absolue : **dès lors que la directive `view` est utilisée, TOUTES les zones gérées par le serveur doivent obligatoirement être déclarées à l'intérieur d'un bloc `view`**. Si cette ligne était restée active, ces zones par défaut se seraient retrouvées "hors vue", provoquant une erreur de syntaxe fatale au démarrage du service (`error: 'zones' found outside of a view`).

---

## 3. Logique de routage et filtrage (`/etc/bind/named.conf.local`)

Ce fichier instancie la logique algorithmique du Split DNS. Il applique un traitement conditionnel basé sur la provenance (l'adresse IP source) des requêtes.

### Contenu complet du fichier :
```text
acl "lan" {
        127.0.0.1;
        192.168.149.0/24;
        172.28.0.0/19;
};

view "inside" {
        match-clients {
                lan;
        };

        zone "orleans.sportludique.fr" {
                type master;
                file "/etc/bind/db.orleans.sportludique.fr.interne";
        };
};

view "outside" {
        match-clients {
                any;
        };

        zone "orleans.sportludique.fr" {
                type master;
                file "/etc/bind/db.orleans.sportludique.fr.externe";
        };
};
```

### Explications détaillées :
*   **`acl "lan"`** : Crée une liste de contrôle d'accès (Access Control List) nommée `"lan"`. Elle sert d'alias pour regrouper un ensemble d'adresses réseau de confiance afin de simplifier la maintenance du fichier.
    *   `127.0.0.1;` : Autorise la boucle locale du serveur DNS lui-même à interroger et matcher sa propre vue interne (utile pour les diagnostics locaux).
    *   `192.168.149.0/24;` : Représente le sous-réseau complet de la **DMZ**. Cela garantit que les serveurs applicatifs (Web, messagerie, etc.) situés dans la DMZ reçoivent les réponses DNS avec l'adressage privé.
    *   `172.28.0.0/19;` : Correspond à la plage réseau globale de vos **clients LAN** (transmise à BIND9 par l'intermédiaire du résolveur Unbound).
*   **`view "inside"` et `match-clients { lan; };`** : Déclare la vue réservée à l'infrastructure privée. BIND9 analyse l'adresse IP source du paquet UDP/TCP DNS entrant. Si cette IP appartient à l'ACL `"lan"`, le serveur bascule la requête exclusivement dans ce bloc et ignore le reste du fichier.
    *   `type master;` : Spécifie que ce serveur est le maître d'autorité sur cette zone (il détient le fichier original).
    *   `file "/etc/bind/db.orleans.sportludique.fr.interne";` : Pointe vers le fichier de zone interne.
*   **`view "outside"` et `match-clients { any; };`** : Le mot-clé `any` agit comme une règle par défaut (*catch-all*). Il désigne n'importe quelle adresse IP sur Internet.
    *   *Pourquoi l'ordre de déclaration est-il crucial ?* BIND9 analyse les blocs `view` de manière séquentielle, du haut vers le bas. Lorsqu'une requête arrive, il vérifie d'abord si elle matche l'ACL `lan`. Si c'est le cas, elle est traitée par la vue interne. Si elle ne provient pas du LAN, il passe à la vue suivante (`outside`). Comme cette dernière accepte `any`, toutes les requêtes externes (Internet, réseau enseignant) y sont capturées. **Si vous inversiez l'ordre des vues, les clients du LAN correspondraient immédiatement au `any` de la vue externe et recevraient des adresses IP publiques, brisant le Split DNS.**

---

## 4. Structure sémantique des fichiers de zone (`db.orleans...`)

Les fichiers de zone traduisent les noms d'hôtes en adresses IP. Bien que leurs contenus (les adresses IP) diffèrent pour les deux populations de clients, leur structure syntaxique reste identique.

### A. Contenu complet de la zone interne (`/etc/bind/db.orleans.sportludique.fr.interne`)
```text
$TTL    86400
@       IN      SOA     ns2.orleans.sportludique.fr. hostmaster.orleans.sportludique.fr. (
                              2026101405 ; Serial
                                    3600 ; Refresh
                                     900 ; Retry
                                 1209600 ; Expire
                                   86400 ) ; Negative Cache TTL

@       IN      NS      ns2.orleans.sportludique.fr.

ns2     IN      A       192.168.149.4

@       IN      A       192.168.149.5
www     IN      A       192.168.149.5
```

### B. Contenu complet de la zone externe (`/etc/bind/db.orleans.sportludique.fr.externe`)
```text
$TTL    86400
@       IN      SOA     ns2.orleans.sportludique.fr. hostmaster.orleans.sportludique.fr. (
                              2026101405 ; Serial
                                    3600 ; Refresh
                                     900 ; Retry
                                 1209600 ; Expire
                                   86400 ) ; Negative Cache TTL

@       IN      NS      ns2.orleans.sportludique.fr.
@       IN      NS      ns1.orleans.sportludique.fr.

ns2     IN      A       221.87.145.1
ns1     IN      A       183.44.45.2

@       IN      A       221.87.145.1
@       IN      A       183.44.45.2

www     IN      A       221.87.145.1
www     IN      A       183.44.45.2
```

### Explications détaillées des composants de zone :
*  *   **Les enregistrements de serveurs de noms (NS) :**
    *   `@ IN NS ns2.orleans.sportludique.fr.` : (Présent dans la zone interne et externe). Cet enregistrement indique que la machine nommée `ns2` possède l'autorité officielle sur la zone `orleans.sportludique.fr.` et a le droit de répondre aux clients.
    *   `@ IN NS ns1.orleans.sportludique.fr.` : (Présent uniquement dans la zone externe). Déclare un deuxième serveur de noms (`ns1`) visible depuis Internet. Cela permet d'assurer la haute disponibilité (redondance) : si un serveur DNS externe tombe en panne, l'autre prend le relais pour résoudre le nom de domaine.

*   **L'enregistrement d'autorité racine de la zone (`@ IN A ...`) :**
    *   `@ IN A 192.168.149.5` : (Zone interne). Fait pointer le domaine racine "nu" (sans `www`, soit juste `orleans.sportludique.fr`) directement vers l'adresse IP privée de votre serveur applicatif ou serveur web interne.
    *   `@ IN A 221.87.145.1` et `@ IN A 183.44.45.2` : (Zone externe). Permet aux utilisateurs d'Internet d'accéder à votre site web de l'extérieur sans taper les trois `www`. Le fait de déclarer deux adresses IP distinctes permet à BIND9 de faire de la répartition de charge (*Round-Robin*).

*   **Les hôtes spécifiques (Enregistrements A) :**
    *   `ns2 IN A 192.168.149.4` : (Zone interne). Enregistrement d'encapsulation appelé "Glue Record". Il associe de manière rigide le nom `ns2` (déclaré plus haut dans le SOA et le NS) à l'adresse IP privée de votre serveur DNS d'autorité en DMZ.
    *   `ns2 IN A 221.87.145.1` et `ns1 IN A 183.44.45.2` : (Zone externe). Associent les noms d'hôtes de vos serveurs DNS externes à leurs adresses IP publiques respectives afin qu'Internet puisse les localiser et les requêter.
    *   `www IN A 192.168.149.5` : (Zone interne). Résout l'adresse du serveur web pour les utilisateurs locaux. Elle renvoie vers l'IP de la DMZ privée, évitant ainsi aux paquets de sortir inutilement du réseau d'entreprise.
    *   `www IN A 221.87.145.1` et `www IN A 183.44.45.2` : (Zone externe). Offrent les points d'entrée publics pour charger le site web de l'organisation à l'extérieur du réseau de l'école.
