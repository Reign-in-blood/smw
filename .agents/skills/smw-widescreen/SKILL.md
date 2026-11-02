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

## Références complémentaires **obligatoires** avant tout nouveau correctif 16:9

**Demande explicite du propriétaire (2 novembre 2026) : lire effectivement ces sources avant de corriger le spawn, les plateformes, l'OAM, le scrolling ou le HUD. Ne pas simplement les citer.**

1. **[RetroPortingToolKit/snesrecomp](https://github.com/RetroPortingToolKit/snesrecomp)**, notamment **[docs/WIDESCREEN_PATTERNS.md](https://github.com/RetroPortingToolKit/snesrecomp/blob/main/docs/WIDESCREEN_PATTERNS.md)**. Checklist de problèmes déjà rencontrés dans des ports SNES. Priorité aux invariants P1 (phase camera PPU), P7/P8 (deux barrières : culling + écriture OAM), P9/P10 (spawn anticipé + règles de progression séparées), P12 (objets de grande taille), P14 (interrupteurs indépendants), P16 (parité 4:3). Les concepts de Mega Man X ne s'appliquent pas à l'identique aux routines SMW : vérifier.
2. **[VitorVilela7/wide-snes](https://github.com/VitorVilela7/wide-snes)**, en particulier **[super-mario-world/asm/level_sprites.asm](https://github.com/VitorVilela7/wide-snes/blob/master/super-mario-world/asm/level_sprites.asm)**, **[level_render.asm](https://github.com/VitorVilela7/wide-snes/blob/master/super-mario-world/asm/level_render.asm)** et **[main.asm](https://github.com/VitorVilela7/wide-snes/blob/master/super-mario-world/asm/main.asm)**. Dans `level_sprites.asm` se trouvent les fenêtres d'apparition, les tableaux `SUB_OFF_SCREEN`, le marquage des positions widescreen, la routine OAM ordinaire **et une correction dédiée `chain_test_for_offscreen` pour les plateformes à chaîne** (région correspondant à `$01C957`). Ce patch nécessite une ROM modifiée et **bsnes-hd** ; nous devons transposer des principes, jamais appliquer le BPS à notre port ou supposer le runtime identique.
3. **[mstan/SuperMarioWorldRecomp](https://github.com/mstan/SuperMarioWorldRecomp)** est une référence **SMW native déjà dotée du widescreen adaptatif**. Lire en priorité **[docs/adaptive-renderer.md](https://github.com/mstan/SuperMarioWorldRecomp/blob/main/docs/adaptive-renderer.md)**, **[src/smw_renderer.c](https://github.com/mstan/SuperMarioWorldRecomp/blob/main/src/smw_renderer.c)**, **[src/smw_renderer_hooks.c](https://github.com/mstan/SuperMarioWorldRecomp/blob/main/src/smw_renderer_hooks.c)** et **[tools/apply_renderer_hooks.py](https://github.com/mstan/SuperMarioWorldRecomp/blob/main/tools/apply_renderer_hooks.py)**. Cette implémentation traite exactement l'activation à distance +32px, les limites de niveau, les sprites qui réutilisent le même OAM, les objets multi-tuiles, le HUD et les snapshots de caméra/VRAM/IRQ.
4. **[snesrev/zelda3](https://github.com/snesrev/zelda3)** : référence générale pour le PPU 9 bits, **pas** suffisante pour les sprites complexes de SMW.

**Architecture déjà éprouvée dans SuperMarioWorldRecomp :** conserver le PPU natif à 256×224 ; construire une surface hôte indépendante et capturer les coordonnées **signées et l'image OAM finale de chaque dessin d'objet** avant qu'un autre dessin ne réutilise son slot. Un index OAM avec une coordonnée « indice » (`widescreenOamX[128]`) ne peut pas identifier sans ambiguïté un élément : une plateforme peut occuper plusieurs slots, plusieurs dessins peuvent réutiliser le même slot, et un X positif 9 bits peut représenter la projection d'un objet situé à gauche. La simple vérification des huit bits de X n'est pas une identité d'objet.

**Règle anti-boucle de débogage :** aucun autre patch combinant spawn, OAM et plateformes sans :
- un scénario de reproduction précis (niveau/sous-niveau, objets, mouvement caméra, commit) et un diagnostic de la **cause** (création, culling, OAM, raster, streaming) ;
- comparaison avec les fichiers **SMW** ci-dessus et la routine C équivalente de ce fork ;
- un changement isolé, désactivable séparément des autres extensions ;
- critères de régression : Koopa marge gauche/droite, plateformes mobiles et leurs collisions, eau BG3, item de réserve, tuyau/bonus, mode 4:3 ; conserver les résultats précédents ;
- distinction nette entre « corrigé dans le code » et « confirmé par test Windows utilisateur ».

Ne pas reprendre une solution d'un autre moteur sans vérifier la licence des fichiers concernés. **Pas de réécriture immédiate du moteur autorisée par ces notes :** d'abord faire une comparaison de conception et préparer des tests reproductibles.

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

## Comparaison ciblée avec Zelda3 : décodage 9 bits des sprites (2 novembre 2026)

Le test utilisateur du commit `1694bb8` confirme que les monstres apparaissent toujours à la limite de l'ancienne fenêtre de 256 pixels. **Différence structurelle retrouvée dans Zelda3** : `snesrev/zelda3/snes/ppu.c::ppu_evaluateSprites` applique la soustraction 512 uniquement si `x >= 256 + extra_left_right`. La version SMW appliquait systématiquement `if (x > 255) x -= 512`, ce qui transforme un sprite situé à x=300 en x=-212, même quand le nouveau PPU affiche le décor à droite. Voir https://github.com/snesrev/zelda3/blob/master/snes/ppu.c#L1239-L1267.

**Correctif ciblé** : `src/snes/ppu.c::ppu_evaluateSprites` teste désormais `x >= 256 + ppu->extraRightCur`. Le mode 4:3 conserve exactement le seuil historique de 256 ; la largeur supplémentaire n'est prise en compte que si elle est autorisée pour cette ligne. Le correctif hôte `widescreenOamX` reste en place. Cela ne prouve **pas** que les ennemis s'afficheront : seule une nouvelle compilation/test Windows le confirmera.

**Autre référence Zelda3 à maintenir** : `src/sprite.c::Sprite_ActivateWhenProximal` et `Sprite_PrepOamCoordOrDoubleRet` élargissent séparément activation, visibilité et conservation avec `kFeatures0_ExtendScreen64`. Le PPU seul ne suffit pas. Les doublons et le niveau bonus à la roue de blocs restent des régressions ouvertes.

## Régressions graphiques graves — BG3 et plateformes (retour utilisateur, 2 novembre 2026)

**Confirmé par test utilisateur du commit `0ccdc1f` :** les mobs ordinaires semblent enfin s'afficher dans le 16:9. **Mais** des niveaux deviennent injouables : eau d'un sous-niveau bonus limitée aux 256 px d'origine en bas d'écran, objets manquants/déplacés ; autre niveau normal avec plateformes mobiles tournant autour d'un axe devenues complètement invisibles alors qu'elles fonctionnaient quelques commits auparavant.

**Constats dans le code :** `src/snes/ppu.c::PpuWindows_Clear/Calc` excluait délibérément la couche BG3 (`layer == 2`) de l'extension sur **toutes** les lignes. Cela protège la barre d'état en haut de l'écran, mais coupe l'eau BG3 dans le bas. Autre problème probable : le préchargeur ajoutait tous les types de sprites `id < 0xC9`, y compris les plateformes `0x5f`/ `0xa3` et boules/chaînes `0x9e`, qui dépendent de slots OAM et de l'ordre d'apparition natif.

**Correctif expérimental :** BG3 peut maintenant couvrir les marges **seulement sous les 32 premières lignes** du HUD ; le rendu doit être revalidé sur les niveaux aquatiques et les sous-niveaux avec eau, car le tilemap BG3 n'a pas été étendu/chargé spécifiquement. Le préchargeur est limité aux Koopas/Goombas/Buzzy ordinaires (`id<=0x0c`, `0x0f`, `0x10`, `0x11`), conserve quatre slots de sprites libres, et se désactive dans les niveaux contenant une plateforme/chaîne complexe (`0x5f`, `0x9e`, `0xa3`). Le PPU augmente le budget de sprites/tuiles sur une ligne en 16:9 seulement, pour ne pas masquer des plateformes multi-tuiles juste parce que le champ est plus large.

**Non vérifié en jeu** : Windows TCC reste à compiler/tester. La disparition des plateformes peut avoir d'autres causes (OAM de sprites spéciaux, état C natif). Régression du bonus : cercle de blocs trop haut et autres objets manquants **encore ouverts** malgré l'essai BG3. Surtout ne pas déclarer ces problèmes résolus avant reproduction.

## Correctif HUD et rétablissement des ennemis dans les niveaux à plateformes (2 novembre 2026)

**Rapport utilisateur sur `0e5cb9f` :** eau BG3 étendue = OK, plateformes tournantes = de nouveau visibles mais avec bugs, **ennemis = retour à l'apparition au bord 256 px**, et **item de réserve HUD (case en haut au centre) se décale hors de la case puis se recentre au fil du scrolling / lors de gros changements de caméra**.

**Erreur trouvée dans le correctif précédent :** `SmwWidescreenLoadPlacedSprites` désactivait TOUS les spawns anticipés dès qu'un `0x5F`, `0x9E` ou `0xA3` était présent dans la liste du niveau, y compris les ennemis de base. Ce garde global a été supprimé. La liste blanche des seuls ennemis classiques, les 4 slots laissés libres, et le chargement natif des mécanismes spéciaux restent en place. Cela préserve le progrès des plateformes tout en réactivant la préapparition des Koopas dans ces niveaux. **À revalider** : certains types d'ennemis toujours hors liste blanche spawneront en 1:1.

**Cause probable du bug HUD :** la table `Ppu::widescreenOamX` reçoit des corrections hôte attachées à l'index OAM ; la SNES réutilise des slots pour des objets graphiques différents (dont l'item de réserve à X=120, Y=15 dans `UpdateStatusBarCounters_DrawItemBoxItem`). Une coordonnée d'un ancien sprite peut alors déplacer un nouveau sprite fixe. Le PPU ne consomme plus un indice hôte si ses 8 bits faibles ne correspondent pas à ceux de **l'OAM courant**. Cela ne modifie pas l'OAM original ni les coordonnées natives, et conserve le décodage étendu de Zelda3.

**Tests Windows à réaliser (aucune compilation par l'assistant) :** niveau avec plateforme tournante + Koopa, eau BG3, item réserve pendant plusieurs passages de frontières de caméra à gauche/droite et sauvegarde/chargement. Noter défauts résiduels des plateformes et de l'eau ; bug du bonus (roue de blocs trop haute) toujours ouvert.

## Test du 2 novembre 2026 après `af68f69` — état de référence, régressions ouvertes

- **Eau BG3 : OK**, rendu plein champ en 16:9 ; ne pas la casser.
- **Objet de réserve dans la case au centre du HUD : OK**, décalage temporaire corrigé ; ne pas le casser.
- **Plateformes mobiles tournantes/chaînes : NON**. Quand la caméra se déplace vers la droite, la plateforme à gauche se représente à droite ; les deux images deviennent figées, puis disparaissent. Les plateformes étaient visibles auparavant mais avec des bugs. Hypothèses à départager : interprétation ambiguë du X OAM 9 bits, réemploi des slots OAM, dessin multi-tuiles, culling propre au sprite `0x5F`. `wide-snes/level_sprites.asm::chain_test_for_offscreen` est une piste directe (le projet considère ce cas **distinct** du culling générique).
- **Ennemis : NON**, apparition toujours calée sur la fenêtre native 256 pixels malgré la marge 16:9. La liste blanche de préchargement dans notre `src/smw_02.c` ne suffit pas : séparer dans un diagnostic **record effectivement chargé**, **sprite actif**, **draw info/culling**, **OAM final**, **pixel PPU visible**. L'apparition en 1:1 peut venir de n'importe quel stade ; ne pas retoucher les seuils au hasard.
- **Bonus de tuyau : régressions toujours ouvertes** (objets erronés/absents, roue de blocs trop haute). Ne pas déclarer la prise en charge des sous-niveaux terminée.

**Décision technique du jalon suivant :** SUSPENDRE les ajustements incrémentaux non mesurés de seuils. Étudier le `src/smw_renderer_hooks.c` et le `src/smw_renderer.c` de SuperMarioWorldRecomp : ce projet conserve les dessins OAM de chaque objet à la frontière des routines, avant leur réutilisation, puis reconstruit un affichage hôte distinct. Comparer point par point avec notre `widescreenOamX[128]`, qui associe à tort une coordonnée à un seul emplacement OAM mutable. Préférer un prototype isolé de capture de **chaque dessin terminé** à une accumulation de correctifs pour chaque ennemi. Préparer des mesures avant toute nouvelle modification. Aucun correctif de gameplay n'a été appliqué pendant cette mise à jour documentaire.

## Correctif SMW ciblé : ambiguïté OAM signée et préchargement d'ennemis (2 novembre 2026)

**Comparaison effectuée avant patch :** `wide-snes/super-mario-world/asm/level_sprites.asm` ajuste **séparément** `chain_test_for_offscreen` (`$01C957`), les dessins OAM ordinaires, les flags d'offscreen et le chargeur de sprites. `snesrecomp/docs/WIDESCREEN_PATTERNS.md` P8 confirme qu'un filtre OAM reste un second verrou. `mstan/SuperMarioWorldRecomp/src/smw_renderer.c` capture des dessins complets avant que l'OAM ne soit réutilisé et utilise un X signé réel ; ce résultat est plus fort qu'un simple 9e bit.

**Bug spécifique prouvé par les équations OAM** : l'étape précédente a modifié `ppu_evaluateSprites` pour interpréter tout X 9 bits dans [256,326] comme positif. Un sprite à **X=-212** a exactement le même code OAM à 9 bits qu'un sprite à **X=+300** ; lorsque la caméra avance, un morceau de la plateforme quittant la gauche devient une copie à droite. Le PPU a retrouvé sa règle SNES `x >= 256 => x -= 512` **par défaut**. La correction pour les objets authentiquement à droite requiert désormais un indice hôte avec **le X signé calculé par la routine C du sprite**, validé sur les 8 bits faibles de l'OAM final. Des indices de X par-tuile sont ajoutés à `Spr05F_BrownChainedPlatform` et aux dessins `Spr0A3_GreyChainedPlatform_02D813/_02D848`, les ailes des Koopas ajoutées à leur routine. Le mécanisme préexistant `PpuWidescreenSetOamX` reste limité aux tuiles réellement dans la marge droite ; le 4:3 garde le comportement SNES.

**Ennemis** : la liste blanche de `SmwWidescreenLoadPlacedSprites` était beaucoup trop restrictive. Elle inclut maintenant davantage d'ennemis normaux (types `0x00..0x16`, sous-plages explicites jusqu'à `0x4f`) tout en excluant les plateformes/générateurs et contrôleurs. Les quatre slots réservés sont maintenus. Instrumentation optionnelle : lancer sous PowerShell `$env:SMW_WS_TRACE='1'; .\\run_with_tcc.bat` ; chaque tentative d'activation avancée imprime `WS spawn ... status=1 blocked=0` (chargeur accepté) ou `status=0 blocked=1` (pas de slot). Les instructions de désactivation sont `Remove-Item Env:SMW_WS_TRACE`. Cette trace est cruciale si des ennemis restent invisibles : une ligne `WS spawn` confirme seulement leur création, pas leur rendu.

**Contrôle complémentaire avant test :** un correctif immédiat a restauré le prédicat 4:3 `t2 < 256` des ailes (important pour la parité 4:3) et retiré du préchargeur les contrôleurs de message, keyhole, boss, power-ups, objets de niveau, etc. L'identification des ennemis est fondée sur la table `kSprStatus08SpriteNormalPtrs` de `src/smw_01.c`. Les ennemis spéciaux non listés restent natifs.

**À tester / non validé Windows** : niveau avec plateformes tournantes (défilement répété droite/gauche pour détecter les copies et la disparition), Koopas et autres ennemis à droite avant le bord natif, item de réserve centré, eau BG3 complète, 4:3, sous-niveau bonus. **Les sprites complexes dont le code n'émet pas d'indice hôte n'apparaîtront pas nécessairement dans les marges.** Cette solution corrige une ambiguïté de rendu, mais ne remplace pas l'architecture de dessins OAM indépendants de SuperMarioWorldRecomp pour tous les objets.

## Jalon 8 : capture OAM par acteur, après le constat de scintillement

Retour utilisateur `743d2a6` : plateformes et gros mobs (Bullet Bills) scintillent par colonnes verticales à l'intérieur du 256px, ailes de Koopas à des endroits absurdes, ennemis toujours majoritairement absents des marges ; amélioration confirmée : plus de plateforme de gauche dupliquée à droite. Trace obtenue `WS spawn frame=78 record=11 id=4F world=1808 camera=1450 status=1 blocked=0` prouve le chargement à X écran 358, pas son dessin.

Nouvelle stratégie d'après `mstan/SuperMarioWorldRecomp` : ne plus déplacer les objets dans `ppu_evaluateSprites` via un indice par OAM mutable. Capturer après chaque `ProcessNormalSprites_HandleSprite` les pièces OAM modifiées (leur image finale, le X signé calculé par l'acteur, tailles), avant le prochain acteur. Rendre ces images **uniquement dans la marge droite** et préserver les 256px centraux par le PPU original. Les plateformes complexes nécessitent des indices de X propres à leurs morceaux. Limite du prototype : la détection de changements ne capture pas toute la logique amont, pas encore les sprites étendus. Mode 4:3, HUD et eau préservés par conception.

Le `SMW_WS_TRACE` existant ne trace que les tentatives réussies/échouées du préchargeur, pas chaque image de sprite ; un unique log est donc normal. Évaluer les captures avec un test Windows ; ne pas affirmer corrigé avant validation.

## Jalon 8 : revue statique et deux bugs certains corrigés

**Priorité haute pour les régressions précédentes :** `PpuWidescreenSetOamX` enregistrait l'index `byte_offset >> 2` dans `widescreenOamX[0..63]`, mais tous les appelants ordinaires utilisent les **slots OAM supérieurs 64..127** via `oam_buf[64]`. L'index correct est `64 + (byte_offset >> 2)`. Ancien comportement : sprites sans indication de X à droite ; fuites potentielles vers les éléments du HUD gérés dans la moitié basse. Deuxième bug certain : `return tilesFound != 0 || ppu_drawCapturedWidescreenSprites(...)` ne lançait pas le moteur de capture si un sprite natif était déjà présent sur la ligne (court-circuit de `||`). Corrigé en appel indépendant suivi de la combinaison des booléens. Exiger une recompilation et test Windows avant de considérer la capture prête.

## Règles de réutilisation

Respecter les licences individuelles et les auteurs ; ne pas intégrer ROM, ressources propriétaires, patchs binaires ou code externe sans vérification des conditions. Toute référence externe doit être un lien vers le dépôt d'origine et une idée vérifiable, pas une garantie de compatibilité.
