---
name: integrated-menu
description: Concevoir, intégrer et tester un menu de réglages directement dans Super Mario World PC lors de la pause, avec clavier, manette et rendu SDL/OpenGL.
---

# Integrated Menu : skill de développement

## Quand l'utiliser

Lire ce skill pour toute tâche concernant un overlay, un écran de pause côté application, les réglages en jeu, la navigation manette/clavier, la gestion du son en pause, les options graphiques ou la persistance de la configuration. En cas d'options 16:9, lire aussi `../smw-widescreen/SKILL.md`.

## Objectif et état

- Intégrer un **menu de l'application PC par-dessus l'image du jeu**, accessible pendant la pause.
- Le menu doit pouvoir être quitté sans modifier ou corrompre l'état SNES, les contrôles ou les sauvegardes.
- Commencer par une expérience simple : `Reprendre`, `Affichage` (4:3/16:9 lorsqu'implémenté), `Plein écran`, `Audio`, `Quitter` (avec confirmation).
- **État à la création** : document de conception ; aucun menu intégré nouveau n'a encore été codé.
- Séparer **pause native du jeu** (bouton SNES Start) et **pause de l'application** (raccourcis PC) ; définir le déclencheur attendu avant de connecter un menu à Start. Un Start SNES ne doit pas ouvrir par surprise un menu système.

## Points d'entrée vérifiés dans le dépôt

- `src/main.c::HandleInput` → `FindCmdForSdlKey` → `HandleCommand` : commandes clavier.
- `src/main.c::HandleGamepadInput` / `HandleGamepadAxisInput` : commandes manette.
- `src/main.c::HandleCommand` :
  ```c
  case kKeys_Pause: g_paused = !g_paused; break;
  case kKeys_PauseDimmed:
    g_paused = !g_paused;
    // ... remplissage SDL sur Windows ...
    break;
  ```
- `smw.ini`, section `[KeyMap]` : `Pause = Shift+p`, `PauseDimmed = p`, `Fullscreen = Alt+Return` ; `[GamepadMap]` pour les manettes. Par défaut P met le jeu en pause avec assombrissement ; ce n'est **pas** une navigation de menu.
- Dans la boucle principale de `src/main.c`, `SDL_PollEvent` traite clavier et manettes, puis :
  ```c
  if (g_paused) {
    SDL_Delay(16);
    continue;
  }
  ```
  Donc pendant la pause, **la simulation et le rendu normal ne tournent plus** ; un menu devra maintenir son propre rafraîchissement sans simuler une frame de jeu.
- `src/main.c::DrawPpuFrameWithPerf` récupère un tampon (`BeginDraw`), y dessine le jeu puis appelle `EndDraw`. C'est un emplacement à étudier pour une composition d'overlay, sans modifier le moteur SNES.
- `src/main.c::SdlRenderer_*` et `src/opengl.c` : deux backends. Le code de pause assombrie utilise directement `g_renderer` sous Windows ; ne pas le réutiliser sans garde car le backend OpenGL peut ne pas avoir de `SDL_Renderer` valide.
- `src/config.c::ParseConfigFile` préfère `smw.user.ini` s'il existe, sinon `smw.ini`. Examiner comment écrire les préférences utilisateur sans toucher au fichier de configuration de référence versionné.

## Références à consulter

1. https://github.com/snesrev/smw/blob/main/src/main.c — boucle SDL, commandes, pause et abstraction du rendu.
2. https://github.com/snesrev/smw/blob/main/src/config.c — lecture INI et mappings.
3. https://github.com/snesrev/smw/blob/main/src/opengl.c — backend OpenGL.
4. https://github.com/snesrev/zelda3 — architecture apparentée et séparation de l'affichage natif.
5. https://github.com/Christopher876/smw — fork ayant expérimenté une interface Nuklear et le scripting Lua ; **prototype de référence**, ne pas supposer une intégration prête à fusionner.
6. https://github.com/Immediate-Mode-UI/Nuklear — interface immédiate C ; évaluer dépendances, taille, licence et intégration avec SDL2.
7. https://wiki.libsdl.org/SDL2/CategoryAPI — événements, manettes, textures et composition SDL2.

## Architecture conseillée

1. Créer un état explicite du menu (`closed/open/settings/confirm-exit`), séparé de l'état du CPU émulé et de la pause SNES.
2. À l'ouverture : figer proprement le temps du jeu et l'audio ; capturer la frame affichée, ou conserver une surface persistante. Le menu continue à recevoir des événements.
3. Intercepter les touches/boutons destinés au menu **avant** qu'ils alimentent `g_input_state`, sauf commandes système explicitement autorisées ; éviter les touches « maintenues » après fermeture.
4. Dessiner un overlay transparent par-dessus la dernière image de jeu **à chaque itération de la pause**, sans appeler `RtlRunFrame`. Implémenter proprement SDL et OpenGL, ou partager une composition pixel/texture indépendante des backends.
5. Utiliser une logique de navigation commune clavier/manette (haut/bas, valider, retour) ; mouse optionnelle plus tard. Gestion d'auto-repeat, focus et déconnexion manette à tester.
6. Séparer **settings en mémoire** et **écriture persistante**. D'abord `Appliquer`/`Annuler` et retour ; ensuite sauvegarde en configuration utilisateur dédiée. Les modifications de ratio peuvent nécessiter redimensionnement des textures/tampons, pas seulement la fenêtre.
7. Garder l'affichage lisible en 4:3 et 16:9, fullscreen, petites tailles et différents backends ; prévoir une police utilisable, sans distribuer d'assets propriétaires.

## Plan par incréments

- M0 : conception et carte des événements/rendu (ce document), aucune modification du jeu.
- M1 : ouvrir/fermer un menu minimal sur la **pause PC** sans bloquer le traitement SDL.
- M2 : dessiner correctement fond assombri + liste `Reprendre / Réglages / Quitter` sous SDL **et** OpenGL.
- M3 : navigation manette, retour et préservation des états d'entrées.
- M4 : options fullscreen, audio, ratio original/16:9 (uniquement lorsque l'implémentation 16:9 existe).
- M5 : persistance, tests de sauvegardes, validation régression.

## Tests obligatoires

- Ouvrir/fermer en jeu, sur écran-titre et map ; pause native Start sans menu involontaire.
- Simuler le menu ne doit pas faire défiler les frames du jeu ; audio suspendu/restauré et pas de touche coincée.
- SDL, SDL-Software, OpenGL ; shaders, taille de fenêtre, plein écran et changement de ratio.
- Manette connectée/déconnectée, navigation clavier AZERTY et manette ; réactivité des contrôles après retour.
- Sauvegarde/chargement, turbo, raccourcis F1–F10 et sortie ; aucune corruption, aucun crash.
- Le jeu sans menu doit rester identique. Les tests sur Windows doivent être exécutés réellement avant d'affirmer qu'ils passent.

## Maintenance du skill

Après chaque modification, noter : fichiers/commits concernés, mécanisme choisi, comportements validés, bugs connus, décisions UI non prises et prochaines étapes. Ne pas confondre documentation planifiée et fonctionnalité implémentée.
