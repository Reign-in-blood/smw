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

## Journal des modifications — 1er prototype

Un premier incrément expérimental a été ajouté sur `feature/widescreen-16-9` : option `ExtendedAspectRatio` désactivée par défaut, PPU élargi à 400 colonnes de capacité avec viewport visible 398x224, rendu latéral Mode 1 (hors mosaïque), repli 256px sur autres modes. **Pas de support achevé de la caméra, VRAM/Map16, spawn des ennemis et HUD.** Compilation Windows et comportement en jeu à valider par l'utilisateur.

Lire [`docs/WIDESCREEN_PROTOTYPE.md`](../../../docs/WIDESCREEN_PROTOTYPE.md) avant les prochains changements.

## Retours des premiers tests Windows (rapport utilisateur, 2 novembre 2026)

- Le jeu **compile et démarre** avec TCC sous Windows ; avertissement non bloquant `M_PI redefined` depuis stb/TCC.
- Le terrain supplémentaire sur les côtés est **effectivement visible en niveau**, en mode fenêtré comme en plein écran.
- Des éléments graphiques supplémentaires remplissent également les marges de certains menus, mais le résultat visuel est incorrect.
- La touche `R` **existait déjà dans le dépôt original** : elle sélectionne alternativement les moteurs PPU. Le moteur historique affiche les bandes noires latérales, le nouveau moteur affiche les pixels supplémentaires. Ce raccourci n'est pas une commande de ratio 4:3/16:9.
- L'utilisateur constate une impression de proportions erronées en plein écran ; diagnostic et correction différés.
- Ne pas considérer ces observations comme des validations générales : les VRAM/Map16, les ennemis, le HUD et les scènes spéciales restent à examiner.
- Dans `[KeyMap]`, le nom de commande reconnu est `ToggleRenderer = r` et **non** `NewRenderer = r`. Dans `[Graphics]`, `NewRenderer = 1` reste un booléen.


## Jalon suivant — activation anticipée des ennemis (2 novembre 2026)

Première implémentation dans `src/smw_02.c` : pour les niveaux **horizontaux classiques**, le parseur original conserve son fonctionnement, puis une analyse supplémentaire du fichier de placements charge les ennemis ordinaires lorsque leur position entre dans la zone étendue de 398 pixels plus **32 pixels de sécurité par côté**. Un garde-fou par index de placement évite de recréer en boucle un ennemi éliminé tant que son emplacement reste dans la zone d'activation ; il est réinitialisé au chargement d'un niveau et lorsqu'un placement quitte la zone.

`src/smw_rtl.c::SmwWidescreenKeepSpriteAlive` élargit la durée de conservation horizontale. Les trois banques de routines `SubOffscreen_Bank01/02/03` utilisent ce test mais continuent leurs propres contrôles de hauteur et propriétés. `GetDrawInfo_Bank01` et `GetDrawInfo_Bank23` n'écartent plus automatiquement les objets dont la position appartient encore à cette plage.

**Restrictions :** générateurs, shooters et sprites spéciaux, niveaux verticaux, patches Lunar Magic, scènes Mode 7 exclus du chargement anticipé. Nombre de slots standard conservé (12). Les coordonnées OAM SNES à 9 bits, les assemblages graphiques, les collisions et les limites de streaming restent sujets à vérification ; la nouvelle logique ne garantit pas que tous les monstres s'affichent parfaitement aux bords. Gardes d'activation non sérialisés dans les savestates, à traiter après test. Aucun test Windows réalisé par l'assistant.

**Essais prioritaires :** Yoshi's Island 1/2 : avancer vers un Koopa puis reculer et revenir ; tuer le Koopa et rester à proximité (pas de respawn immédiat) ; sortir de sa zone puis revenir ; faire défiler un ennemi au-delà du bord droit/gauche en 16:9 et vérifier son retour ; comparer `4:3` ; reprendre une sauvegarde ; capturer les ennemis invisibles / graphiques erronés.

## Correctif de l'échec du comparateur (2 novembre 2026)

Retour utilisateur : à partir de la frame `5216`, le jeu a affiché `Memory compare failed`, `VRAM OAM compare failed`, puis `Verify failure!`. **Cause confirmée dans `src/common_cpu_infra.c` :** le mode `RM_BOTH` exécute la simulation de la ROM originale et la recompilation C, puis compare RAM/OAM. Les spawns anticipés C diffèrent délibérément de la ROM, déclenchant le repli sur l'état original : le nouveau chargement ne persistait pas. Le correctif choisit explicitement `RM_MINE` pour le mode 16:9 expérimental uniquement, avec message dans la console. La comparaison en 4:3 reste intacte ; ne pas masquer les véritables erreurs C dans les deux modes.

Un second correctif relie `g_widescreen_spawn_guard` au parseur de sprites d'origine afin d'éviter des réapparitions doubles lors du passage par sa bande de chargement 256px.

**À valider :** compilation TCC, apparition de la ligne de console `Experimental widescreen: native C gameplay only; ROM/C comparison disabled.`, comparaison 4:3, ennemi qui entre depuis la droite, ennemis éliminés, déplacement caméra aller/retour. L'absence de warnings en mode natif ne garantit pas le succès ; vérifier l'écran et la physique.

## Tests utilisateur et traitement du jalon 3 (2 novembre 2026)

**Retour utilisateur confirmé** sur le correctif précédent : les ennemis ordinaires n'apparaissent toujours pas dans les marges 16:9. En faisant défiler la caméra de droite à gauche puis en sens inverse, un ennemi encore actif peut coexister avec une copie réapparue à son ancien point de placement. **À revalider** après la correction du garde de chargement.

**Autre régression observée, volontairement différée** : dans un petit niveau bonus accessible par un tuyau, la roue de blocs à frapper est placée trop haut pour être atteinte et de nombreux éléments graphiques sont incorrects. Il s'agit d'un bug bloquant pour ce sous-niveau ; conserver une entrée de suivi pour l'analyse des niveaux bonus, transitions, scrolling et mise en page, après correction des ennemis. Ne pas le considérer comme résolu.

**Analyse technique :** `src/snes/ppu.c::ppu_evaluateSprites` décode l'OAM 9 bits `x>255` en `x-512`, donc le terrain étendu à droite n'implique pas que les sprites puissent apparaître à `x=256..327`. Nouveau jalon : table **hôte uniquement** `Ppu::widescreenOamX[128]`, remise à zéro avant chaque frame, complétée par les chemins C `FinishOAMWrite` et les routines génériques 1 et 2 tuiles. Le PPU remplace la coordonnée OAM uniquement si le moteur connaît une position à droite du champ natif. L'OAM original, les savestates SNES et le mode 4:3 restent inchangés.

Le garde de spawn vérifie maintenant les 12 slots réellement actifs et ne réinitialise l'identité d'un placement hors fenêtre que si aucun ennemi issu de ce placement n'est encore vivant. **Limites** : chemins de dessin spéciaux, sprites supplémentaires, limites OAM SNES, état des gardes non sérialisé ; autres anomalies graphiques et niveau bonus toujours non corrigés. Ne pas déclarer « fonctionnel » sans nouveau test TCC Windows.

## Règles de réutilisation

Respecter les licences individuelles et les auteurs ; ne pas intégrer ROM, ressources propriétaires, patchs binaires ou code externe sans vérification des conditions. Toute référence externe doit être un lien vers le dépôt d'origine et une idée vérifiable, pas une garantie de compatibilité.
