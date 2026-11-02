# Instructions de travail pour les assistants

Ce fork de `snesrev/smw` vise à conserver une version jouable de Super Mario World tout en ajoutant progressivement des fonctionnalités PC. Langue de travail : français.

## Skills du dépôt

Avant une tâche liée à une fonctionnalité, lire le skill correspondant et suivre ses références, ses étapes et ses critères de validation :

- **Affichage 16:9 / rendu étendu / caméra / sprites / HUD** : [`.agents/skills/smw-widescreen/SKILL.md`](.agents/skills/smw-widescreen/SKILL.md)
- **Menu intégré / overlay / pause / navigation clavier-manette / réglages** : [`.agents/skills/integrated-menu/SKILL.md`](.agents/skills/integrated-menu/SKILL.md)

Si la tâche touche les deux domaines, consulter les deux skills. Leurs fichiers sont de la documentation versionnée : leur présence ne garantit pas qu'une plateforme les charge automatiquement. Les lire explicitement lorsque nécessaire.

## Règles du projet

1. Conserver `main` stable ; travailler sur une branche de fonctionnalité. Branche initiale : `feature/widescreen-16-9`.
2. Apporter des changements petits, vérifiables et réversibles ; séparer la documentation, les prototypes et les corrections du moteur.
3. Le mode d'affichage original 256×224 doit rester disponible ; un 16:9 correct montre davantage de jeu et **ne doit pas** étirer Mario.
4. Ne jamais ajouter de ROM Nintendo, d'assets extraits, de sauvegardes, d'exécutables ou de bibliothèques binaires téléchargées. Vérifier `.gitignore` et `third_party/.gitignore` avant tout commit.
5. Garder les notices de licence. Étudier les projets externes comme références ; ne pas copier leur code sans vérifier compatibilité des licences et attribution.
6. Ne jamais prétendre avoir compilé/testé sous Windows sans exécution réelle. Les validations manuelles sur le PC du propriétaire doivent être demandées après les modifications.
7. Quand une hypothèse est invalidée, actualiser les notes du skill correspondant avec faits, liens, limitations et décisions.

## Synchronisation Git obligatoire après chaque intervention

**Exigence du propriétaire :** après **chaque opération de l'assistant qui modifie le dépôt distant** (code, documentation, skill, configuration, commit ou fusion), terminer la réponse par une commande PowerShell **prête à copier-coller** pour récupérer le changement sur son PC.

- Indiquer explicitement le **nom de la branche modifiée** et, si possible, le lien ou le SHA du commit.
- Donner un **bloc de code `powershell` complet**, comprenant le chemin local connu, `git fetch origin`, `git switch <branche>` et `git pull --ff-only origin <branche>`. Adapter le nom de branche à l'opération réelle : ne jamais supposer que c'est toujours `main`.
- L'utilisateur travaille dans `X:\GitHub project\smw` sous Windows PowerShell. Utiliser des commandes Git simples et compatibles.
- Pour la branche de travail actuelle, le bloc recommandé est :

```powershell
cd "X:\GitHub project\smw"
git fetch origin
git switch feature/widescreen-16-9
git pull --ff-only origin feature/widescreen-16-9
```

- Si la branche locale n'existe pas, `git switch <branche>` peut la créer automatiquement depuis `origin/<branche>` lorsqu'elle est unique ; sinon fournir la variante `git switch --track origin/<branche>`.
- Ne pas indiquer de `git pull` comme s'il y avait du nouveau si aucun changement n'a été publié ; signaler alors qu'aucune synchronisation n'est nécessaire.
- Ne jamais proposer de `git reset --hard`, `git clean -fd`, de push forcé ou de commande pouvant écraser ses fichiers locaux sans justification et accord explicite. En cas de modifications locales, signaler le risque avant le changement de branche.
- Si la compilation ou des tests manuels sont nécessaires après le pull, les proposer **après** le bloc Git.

## Compilation et contrôle locaux (PowerShell)

```powershell
cd "X:\GitHub project\smw"
git fetch origin
git switch feature/widescreen-16-9
git pull --ff-only origin feature/widescreen-16-9
.\run_with_tcc.bat
```

La première fois, si la branche distante n'est pas encore locale : `git switch --track origin/feature/widescreen-16-9` (après `git fetch origin`). Ne pas exécuter aveuglément `git switch` si des changements locaux non commités risquent d'être écrasés.

Prérequis locaux déjà utilisés : Python 3.11, TCC 64 bits dans `third_party/tcc/`, SDL2 dans `third_party/SDL2-2.28.1/` et `smw_assets.dat` généré à partir de la ROM personnelle. Ces fichiers ne sont pas publiés.
