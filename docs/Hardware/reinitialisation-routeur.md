# Réinitialisation d'un routeur Cisco 1921 

## Objectif

Cette procédure décrit la réinitialisation d'un **Cisco ISR 1921** à l'aide d'une connexion console et de la combinaison de touches **`Ctrl` + `Pause`**.

# Connexion au routeur

Connecter le câble console au routeur puis ouvrir le logiciel de terminal.

Mettre le routeur sous tension.

Pendant le démarrage,Le démarrage doit être interrompu pour accéder au mode ROMMON.

# Interrompre le démarrage avec `Ctrl` + `Pause`

Pendant les premières secondes du démarrage, envoyer la combinaison :

```
Ctrl + Pause
```

Selon le logiciel utilisé, la touche `Pause/Break` peut être appelée :

* `Pause`
* `Break`
* `Break/Pause`

Si l'interruption est prise en compte, le routeur s'arrête dans le **ROMMON**.

Le prompt doit ressembler à :

```
rommon 1 >
```

# Démarrer le routeur et charger le startup-config

Pour récupurer l'accès au routeur tout en démarrant sur le startup-config

```
rommon 1 > confreg 0x2102 
rommon 2 > reset
```

# Démarrage d'IOS

Le routeur redémarre.

Attendre le chargement complet d'IOS.

Le routeur peut afficher :

```
Would you like to enter the initial configuration dialog? [yes/no]:
```

Répondre :

```
no
```

Puis :

```
Press RETURN to get started!
```

Appuyer sur `Entrée`.

Le prompt doit être similaire à :

```
Router>
```
