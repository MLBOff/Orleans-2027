# Renumérotation du switch 2 en switch 1

## Contexte de la situation

Lors de la remise en service d’un switch dans une pile (stack), il peut arriver que certaines configurations précédemment enregistrées soient conservées malgré une remise aux paramètres d’usine.

Dans le cas présent, le **switch 2** conserve une configuration de provisionnement. Il apparaît alors comme **« Switch 2 Provisionnel »**, alors que l’objectif est de repartir avec une configuration propre et de disposer uniquement des équipements réellement présents dans la pile.

Pour corriger cette situation, il est nécessaire de **renuméroter le switch 2 en switch 1**, puis de supprimer la configuration de provisionnement restante.

## Procédure de renumérotation du switch 2

Se connecter au switch et passer en mode privilégié :

```text
enable
configure terminal
switch 2 renumber 1
end
reload```

## Vérification après redémarrage

Suite au redémarrage du switch vérifier si la modification a bien eu lieu il devrait y avoir écrit :
    - Switch 1
    - Switch 2

Si il y a marquer **« Switch 2 Provisionnel »** alors supprimer la avec ces commandes:

```text
enable
configure terminal
no switch 2 provision
end
reload```

## Résultat attndu

Après suppression du provisionnement, la configuration doit être nettoyée.

Le switch doit désormais apparaître uniquement comme :

```Switch 1 Ready```

Cette opération permet donc de **réinitialiser la numérotation du switch et de supprimer les anciennes informations** de provisionnement persistantes.