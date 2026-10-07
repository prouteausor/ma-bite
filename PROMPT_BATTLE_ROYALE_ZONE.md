# PROMPT — Zone Battle Royale (FiveM)

> Copie tout ce qui suit dans Claude Code, à la racine de ton serveur FiveM.

---

## 1. Ton rôle

Tu es un **développeur FiveM senior** : expert Lua 5.4, CitizenFX, OneSync, architecture client/serveur, optimisation (resmon), sécurité et anti-triche. Tu travailles comme sur un serveur en production :

- tu **analyses l'existant avant de coder** et tu restes cohérent avec ce qui est déjà en place ;
- tu **repenses et améliores** mon cahier des charges quand c'est pertinent, et tu justifies tes choix ;
- ton code est **propre, modulaire, optimisé, sécurisé**, sans code mort, sans `TODO` laissé en plan, sans régression sur les autres ressources.

## 2. Contexte

Tu as accès aux fichiers de mon serveur FiveM. Il y a déjà une **redzone** et une **bluezone**.

Je veux une nouvelle zone : une **zone Battle Royale** en format réduit (**20 à 30 joueurs**) dont le périmètre **rétrécit par phases**, visible en jeu.

C'est la **première brique d'un mode de jeu complet** : plus tard j'ajouterai d'autres fonctionnalités (loot, kits, récompenses, classement, spectateur…). L'architecture doit donc être pensée pour être **étendue sans réécrire le cœur**.

## 3. Étape 0 — Analyse obligatoire, puis plan (avant d'écrire du code)

1. Trouve les ressources **redzone** et **bluezone** et lis-les entièrement. Identifie :
   - le framework (ESX / QBCore / Qbox / standalone) et sa version, les libs utilisées (ox_lib, PolyZone…) ;
   - le système de notifications / HUD / textUI, les conventions de nommage (ressources, events, fichiers), la structure des configs ;
   - comment elles détectent l'entrée/sortie, comment elles affichent la zone, comment elles gèrent la mort et le respawn.
2. Vérifie : OneSync activé ? `lua54` ? Quelle ressource gère la mort / l'ambulance ? Quel système de permissions admin (ACE, groupes du framework) ?
3. Présente-moi un **rapport court** + un **plan d'architecture** (arborescence, machine à états, flux des events). **Attends ma validation avant d'implémenter.** Si un point est vraiment bloquant, pose la question ; sinon prends la décision la plus pro et signale-la.
4. **Ne modifie pas** la redzone ni la bluezone (si c'est indispensable, explique pourquoi avant). Réutilise leurs conventions et leur système de notifications pour que tout soit cohérent visuellement. Vérifie qu'il n'y a aucun conflit (noms d'events, zones qui se chevauchent).

## 4. Fonctionnement attendu (cycle d'une partie)

Le **serveur est la seule source de vérité**. La zone suit une machine à états :

| État        | Couleur du mur        | Zone                 | Entrer                        | Sortir (pour un inscrit) |
|-------------|-----------------------|----------------------|-------------------------------|--------------------------|
| `WAITING`   | Orange                | Fixe (rayon initial) | Ouvert tant que < max joueurs | Impossible               |
| `COUNTDOWN` | Orange (léger pulse)  | Fixe                 | Ouvert tant que < max joueurs | Impossible               |
| `RUNNING`   | **Rouge**             | **Rétrécit**         | **Fermé**                     | Impossible               |
| `ENDED`     | —                     | —                    | Fermé                         | Libéré                   |

Règles précises :

1. **Inscription** : un joueur qui entre physiquement dans la zone (à pied ou en véhicule) pendant `WAITING` ou `COUNTDOWN` est inscrit automatiquement. Le serveur valide sa position (jamais le client seul). **Dès qu'il est inscrit, il ne peut plus sortir.**
2. **Minimum atteint** (ex. 20) → passage en `COUNTDOWN` : **décompte de 60 s** (configurable), affiché à tous les inscrits, avec bip sonore sur les 10 dernières secondes. **La zone reste orange et ne rétrécit pas.**
3. Pendant le décompte, d'autres joueurs peuvent encore entrer **jusqu'au maximum** (ex. 30).
4. **Maximum atteint** → la zone est **fermée immédiatement** : plus personne ne peut entrer (repoussé + message « Zone complète »). Le décompte continue (option : le raccourcir, voir config).
5. Si le nombre d'inscrits **repasse sous le minimum** pendant le décompte (déconnexion, mort…) → décompte annulé, retour en `WAITING` + notification.
6. **Fin du décompte** → `RUNNING` : la zone est **verrouillée** (même si le max n'est pas atteint), le mur **passe au rouge**, les phases de rétrécissement commencent.
7. **Fin de partie** : il reste 1 joueur en vie → vainqueur annoncé à tous, passage en `ENDED`, puis **reset automatique** après X secondes vers `WAITING`. S'il ne reste personne → reset direct.

## 5. La zone qui rétrécit

### Deux limites distinctes (choix de design)

1. **Bordure de l'arène** (rayon initial, fixe) : mur **infranchissable**. Un inscrit ne peut pas sortir ; un non-inscrit ne peut pas entrer quand la zone est fermée.
2. **Zone sûre** (rouge, qui rétrécit) : être à l'extérieur de la zone sûre (donc entre elle et la bordure) inflige des **dégâts croissants selon la phase**. C'est ce qui force les joueurs à se rapprocher et à se battre.

Ajoute une option `Config.ShrinkMode` :
- `'damage'` (par défaut, standard Battle Royale) : fonctionnement décrit ci-dessus ;
- `'wall'` : le mur qui rétrécit est lui-même la barrière physique et pousse les joueurs vers l'intérieur (pas de dégâts).

### Phases

Les phases sont définies dans un tableau de config. Pour chaque phase :
- `wait` : pause (s) avant le rétrécissement — la **prochaine zone** est déjà affichée pendant ce temps ;
- `duration` **ou** `speed` (m/s) : durée ou vitesse du rétrécissement (le script calcule l'autre) ;
- `radius` : rayon cible à la fin de la phase ;
- `damage` : dégâts par seconde hors de la zone sûre ;
- `moveCenter` : si `true`, le nouveau centre est tiré au hasard **de façon à ce que le nouveau cercle soit entièrement contenu dans l'ancien** (comme dans les vrais BR), et le centre glisse en douceur pendant le rétrécissement.

La dernière phase doit pouvoir descendre à `0` pour garantir une fin de partie.

### Rendu en jeu (point très important)

Je veux **voir la zone rétrécir de mes propres yeux**, de façon fluide et propre :

- **Mur cylindrique** visible de loin : hauteur configurable, légère transparence, effet de bandes ou de dégradé. Choisis la technique la plus propre et performante (par ex. segments en `DrawPoly` double face avec un nombre de segments configurable, en ne dessinant que les segments proches/visibles) et **justifie ton choix** (un `DrawMarker` géant pose souvent des problèmes de rendu à grande échelle).
- **Rétrécissement fluide image par image** : le serveur n'envoie que les paramètres de la phase (centre/rayon de départ et cible, durée, temps déjà écoulé) ; le client **interpole localement** à chaque frame. Aucun envoi réseau en boucle.
- **Couleurs configurables** : orange (attente), orange pulsé (décompte), rouge (partie), blanc/contour pour la prochaine zone.
- **Carte / minimap** : cercle (`AddBlipForRadius`) mis à jour pendant le rétrécissement, cercle de la prochaine zone, blip au centre.
- **Effets** : filtre rouge + son quand on est hors de la zone sûre, bips du décompte, annonces de phase (« La zone se rétrécit dans 30 s »).
- **HUD propre et discret** (dans le style du serveur) : état, joueurs `X/30`, décompte, phase `N/M`, temps avant le prochain rétrécissement, joueurs en vie, distance à la zone sûre.

## 6. Barrière « impossible de sortir / impossible d'entrer »

- **Côté client** : à l'approche de la bordure, repousse le joueur vers l'intérieur. S'il la dépasse quand même (vitesse, véhicule, ragdoll), replace-le juste à l'intérieur, **au sol (bon Z)**. Gère les piétons et les véhicules (conducteur + passagers, voitures rapides, motos, véhicules aériens).
- **Non-inscrit quand la zone est fermée** : même logique, mais repoussé vers l'extérieur, avec un message clair (« Partie en cours » / « Zone complète »).
- **Côté serveur (anti-triche, OneSync)** : vérification périodique (1–2 s) de la position des inscrits. Un inscrit trop loin hors de l'arène → ramené à l'intérieur + log. Un non-inscrit dans l'arène pendant `RUNNING` → expulsé.
- **Bypass admin** via permission, pour pouvoir observer une partie.

## 7. Cas limites à gérer

- **Déconnexion / crash** pendant chaque état : retrait propre, compteur recalculé, annulation du décompte si besoin, vérification de la condition de victoire.
- **Mort = éliminé définitivement** pour la partie (même s'il est réanimé) : la barrière ne le retient plus et il ne peut pas revenir. Compatible avec le système de mort/ambulance du serveur. Prévois un point d'entrée pour un futur mode spectateur.
- **Joueur qui se connecte en cours de partie** : il reçoit l'état actuel et voit la zone correctement.
- **Restart de la ressource** : nettoyage complet (blips, threads, HUD, effets) et état propre.
- **Plusieurs joueurs qui entrent en même temps alors qu'il reste 1 place** : le serveur tranche, les refusés sont repoussés dehors.
- Téléportation, noclip, véhicules à grande vitesse, chevauchement avec redzone/bluezone.

## 8. Architecture et qualité du code

Arborescence indicative (adapte les noms aux conventions du serveur) :

```
<nom_ressource>/
├── fxmanifest.lua
├── config.lua
├── locales/fr.lua
├── shared/utils.lua       -- maths du cercle, interpolation
├── server/main.lua        -- machine à états, inscriptions, phases
├── server/commands.lua
├── client/main.lua        -- synchro de l'état, détection
├── client/render.lua      -- mur, blips
├── client/barrier.lua
├── client/hud.lua         (+ html/ si NUI)
└── README.md
```

Exigences :

- **Serveur autoritaire** : events validés (source, état, distance), noms préfixés, aucun event exploitable par un client.
- **Synchronisation** via GlobalState / state bags ou events ciblés, jamais de boucle réseau par frame. Ne compare jamais l'horloge du client à celle du serveur : transmets le temps écoulé et recalcule localement avec `GetGameTimer()`.
- **Performance** : threads adaptatifs (`Wait(1000)` ou plus loin de la zone, `Wait(0)` uniquement quand il faut dessiner). Objectif resmon : < 0.05 ms hors zone, le plus bas possible en zone (vise < 0.3 ms).
- Lua 5.4, variables `local`, pas de globales inutiles, logs uniquement si `Config.Debug`.
- **Validation de la config au démarrage** (min ≤ max, rayons décroissants, valeurs positives…) avec messages d'erreur clairs.
- Encapsule la logique d'une arène dans un objet pour pouvoir gérer **plusieurs arènes plus tard** (n'en implémente qu'une pour l'instant).
- **Extensibilité** — exports et events publics documentés, pour que je branche loot / kits / récompenses sans toucher au cœur :
  - exports : `GetGameState()`, `IsPlayerInGame(src)`, `GetAlivePlayers()`, `ForceStart()`, `StopGame()` ;
  - events serveur : `playerJoined`, `countdownStarted`, `countdownCancelled`, `gameStarted`, `phaseChanged`, `playerEliminated (src, killer)`, `gameEnded (winner)`.
- Tous les textes en jeu en **français**, dans le fichier de locale.

## 9. Configuration attendue (exemple)

```lua
Config = {}

Config.Debug = false

Config.Zone = {
    center = vector3(0.0, 0.0, 0.0), -- à placer en jeu avec /br_setcenter
    radius = 250.0,                  -- rayon initial (m)
    wallHeight = 150.0,
}

Config.Players = { min = 20, max = 30 }

Config.Countdown = {
    duration = 60,  -- secondes avant le lancement une fois le minimum atteint
    whenFull = 10,  -- si la zone est pleine, raccourcit le décompte à 10 s (nil = désactivé)
}

Config.ShrinkMode = 'damage' -- 'damage' | 'wall'

-- duration (s) OU speed (m/s) pour chaque phase
Config.Phases = {
    { wait = 60, duration = 90, radius = 175.0, damage = 1,  moveCenter = true  },
    { wait = 45, duration = 60, radius = 110.0, damage = 2,  moveCenter = true  },
    { wait = 30, duration = 45, radius = 60.0,  damage = 4,  moveCenter = true  },
    { wait = 20, duration = 30, radius = 25.0,  damage = 7,  moveCenter = false },
    { wait = 15, duration = 20, radius = 0.0,   damage = 10, moveCenter = false },
}

Config.Colors = {
    waiting   = { r = 255, g = 140, b = 0,   a = 90  },
    countdown = { r = 255, g = 140, b = 0,   a = 120 },
    running   = { r = 220, g = 20,  b = 20,  a = 110 },
    nextZone  = { r = 255, g = 255, b = 255, a = 60  },
}

Config.EndResetDelay = 15         -- secondes avant de rouvrir la zone après une partie
Config.AdminPermission = 'brzone.admin'
```

Chaque option doit être commentée en français dans le fichier final.

## 10. Commandes admin (protégées par permission)

- `/br_setcenter` : place le centre de la zone sur ma position (appliqué immédiatement et sauvegardé, ou coordonnées affichées pour les copier dans la config) ;
- `/br_radius <m>` : change le rayon initial ;
- `/br_start` : force le lancement (ignore le minimum) ;
- `/br_stop` et `/br_reset` : arrête / réinitialise la partie ;
- `/br_debug` : affiche l'état, la phase, le rayon, les joueurs, les timers.

## 11. Tests

Je ne peux pas réunir 20 joueurs pour tester : prévois un **mode test** (minimum 1, maximum 2, phases courtes) pour valider tout le cycle seul ou à deux.

Fournis une **checklist de tests manuels**, au minimum :

1. J'entre dans la zone → je suis inscrit, le compteur se met à jour.
2. J'essaie de sortir à pied, en voiture, en moto rapide → je suis bloqué.
3. Minimum atteint → décompte ; un joueur se déconnecte → décompte annulé.
4. Maximum atteint → un non-inscrit ne peut pas entrer.
5. Lancement → mur rouge, rétrécissement fluide, carte à jour.
6. Hors de la zone sûre → dégâts + effets.
7. Mort → éliminé, libéré, impossible de revenir.
8. Dernier survivant → victoire annoncée, reset.
9. Restart de la ressource en pleine partie → tout est propre.
10. Un joueur se connecte en cours de partie → il voit la zone correctement.

Relis ton code avant de me le rendre (syntaxe, nil checks, fuites de threads, events non protégés).

## 12. Livrables

- La ressource complète, prête à être ajoutée avec `ensure` dans le `server.cfg`.
- Un **README en français** : installation, dépendances, explication de chaque option de config, commandes, exports/events, comment brancher de nouvelles fonctionnalités.
- Un **résumé final** : ce qui a été fait, les choix techniques et pourquoi, les limites connues, et des idées pour la suite (loot, kits, spectateur, classement, routing bucket dédié, multi-arènes…).
