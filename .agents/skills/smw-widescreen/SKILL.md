---
name: smw-widescreen
description: Concevoir, implémenter et vérifier un vrai rendu 16:9 natif de Super Mario World dans le fork snesrev/smw, sans déformer les sprites ni modifier le mode original.
---

# SMW Widescreen : skill de développement

## Quand l'utiliser

Lire ce fichier avant de modifier le viewport, la résolution interne, le PPU, les sprites/OAM, le scrolling, la caméra, les collisions liées à l'écran, les ennemis, le HUD ou les paramètres 16:9. Le lire aussi avant de reprendre une idée de Zelda3, wide-snes ou SuperMarioWorldRecomp.

## Objectif et état initial

- Cible : `Reign-in-blood/smw`, branche `feature/widescreen-16-9`.
- Point de départ fonctionnel : build Windows TCC + SDL2 du port `snesrev/smw`. Jeu de base déclaré jouable de bout en bout.
- **Objectif** : vrai viewport horizontal plus large, terrain et acteurs supplémentaires visibles à gauche/droite ; Mario et sprites non étirés ; ratio original toujours accessible.
- **État lors de la création du skill** : documentation et recherche uniquement ; **aucun mode 16:9 implémenté ni testé dans ce fork**. À actualiser après chaque jalon.
- Distinguer « fenêtre 16:9 » (étirement) de « scène 16:9 » (nouvelle zone de jeu).

## Références prioritaires et ce qu'elles apportent

1. **Base du projet** : https://github.com/snesrev/smw — architecture actuelle, `src/main.c`, `src/snes/ppu.c`, `src/snes/ppu.h`, `src/opengl.c`, `src/config.c`, `smw.ini`.
2. **Port Zelda3 du même auteur** : https://github.com/snesrev/zelda3 — `src/config.c` (option `ExtendedAspectRatio`), `src/main.c` (largeur calculée), `src/zelda_rtl.c` (`ConfigurePpuSideSpace`), `snes/ppu.h` et `snes/ppu.c` (marges), `src/types.h` (constante `kPpuExtraLeftRight`). Exemple de principe, **pas un patch transposable tel quel**.
3. **ROM hack widescreen** : https://github.com/VitorVilela7/wide-snes — code assembleur, conditions spécifiques à SMW et tests de niveaux. Ce projet utilise une ROM patchée et **bsnes-hd**, pas notre exécutable SDL : référence conceptuelle, pas dépendance.
4. **Rendu adaptatif d'une recompilation distincte** : https://github.com/mstan/SuperMarioWorldRecomp et https://github.com/mstan/SuperMarioWorldRecomp/blob/main/docs/adaptive-renderer.md — stratégie de composition côté hôte, Map16, coordonnées signées, traitement des OAM, spawn/despawn, HUD. Projet séparé et expérimental ; comparer les architectures, vérifier les droits avant toute réutilisation.
5. **Émulateur de référence** : https://github.com/elzo-d/LakeSnes — provenance d'une partie des implémentations SNES utilisées par snesrev.
6. **Désassemblage et symboles SMW** : https://github.com/IsoFrieze/SMWDisX — repérer les routines caméra, sprites, niveaux, chargement d'objets.
7. **Issue originale sur le sujet** : https://github.com/snesrev/smw/issues/52 — demande de support widescreen, pas une implémentation.
8. **Documentation des modifications visuelles dans un fork** : https://github.com/danielsoares02/smw — changements à `src/opengl.c` / `src/main.c` à étudier séparément.

## Observations vérifiées dans notre code

- `src/config.h` définit déjà `uint8 extended_aspect_ratio`, mais `src/config.c` ne propose pas de clé INI fonctionnelle pour l'activer.
- `src/main.c` contient :
  ```c
  g_snes_width = (g_config.extended_aspect_ratio * 2 + 256);
  g_snes_height = 224;
  ```
  Cela ne suffit pas : la génération des pixels, les buffers et la logique des entités restent essentiellement conçus pour 256 pixels.
- `src/main.c::RtlDrawPpuFrame` copie explicitement **256 pixels** par ligne :
  ```c
  memcpy((uint8 *)pixel_buffer + y * pitch,
         ppu_pixels + y * 256 * 4,
         256 * 4);
  ```
  Changer uniquement `g_snes_width` produirait des zones non calculées ou incohérentes.
- `src/snes/ppu.h` définit actuellement `kPpuXPixels = 256` et `kPpuExtraLeftRight = 0`. Le rendu des BG, sprites, fenêtres et masques doit être audité avant d'élargir ces valeurs. Vérifier les allocs, le pitch, les coordonnées négatives et chaque chemin de rendu (`ppu.c` et `ppu_old.c`).
- `smw.ini` : `IgnoreAspectRatio = 1` autorise l'étirement de l'image. **Ce n'est pas** le mode recherché.
- `src/opengl.c` et le renderer SDL n'ont pas nécessairement la même gestion des dimensions ; tester les deux.

## Mécanique à analyser (Zelda3 comme référence)

Dans Zelda3, une largeur écran de 256 + 2×marge est accompagnée d'une capacité PPU latérale et d'une logique `ConfigurePpuSideSpace` qui borne les marges selon le monde. Exemple réellement présent dans `zelda3/src/main.c` :

```c
g_zenv.ppu->extraLeftRight =
    UintMin(g_config.extended_aspect_ratio, kPpuExtraLeftRight);
g_snes_width = g_config.extended_aspect_ratio * 2 + 256;
```

**Attention** : l'existence de ces champs dans SMW ne signifie pas que la fonctionnalité y est intégrée. Super Mario World dépend de son propre scrolling, de sa carte Map16, des routines d'apparition et de disparition des ennemis et du placement HUD.

### Largeur et aspect

- Image SNES native courante : 256×224 avec pixels non carrés (présentation approximative 4:3).
- Avec proportions de pixels traditionnelles préservées : un rendu 16:9 à hauteur 224 demande environ **342 colonnes de contenu** ; c'est l'approche décrite par le rendu adaptatif de mstan.
- `wide-snes` propose notamment 352×224 (16:10 sans étirement) et 384×224 (16:9 sans étirement selon son modèle de pixels) : conventions différentes. Ne pas mélanger résolution brute et ratio affiché.
- Le prototype doit expliciter la convention de pixels retenue et éviter toute déformation de l'image utile.

## Méthode de travail progressive

1. **Mesurer le baseline** : reproduire la compilation TCC Windows actuelle, confirmer ratio original, SDL/OpenGL, saves et performances. Ne pas modifier le gameplay.
2. **Inventorier les limites 256** : rechercher `256`, `0xff`, `kPpuXPixels`, `kPpuExtraLeftRight`, `pitch`, fenêtres PPU, sprites, buffers et copier-coller de scanlines. Écrire le schéma des données et leurs propriétaires.
3. **Prototype visuel non destructif** : option de configuration désactivée par défaut, tampon élargi initialisé, mode original identique. Une marge noire n'est **pas** une fonctionnalité widescreen terminée.
4. **Rendre de vraies colonnes supplémentaires** : niveaux/Map16, scrolling et parallax, fenêtres BG, sprites et priorités (y compris objets complexes et Yoshi).
5. **Synchroniser la logique de simulation** : visibilité/activation des ennemis, frontières des niveaux, suppression des entités, coordonnées hors écran, caméra et transitions ; ne pas altérer la physique sans nécessité.
6. **Adapter le HUD** : vies côté gauche, réserve au centre, temps/score à droite ; éviter coupure/double affichage.
7. **Compatibilité** : écrans-titres, carte du monde, niveaux verticaux, boss, Mode 7, fades et savestates peuvent temporairement revenir en vue native, explicitement documentée.
8. **Poursuivre par petits commits** : comparer 4:3 vs 16:9, isoler les régressions, documenter les comportements non couverts.

## Vérifications / critères d'acceptation

- `.\run_with_tcc.bat` compile sur Windows ; aucun asset/ROM n'entre dans Git.
- Avec le mode désactivé : rendu, mouvement, son, contrôles, sauvegardes et fullscreen identiques au baseline.
- Avec le mode activé : davantage de terrain réellement visible, formes et tailles des personnages conservées, pas de fuite mémoire ni lecture hors limites.
- Essais recommandés : Yoshi's Island 1 et 2, niveaux avec scrolling et sprites près des bords, tuyaux, montées/descentes, maisons hantées, boss, niveaux verticaux, Overworld, title, fin de niveau, changements de fenêtre.
- Tester jeu SDL et OpenGL, plein écran et fenêtré, filtre/GLSL, pause et chargement de sauvegardes.
- Consigner résultat réel, branche/commit, capture ou trace, anomalie et procédure de reproduction. Ne pas déclarer « testé » sans l'avoir exécuté.

## Règles de réutilisation

Respecter les licences individuelles et les auteurs ; ne pas intégrer ROM, ressources propriétaires, patchs binaires ou code externe sans vérification des conditions. Toute référence externe doit être un lien vers le dépôt d'origine et une idée vérifiable, pas une garantie de compatibilité.
