# PROMPT — Mode Battle Royale (zone, squads, KO, réanimation, portage, loot) — FiveM

> Copie tout ce qui suit dans Claude Code, à la racine de ton serveur FiveM.

---

## 1. Ton rôle

Tu es un **développeur FiveM senior** : expert Lua 5.4, CitizenFX, OneSync, architecture client/serveur, synchronisation réseau et optimisation (resmon). Tu travailles comme sur un serveur en production :

- tu **analyses l'existant avant de coder** et tu restes cohérent avec ce qui est déjà en place ;
- tu **repenses et améliores** mon cahier des charges quand c'est pertinent, et tu justifies tes choix ;
- ton code est **propre, modulaire, optimisé**, sans code mort, sans `TODO` laissé en plan, sans régression sur les autres ressources.

## 2. Contexte

Tu as accès aux fichiers de mon serveur FiveM. Il y a déjà une **redzone** et une **bluezone**.

Je veux créer un **mode Battle Royale** en format réduit (**20 à 30 joueurs**) qui se joue dans une zone de la map. Il comprend :

- une **zone qui rétrécit par phases**, visible en jeu ;
- des **squads** (équipes) ;
- un système de **mise au sol (KO)** à la place de la mort, avec **réanimation par un membre de sa squad** ;
- un système pour **porter** un joueur au sol (n'importe quel joueur peut porter) ;
- un système de **loot** : un ennemi peut récupérer le stuff d'un joueur au sol ;
- un **retour au spawn** possible (mais pas obligatoire) 30 s après être tombé.

C'est la **première brique d'un mode de jeu complet** : plus tard j'ajouterai d'autres fonctionnalités (kits, récompenses, classement, spectateur, achèvement, anti-cheat…). L'architecture doit donc être pensée pour être **étendue sans réécrire le cœur**.

## 3. Étape 0 — Analyse obligatoire, puis plan (avant d'écrire du code)

1. Trouve les ressources **redzone** et **bluezone** et lis-les entièrement. Identifie :
   - le framework (ESX / QBCore / Qbox / standalone) et sa version, les libs utilisées (ox_lib, PolyZone…) ;
   - le système de notifications / HUD / textUI / progress bar, les conventions de nommage (ressources, events, fichiers), la structure des configs ;
   - comment elles détectent l'entrée/sortie, comment elles affichent la zone, comment elles gèrent la mort et le respawn.
2. Identifie aussi les systèmes avec lesquels le mode doit cohabiter :
   - le **système de mort / ambulance** (esx_ambulancejob, qb-ambulancejob, wasabi_ambulance…) et s'il a déjà un état « last stand » / KO ;
   - l'**inventaire** (ox_inventory, qb-inventory, qs-inventory…) et s'il sait déjà ouvrir / fouiller l'inventaire d'un autre joueur ;
   - un éventuel système de **target** (ox_target, qb-target), de **portage**, ou de **groupes / équipes** déjà présent ;
   - le **point de spawn** du serveur.
3. Vérifie : OneSync activé ? `lua54` ? Quel système de permissions admin (ACE, groupes du framework) ?
4. Présente-moi un **rapport court** + un **plan d'architecture** (arborescence, machines à états, flux des events). **Attends ma validation avant d'implémenter.** Si un point est vraiment bloquant, pose la question ; sinon prends la décision la plus pro et signale-la.
5. **Ne modifie pas** la redzone, la bluezone, le système de mort ni l'inventaire. Si c'est indispensable (par exemple pour que le système de mort ignore les joueurs en partie), fais-le de façon minimale et réversible, et explique-le avant. Réutilise leurs conventions et leur système de notifications pour que tout soit cohérent visuellement. Vérifie qu'il n'y a aucun conflit (noms d'events, zones qui se chevauchent).

## 4. Fonctionnement attendu (cycle d'une partie)

Le **serveur gère l'état de la partie** : c'est lui qui décide qui est inscrit, dans quelle squad, debout, au sol ou éliminé, et qui peut réanimer, porter ou looter. Le client affiche et envoie les actions du joueur.

La zone suit une machine à états :

| État        | Couleur du mur       | Zone                 | Entrer                        | Sortir (inscrit) | Dégâts entre joueurs inscrits        |
|-------------|----------------------|----------------------|-------------------------------|------------------|--------------------------------------|
| `WAITING`   | Orange               | Fixe (rayon initial) | Ouvert tant que < max joueurs | Impossible       | Désactivés                           |
| `COUNTDOWN` | Orange (léger pulse) | Fixe                 | Ouvert tant que < max joueurs | Impossible       | Désactivés                           |
| `RUNNING`   | **Rouge**            | **Rétrécit**         | **Fermé**                     | Impossible       | Activés (sauf entre coéquipiers)     |
| `ENDED`     | —                    | —                    | Fermé                         | Libéré           | Désactivés                           |

Règles précises :

1. **Inscription** : un joueur qui entre physiquement dans la zone (à pied ou en véhicule) pendant `WAITING` ou `COUNTDOWN` est inscrit automatiquement (le serveur confirme sa position). **Dès qu'il est inscrit, il ne peut plus sortir.** Le minimum et le maximum comptent des **joueurs**, pas des squads.
2. **Minimum atteint** (ex. 20) → passage en `COUNTDOWN` : **décompte de 60 s** (configurable), affiché à tous les inscrits, avec bip sonore sur les 10 dernières secondes. **La zone reste orange et ne rétrécit pas.**
3. Pendant le décompte, d'autres joueurs peuvent encore entrer **jusqu'au maximum** (ex. 30).
4. **Maximum atteint** → la zone est **fermée immédiatement** : plus personne ne peut entrer (repoussé + message « Zone complète »). Le décompte continue (option : le raccourcir, voir config).
5. Si le nombre d'inscrits **repasse sous le minimum** pendant le décompte (déconnexion…) → décompte annulé, retour en `WAITING` + notification.
6. **Fin du décompte** → `RUNNING` : la zone est **verrouillée** (même si le max n'est pas atteint), le mur **passe au rouge**, les **squads sont figées**, les phases de rétrécissement commencent.
7. **Fin de partie** : quand il ne reste **qu'une seule squad avec au moins un membre debout** → victoire de cette squad, annoncée à tous. Si les dernières squads tombent au même moment (ex. dégâts de zone), la squad dont le dernier membre est tombé en dernier gagne. Passage en `ENDED`, puis **reset automatique** après X secondes : les joueurs encore au sol sont réanimés et renvoyés au spawn, les vainqueurs sont libérés, la zone repasse en `WAITING`.

## 5. La zone qui rétrécit

### Deux limites distinctes (choix de design)

1. **Bordure de l'arène** (rayon initial, fixe) : mur **infranchissable**. Un inscrit ne peut pas sortir ; un non-inscrit ne peut pas entrer quand la zone est fermée.
2. **Zone sûre** (rouge, qui rétrécit) : un joueur debout à l'extérieur de la zone sûre (donc entre elle et la bordure) subit des **dégâts croissants selon la phase**. C'est ce qui force les joueurs à se rapprocher et à se battre. Si ces dégâts font tomber ses PV à 0, il est **mis au sol** (voir section 8), comme avec une balle.

Ajoute une option `Config.ShrinkMode` :
- `'damage'` (par défaut, standard Battle Royale) : fonctionnement décrit ci-dessus ;
- `'wall'` : le mur qui rétrécit est lui-même la barrière physique et pousse les joueurs vers l'intérieur (pas de dégâts).

Les **joueurs au sol ne subissent pas les dégâts de zone** (ils ne peuvent pas mourir).

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
- **HUD propre et discret** (dans le style du serveur) : état, joueurs `X/30`, décompte, phase `N/M`, temps avant le prochain rétrécissement, joueurs et squads en vie, distance à la zone sûre, état de ma squad.

## 6. Barrière « impossible de sortir / impossible d'entrer »

C'est une mécanique de jeu côté client :

- À l'approche de la bordure, le joueur inscrit est **repoussé vers l'intérieur**. S'il la dépasse quand même (vitesse, véhicule, ragdoll), il est replacé juste à l'intérieur, **au sol (bon Z)**. Gère les piétons et les véhicules (conducteur + passagers, voitures rapides, motos, véhicules aériens). Un joueur porté suit son porteur.
- **Non-inscrit quand la zone est fermée** : même logique, mais repoussé vers l'extérieur, avec un message clair (« Partie en cours » / « Zone complète »).
- Un joueur **éliminé** (squad éliminée ou retour au spawn) n'est plus retenu par la barrière.
- **Bypass admin** via permission, pour pouvoir observer une partie.

## 7. Squads

- **Taille max configurable** (`Config.Squads.maxSize`, ex. 4 ; `1` = mode solo). Un joueur sans squad joue seul (squad d'un seul joueur).
- **Gestion avant le lancement** (dehors, ou dans la zone pendant `WAITING` / `COUNTDOWN`) : créer une squad, inviter un joueur par son ID, accepter / refuser, quitter, exclure (chef de squad). Via commandes + menu propre (ox_lib context ou équivalent du serveur).
- **Au lancement, les squads sont figées** : les membres qui ne sont pas dans la zone sont retirés de la squad.
- S'il ne reste pas assez de places pour toute une squad, les premiers entrés sont inscrits et les suivants sont refusés avec un message clair.
- **Pas de dégâts entre coéquipiers** (configurable).
- **Coéquipiers visibles** : blip sur la minimap + marqueur discret au-dessus de la tête, et un HUD squad avec pour chaque membre : nom, état (debout / au sol / éliminé), vie.
- **Squad éliminée** : quand tous ses membres sont au sol ou éliminés. Tous ses membres sont alors éliminés de la partie (plus aucune réanimation possible), mais ceux qui sont au sol **restent au sol, lootables**, et gardent la possibilité de retourner au spawn.

## 8. Mise au sol (KO) au lieu de la mort

Pendant `RUNNING`, un joueur inscrit **ne meurt jamais** : quand ses PV tombent à 0 (balle, arme blanche, explosion, chute, véhicule, zone…), il est **mis au sol (KO)**.

- **Animation au sol** propre (blessé), synchronisée pour tous les joueurs. Option pour ramper lentement (`Config.Downed.canCrawl`).
- **Au sol** : pas d'arme en main, pas de tir, pas d'accès à son inventaire, pas de véhicule, pas de sprint.
- **Le système de mort / ambulance du serveur est ignoré** pour les joueurs en partie : pas d'écran de mort, pas de respawn à l'hôpital, pas d'appel EMS. Hors de la partie, tout redevient normal.
- **Invulnérable au sol** : personne ne peut l'achever pour l'instant. Prévois un point d'accroche propre pour ajouter un achèvement plus tard.
- Enregistre **qui l'a mis au sol** (attaquant, arme, ou « zone ») → event `playerDowned`, pour un futur kill feed / des stats.
- **HUD au sol** : « Vous êtes au sol », état de la squad, temps restant avant que le retour au spawn soit disponible, et qui est en train de te réanimer / porter / looter.
- **Technique** : choisis l'approche la plus fiable pour intercepter la mort (par ex. réanimer instantanément le joueur au moment du décès puis le passer en état KO, ou empêcher les PV de descendre sous un seuil) et la plus compatible avec le système de mort existant. **Justifie ton choix.**

### Réanimation

- **Seul un membre de sa squad**, debout et proche, peut le réanimer.
- **Action maintenue** (touche ou target) avec barre de progression (durée configurable, ex. 8 s) et animation (accroupi, soins).
- Annulée si le réanimateur relâche, s'éloigne ou est mis au sol. Une seule réanimation à la fois sur un même joueur.
- Impossible tant que le joueur au sol est porté (il faut le poser d'abord).
- Le joueur réanimé se relève avec un pourcentage de PV configurable, retrouve tous ses mouvements, reste dans la partie et dans la zone.

## 9. Porter un joueur au sol

- **N'importe quel joueur debout de la partie** peut porter un joueur au sol, **coéquipier ou ennemi**.
- Prise / dépose avec une touche ou le target. **Animation de portage synchronisée** des deux côtés (type « fireman carry »), le joueur porté est attaché au porteur.
- Le porteur : vitesse réduite, pas de sprint, pas de tir, pas de véhicule (tout est configurable).
- **Dépose automatique** si : le porteur est mis au sol, se déconnecte ou tente d'entrer dans un véhicule ; le joueur porté retourne au spawn ou se déconnecte ; la partie se termine.
- Un joueur ne porte qu'une seule personne, et un joueur au sol n'est porté que par une seule personne à la fois.
- Le portage respecte la barrière (impossible de sortir de l'arène en portant quelqu'un).
- Pas de loot ni de réanimation pendant le portage.

## 10. Loot

- Quand un joueur est au sol, un **ennemi debout** qui se place **juste à côté de lui** (distance configurable, ex. 2 m) peut le **looter** : il ouvre l'inventaire du joueur au sol et récupère ce qu'il veut, objet par objet **ou tout d'un coup** (bouton « Tout prendre »).
- **Tout ce que le joueur a sur lui dans son inventaire** est lootable : objets, armes, munitions (et argent liquide si activé en config).
- **Utilise le système d'inventaire du serveur** : s'il sait déjà ouvrir l'inventaire d'un autre joueur (fouille), utilise cette fonction ; sinon crée une interface de loot propre, avec le transfert des objets fait par le serveur.
- Le serveur vérifie que l'action est possible (le joueur est bien au sol, personne d'autre n'est déjà en train de le looter) avant de transférer quoi que ce soit.
- Respecte les limites de poids / slots du looter : ce qui ne rentre pas reste sur le joueur au sol.
- **Liste noire** configurable d'objets non lootables (ex. carte d'identité, téléphone, clés).
- Option pour autoriser les coéquipiers à se looter entre eux (`Config.Loot.allowSquadmates`, désactivée par défaut).
- **Un seul looter à la fois** par joueur au sol. L'inventaire se ferme si le looter s'éloigne ou est mis au sol, ou si le joueur au sol est réanimé, porté ou retourne au spawn.
- Le joueur au sol voit qu'il est en train d'être looté.

## 11. Retour au spawn

- **30 s après être tombé** (configurable), le joueur au sol voit apparaître l'option **« Retourner au spawn »** (touche à maintenir quelques secondes pour éviter les erreurs).
- **Ce n'est pas obligatoire** : il peut rester au sol pour attendre d'être réanimé par sa squad, aussi longtemps que la partie dure.
- S'il l'utilise : il **abandonne la partie** (éliminé, plus réanimable, ne peut plus revenir dans la zone pendant cette partie), il est téléporté au spawn (point configurable, par défaut le spawn du serveur) et soigné. Il garde ce qu'il lui reste dans son inventaire.
- Option `Config.Respawn.dropLootOnLeave` (désactivée par défaut) : s'il part avant d'avoir été looté, son stuff reste sur place dans un sac lootable.
- Sa squad continue sans lui si elle a encore des membres debout.

## 12. Cas limites à gérer

- **Avant le lancement** (`WAITING` / `COUNTDOWN`), le système de KO n'est pas actif : si un inscrit meurt quand même (chute, PNJ…), il est géré par le système de mort normal du serveur et retiré de la liste des inscrits.
- **Déconnexion** : debout → retiré, compteurs recalculés, condition de victoire vérifiée ; au sol → traité comme un retour au spawn ; en train de porter / réanimer / looter → action annulée proprement pour l'autre joueur.
- **Joueur qui se connecte en cours de partie** : il n'est pas participant, ne peut pas entrer, mais voit la zone correctement.
- **Mis au sol dans un véhicule** → éjecté et posé à côté. **Dans l'eau** → maintenu à la surface ou posé sur la rive la plus proche (choisis le plus fiable).
- Un joueur au sol réanimé hors de la zone sûre reprend les dégâts de zone.
- **Actions simultanées** (deux joueurs veulent porter, looter ou réanimer la même personne ; plusieurs joueurs entrent alors qu'il reste 1 place) : le serveur attribue au premier, les autres reçoivent un message.
- **Restart de la ressource** en pleine partie : tous les joueurs au sol sont réanimés, les portages détachés, les inventaires de loot fermés, blips / threads / HUD / effets nettoyés, état propre.
- Chevauchement avec la redzone / bluezone.

## 13. Architecture et qualité du code

Arborescence indicative (adapte les noms aux conventions du serveur) :

```
<nom_ressource>/
├── fxmanifest.lua
├── config.lua
├── locales/fr.lua
├── bridge/                -- adaptateurs framework / inventaire / mort / notifs / target
├── shared/utils.lua       -- maths du cercle, interpolation
├── server/main.lua        -- machine à états, inscriptions, phases, fin de partie
├── server/squads.lua
├── server/downed.lua      -- KO, réanimation, retour au spawn
├── server/carry.lua
├── server/loot.lua
├── server/commands.lua
├── client/main.lua        -- synchro de l'état, détection
├── client/render.lua      -- mur, blips
├── client/barrier.lua
├── client/downed.lua
├── client/carry.lua
├── client/loot.lua
├── client/hud.lua         (+ html/ si NUI)
└── README.md
```

Exigences :

- **Pas d'anti-cheat** : ne code **aucun** système anti-triche (détection de téléportation ou de noclip, vérifications périodiques de position côté serveur, bannissements, logs de triche…). Je le ferai plus tard. Prévois seulement des events propres pour pouvoir le brancher ensuite.
- **Synchronisation** via GlobalState / state bags ou events ciblés, jamais de boucle réseau par frame. Ne compare jamais l'horloge du client à celle du serveur : transmets le temps écoulé et recalcule localement avec `GetGameTimer()`.
- **Performance** : threads adaptatifs (`Wait(1000)` ou plus loin de la zone, `Wait(0)` uniquement quand il faut dessiner ou gérer une interaction). Objectif resmon : < 0.05 ms hors zone, le plus bas possible en zone (vise < 0.3 ms).
- Lua 5.4, variables `local`, pas de globales inutiles, logs uniquement si `Config.Debug`.
- **Validation de la config au démarrage** (min ≤ max, rayons décroissants, valeurs positives…) avec messages d'erreur clairs.
- **Bridge** : n'écris que les adaptateurs des systèmes réellement utilisés par mon serveur, mais structure-les pour pouvoir en ajouter d'autres facilement.
- Encapsule la logique d'une arène dans un objet pour pouvoir gérer **plusieurs arènes plus tard** (n'en implémente qu'une pour l'instant).
- **Touches** déclarées avec `RegisterKeyMapping` (modifiables par le joueur dans les paramètres GTA).
- **Extensibilité** — exports et events publics documentés, pour que je branche kits / récompenses / classement / anti-cheat sans toucher au cœur :
  - exports : `GetGameState()`, `IsPlayerInGame(src)`, `GetPlayerSquad(src)`, `IsPlayerDowned(src)`, `GetAlivePlayers()`, `RevivePlayer(src)`, `ForceStart()`, `StopGame()` ;
  - events serveur : `playerJoined`, `countdownStarted`, `countdownCancelled`, `gameStarted`, `phaseChanged`, `playerDowned (src, attacker)`, `playerRevived (src, reviver)`, `playerLooted (target, looter)`, `playerLeft (src, reason)`, `squadEliminated (squadId)`, `gameEnded (winningSquad)`.
- Tous les textes en jeu en **français**, dans le fichier de locale.

## 14. Configuration attendue (exemple)

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

Config.Squads = {
    maxSize = 4,           -- 1 = solo
    friendlyFire = false,  -- dégâts entre coéquipiers
}

Config.DisablePvpBeforeStart = true -- pas de dégâts entre inscrits avant le lancement

Config.Downed = {
    canCrawl = true,       -- le joueur au sol peut ramper lentement
}

Config.Revive = {
    duration = 8,          -- secondes d'action maintenue
    distance = 2.0,
    healthPercent = 50,    -- PV après réanimation (%)
}

Config.Carry = {
    distance = 2.0,
    speedMultiplier = 0.7,
    canSprint = false,
}

Config.Loot = {
    distance = 2.0,
    allowSquadmates = false,
    lootMoney = true,
    blacklist = { 'id_card', 'phone' }, -- noms à adapter à ton inventaire
}

Config.Respawn = {
    delay = 30,              -- secondes avant que « Retourner au spawn » soit disponible
    holdTime = 3,            -- secondes à maintenir la touche
    coords = nil,            -- nil = spawn du serveur, sinon vector4(x, y, z, heading)
    dropLootOnLeave = false, -- laisse le stuff restant dans un sac lootable
}

Config.EndResetDelay = 15         -- secondes avant de rouvrir la zone après une partie
Config.AdminPermission = 'brzone.admin'
```

Chaque option doit être commentée en français dans le fichier final.

## 15. Commandes

**Joueurs** :
- `/squad` : ouvre le menu de squad ;
- `/squad invite <id>`, `/squad accept`, `/squad leave`, `/squad kick <id>`.

**Admins** (protégées par permission) :
- `/br_setcenter` : place le centre de la zone sur ma position (appliqué immédiatement et sauvegardé, ou coordonnées affichées pour les copier dans la config) ;
- `/br_radius <m>` : change le rayon initial ;
- `/br_start` : force le lancement (ignore le minimum) ;
- `/br_stop` et `/br_reset` : arrête / réinitialise la partie ;
- `/br_down <id>` et `/br_revive <id>` : met au sol / réanime un joueur (pratique pour tester) ;
- `/br_debug` : affiche l'état, la phase, le rayon, les joueurs, les squads, les timers.

## 16. Tests

Je ne peux pas réunir 20 joueurs pour tester : prévois un **mode test** (minimum 1, maximum 2, phases courtes) pour valider tout le cycle seul ou à deux.

Fournis une **checklist de tests manuels**, au minimum :

1. J'entre dans la zone → je suis inscrit, le compteur se met à jour.
2. J'essaie de sortir à pied, en voiture, en moto rapide → je suis bloqué.
3. Minimum atteint → décompte ; un joueur se déconnecte → décompte annulé.
4. Maximum atteint → un non-inscrit ne peut pas entrer.
5. Lancement → mur rouge, rétrécissement fluide, carte à jour, squads figées.
6. Hors de la zone sûre → dégâts + effets ; à 0 PV → mis au sol.
7. Mis au sol par un joueur → au sol, pas mort, aucun écran de mort du serveur.
8. Un coéquipier me réanime → je me relève ; un ennemi ne peut pas me réanimer.
9. Un coéquipier puis un ennemi me portent, me posent ; le porteur est mis au sol → dépose automatique.
10. Un ennemi me loote (objet par objet, puis « Tout prendre ») ; limite de poids et liste noire respectées.
11. Après 30 s → « Retourner au spawn » apparaît ; si je reste au sol, je peux encore être réanimé ; après un retour au spawn, impossible de revenir dans la zone.
12. Toute une squad au sol → squad éliminée ; dernière squad debout → victoire, reset, joueurs au sol réanimés et renvoyés au spawn.
13. Déconnexion pendant un portage, un loot, une réanimation → rien ne reste bloqué.
14. Restart de la ressource en pleine partie → tout est propre.
15. Un joueur se connecte en cours de partie → il voit la zone correctement.

Relis ton code avant de me le rendre (syntaxe, nil checks, fuites de threads, états jamais nettoyés).

## 17. Livrables

- La ressource complète, prête à être ajoutée avec `ensure` dans le `server.cfg`.
- Un **README en français** : installation, dépendances, explication de chaque option de config, commandes, exports/events, fonctionnement des squads / KO / portage / loot / retour au spawn, comment brancher de nouvelles fonctionnalités.
- Un **résumé final** : ce qui a été fait, les choix techniques et pourquoi, les limites connues, et des idées pour la suite (achèvement, spectateur, kits, kill feed, classement, anti-cheat, routing bucket dédié, multi-arènes…).
