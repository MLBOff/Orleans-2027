# Démarche reinitalisation Switch Cisco Catalyst 3750

## Prérequis

* Accès console ou SSH au switch
* Accès au mode privilégié `enable`
* Câble console en cas de perte d'accès réseau
* Sauvegarde de la configuration si celle-ci doit être conservée
* Accès physique au switch pour la procédure avec le bouton `MODE`

## Méthode avec le bouton MODE

Cette méthode est utile lorsque l'accès au CLI n'est plus possible, notamment en cas de mot de passe oublié.

### Mise hors tension

Débrancher l'alimentation du switch.

### Maintenir le bouton MODE

Maintenir le bouton **MODE** enfoncé pendant la remise sous tension du switch.

Sur le Catalyst 3750, cette procédure permet d'accéder au bootloader `Switch:`. Cisco documente l'utilisation du bouton MODE et d'une connexion console à **9600 bps** pour accéder au bootloader.

Lorsque le prompt suivant apparaît :

```
Switch:
```

le switch est dans le bootloader.

---

### Initialiser la mémoire flash

On va accéder à la mémoire flash

```
Switch: flash_init
```

Puis on supprime le fichier de configuration et la base des VLAN

```
Switch: delete flash:config.text
Switch: delete flash:vlan.dat
```

### Redémarrer le routeur

Après avoir supprimé le fichier de configuration et la base des VLAN, on redémarre le routeur

```
Switch: reload
```

Si le dialogue de configuration apparaît :

```
Continue with configuration dialog? [yes/no]:
```

répondre :

```
no
```