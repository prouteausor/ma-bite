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
   - le **système de mort / ambulance** (esx_ambulancejob, qb-ambulancejob, wasabi_ambulance…), comment il détecte la mort, s'il enregistre l'état « mort » en base, et s'il a déjà un état « last stand » / KO ;
   - l'**inventaire** (ox_inventory, qb-inventory, qs-inventory…), s'il sait déjà ouvrir / fouiller l'inventaire d'un autre joueur et avec quelles conditions, et les fouilles déjà présentes sur le serveur (joueur mort, menotté, mains en l'air) ;
   - un éventuel système de **target** (ox_target, qb-target), de **portage**, ou de **squads / groupes de joueurs** déjà présent ;
   - le **point de spawn** du serveur.
3. Vérifie : OneSync activé ? `lua54` ? Quel système de permissions admin (ACE, groupes du framework) ?
4. Présente-moi un **rapport court** + un **plan d'architecture** (arborescence, machines à états, flux des events, solution retenue pour le KO et pour le loot). **Attends ma validation avant d'implémenter.** Si un point est vraiment bloquant, pose la question ; sinon prends la décision la plus pro et signale-la.
5. **Ne modifie pas** la redzone, la bluezone, le système de mort ni l'inventaire. Si c'est indispensable (par exemple pour que le système de mort ignore les joueurs en partie, voir section 8), fais-le de façon minimale et réversible, et explique-le avant. Réutilise leurs conventions et leur système de notifications pour que tout soit cohérent visuellement. Vérifie qu'il n'y a aucun conflit (noms d'events, zones qui se chevauchent).

## 4. Fonctionnement attendu (cycle d'une partie)

Le **serveur gère l'état de la partie** : c'est lui qui décide qui est inscrit, dans quelle squad, debout, au sol ou éliminé, et qui peut réanimer, porter ou looter. Le client affiche et envoie les actions du joueur.

La zone suit une machine à états :

| État        | Couleur du mur       | Zone                                         | Entrer                        | Sortir (inscrit)          | Dégâts                                                                   |
|-------------|----------------------|----------------------------------------------|-------------------------------|---------------------------|--------------------------------------------------------------------------|
| `WAITING`   | Orange               | Fixe (rayon initial)                         | Ouvert tant que < max joueurs | Impossible                | Aucun pour les inscrits : lobby sûr                                      |
| `COUNTDOWN` | Orange (léger pulse) | Fixe, cible de la phase 1 affichée           | Ouvert tant que < max joueurs | Impossible                | Aucun pour les inscrits : lobby sûr                                      |
| `RUNNING`   | **Rouge**            | **Rétrécit dès le passage au rouge**         | **Fermé**                     | Impossible                | Activés entre inscrits (sauf coéquipiers si `friendlyFire = false`)      |
| `ENDED`     | Rouge, figé          | Figée, plus aucun dégât de zone              | Fermé                         | Libéré (joueurs debout)   | Aucun : personne ne peut tomber                                          |

Règles précises :

1. **Inscription** : un joueur qui entre physiquement dans la zone (à pied ou en véhicule) pendant `WAITING` ou `COUNTDOWN` est inscrit automatiquement : le client détecte l'entrée, le serveur l'inscrit s'il reste de la place. **Dès qu'il est inscrit, il ne peut plus sortir.** Le minimum et le maximum comptent des **joueurs**, pas des squads.
2. **Minimum atteint** (ex. 20) → passage en `COUNTDOWN` : **décompte de 60 s** (configurable), affiché à tous les inscrits, avec bip sonore sur les 10 dernières secondes. **La zone reste orange et ne rétrécit pas.** La cible de la phase 1 est affichée (contour blanc) pour que les joueurs voient où la zone va aller.
3. Pendant le décompte, d'autres joueurs peuvent encore entrer **jusqu'au maximum** (ex. 30).
4. **Maximum atteint** → la zone est **fermée immédiatement** : plus personne ne peut entrer (repoussé + message « Zone complète »). Le décompte de 60 s continue normalement (une option permet de le raccourcir quand la zone est pleine ; elle est désactivée par défaut).
5. Si le nombre d'inscrits **repasse sous le minimum** pendant le décompte (déconnexion…) → décompte annulé, retour en `WAITING` + notification. Exception : un décompte forcé par `/br_start` n'est pas annulé.
6. **Fin du décompte** → `RUNNING` : la zone est **verrouillée** (même si le max n'est pas atteint), le mur **passe au rouge et commence aussitôt à rétrécir**, les **squads sont figées**.
7. **Fin de partie** : après chaque changement d'état (mise au sol, réanimation, retour au spawn, déconnexion), le serveur revérifie l'élimination des squads, puis la condition de fin. Les mises au sol sont traitées une par une, dans l'ordre de réception : dès qu'il ne reste **qu'une seule squad avec au moins un membre debout**, elle **gagne immédiatement** (annonce à tous). Ça règle aussi le cas de squads qui tombent presque en même temps à cause de la zone. La victoire automatique n'est vérifiée que s'il y avait **au moins 2 squads au lancement**. Sinon (mode test, `/br_start` avec une seule squad), la partie ne se termine qu'avec `/br_stop` ou quand plus personne n'est debout. Si plus aucune squad n'a de membre debout, la partie se termine **sans vainqueur**.
8. **Pendant `ENDED`** (de la victoire jusqu'au reset) : plus aucun dégât (zone ou joueurs), plus aucune mise au sol. Les membres au sol de la squad gagnante sont réanimés sur place. Les vainqueurs peuvent encore **looter les joueurs au sol et les sacs de loot** : c'est la récompense du dernier combat. **Reset** après `Config.EndResetDelay` secondes : inventaires de loot fermés, joueurs encore au sol réanimés et renvoyés au spawn, sacs de loot supprimés, joueurs encore dans l'arène (vainqueurs compris) replacés juste à l'extérieur de la bordure, sans être inscrits. Puis la zone repasse en `WAITING`.
9. **Aucun dégât entre un participant et un non-participant**, dans les deux sens et dans tous les états (balles, armes blanches, explosions, véhicules) : personne ne peut tirer dans l'arène depuis l'extérieur, ni de l'arène vers l'extérieur. Un joueur qui a quitté la partie (retour au spawn, déconnexion) redevient un non-participant. C'est une règle de jeu, pas de l'anti-cheat.

**Technique pour les dégâts entre joueurs** : filtre-les **côté serveur** avec `weaponDamageEvent` (`CancelEvent()`) selon l'état de la partie (lobby sûr, coéquipiers, joueurs au sol, participant ↔ non-participant). N'utilise pas `NetworkSetFriendlyFireOption` (global, il casserait la redzone / bluezone) et ne compte pas sur `SetCanAttackFriendly` ni sur les relationship groups (locaux à chaque client). Les explosions et les écrasements en véhicule ne passent pas par `weaponDamageEvent` : gère-les si c'est faisable proprement, sinon documente la limite.

## 5. La zone qui rétrécit

### Deux modes de zone (`Config.ShrinkMode`)

- **`'damage'`** (par défaut, standard Battle Royale) :
  - la **bordure de l'arène** (rayon initial, fixe) est le mur **infranchissable** : un inscrit ne peut pas sortir, un non-inscrit ne peut pas entrer quand la zone est fermée. Pendant `RUNNING`, elle **reste visible** (couleur discrète `Config.Colors.border`, dessinée quand on s'en approche, avec son propre cercle sur la carte), pour qu'on ne bute jamais contre un mur invisible et qu'on ne la confonde pas avec la zone sûre ;
  - la **zone sûre rouge** rétrécit : un joueur debout qui se trouve dehors (entre elle et la bordure) subit des **dégâts croissants selon la phase**. C'est ce qui force les joueurs à se rapprocher et à se battre. Si ces dégâts font tomber ses PV à 0, il est **mis au sol** (voir section 8), comme avec une balle.
- **`'wall'`** : le mur rouge qui rétrécit est lui-même la barrière des inscrits et les repousse vers l'intérieur (les non-inscrits restent bloqués à la bordure fixe). Les joueurs au sol ou portés qu'il rattrape sont replacés juste à l'intérieur. Le rayon ne descend jamais sous `Config.Zone.wallMinRadius`. Une fois la dernière phase terminée, les joueurs debout subissent les dégâts de la dernière phase jusqu'à la fin de partie, pour garantir un vainqueur.

Règles communes :

- Avant le lancement, il n'y a qu'un seul mur : la bordure, en orange.
- Toutes les distances à la zone (inscription, barrière, zone sûre, dégâts) se calculent **en 2D** (`#(coords.xy - center.xy)`), jamais en 3D : la barrière s'applique à toutes les altitudes et on ne peut pas la survoler. Le mur commence **sous le sol** (ex. `center.z - 50`) pour ne laisser aucun trou dans les creux du terrain.
- Les **joueurs au sol ne subissent pas les dégâts de zone** (ils ne peuvent pas mourir). Un joueur réanimé hors de la zone sûre reprend les dégâts.
- **Dégâts de zone** : appliqués par le **client du joueur concerné** (lui seul peut modifier la vie de son ped), par tick d'une seconde. `damage` est en **PV par seconde**, retirés directement de la vie (le gilet ne protège pas de la zone). Si un tick doit faire passer le joueur sous le seuil de mort, **ne tue pas le ped** : déclenche directement la mise au sol (même fonction que pour une balle, cause « zone »), pour que le système de mort du serveur ne se déclenche jamais.

### Phases

Les phases sont définies dans un tableau de config. Pour chaque phase :
- `wait` : pause (s) avant le rétrécissement, pendant laquelle la **prochaine zone** est déjà affichée. **Pour la phase 1, `wait = 0` par défaut** : la zone commence à rétrécir dès qu'elle passe au rouge ;
- `duration` **ou** `speed` (m/s) : durée ou vitesse du rétrécissement (le script calcule l'autre) ;
- `radius` : rayon cible à la fin de la phase ;
- `damage` : dégâts par seconde hors de la zone sûre ;
- `moveCenter` : si `true`, le nouveau centre est tiré au hasard **de façon à ce que le nouveau cercle soit entièrement contenu dans l'ancien** (comme dans les vrais BR), et le centre glisse en douceur pendant le rétrécissement.

En mode `'damage'`, la dernière phase doit pouvoir descendre à `0` pour garantir une fin de partie.

### Rendu en jeu (point très important)

Je veux **voir la zone rétrécir de mes propres yeux**, de façon fluide et propre :

- **Mur cylindrique** visible de loin : hauteur configurable, légère transparence, effet de bandes ou de dégradé. Choisis la technique la plus propre et performante (par ex. segments en `DrawPoly` double face avec un nombre de segments configurable, en ne dessinant que les segments proches/visibles) et **justifie ton choix** (un `DrawMarker` géant pose souvent des problèmes de rendu à grande échelle).
- **Rétrécissement fluide image par image** : le serveur n'envoie que les paramètres de la phase (centre/rayon de départ et cible, durée, temps déjà écoulé) ; le client **interpole localement** à chaque frame. Aucun envoi réseau en boucle.
- **Couleurs configurables** : orange (attente), orange pulsé (décompte), rouge (partie), blanc/contour pour la prochaine zone, couleur discrète pour la bordure fixe.
- **Carte / minimap** : cercle (`AddBlipForRadius`) mis à jour pendant le rétrécissement, cercle de la prochaine zone, cercle de la bordure, blip au centre.
- **Effets** : filtre rouge + son quand on est hors de la zone sûre, bips du décompte, annonces de phase (« La zone se rétrécit dans 30 s »).
- **HUD propre et discret** (dans le style du serveur) : état, joueurs `X/max` (ex. `12/30`), décompte, phase `N/M`, temps avant le prochain rétrécissement, joueurs et squads en vie, distance à la zone sûre, état de ma squad.

## 6. Barrière « impossible de sortir / impossible d'entrer »

C'est une mécanique de jeu côté client :

- À l'approche de la bordure, le joueur inscrit est **repoussé vers l'intérieur**. S'il la dépasse quand même (vitesse, véhicule, ragdoll), il est replacé juste à l'intérieur, **au sol (bon Z)**. Gère les piétons et les véhicules (voitures rapides, motos, véhicules aériens). Pendant un portage, la barrière ne s'applique qu'au porteur : le joueur porté le suit.
- **Véhicules** : seul le propriétaire réseau (en général le conducteur) peut déplacer le véhicule. Si les occupants ne sont pas soumis à la même règle (ex. conducteur inscrit, passagers refusés parce que la zone vient d'être pleine), chaque joueur concerné **sort lui-même du véhicule** et est replacé du bon côté de la bordure.
- **Non-inscrit quand la zone est fermée** : même logique, mais repoussé vers l'extérieur, avec un message clair (« Partie en cours » / « Zone complète »).
- **Non-participant déjà à l'intérieur d'une arène fermée** (démarrage de la ressource avec des joueurs dedans, passager refusé…) : il n'est **jamais inscrit d'office**. Il est replacé juste à l'extérieur de la bordure, au sol, avec un message.
- Un joueur qui s'est **déconnecté en pleine partie** (debout ou au sol) revient **debout et vivant, au point de spawn**, et pas à sa dernière position dans l'arène, même si le framework l'y remet.
- Un joueur **au sol reste retenu par la barrière**, même si sa squad est éliminée (sinon il pourrait ramper hors de l'arène pour ne pas être looté). Ne sont plus retenus que les joueurs qui ont **quitté la partie** (retour au spawn, déconnexion), les joueurs debout pendant `ENDED`, et l'admin en mode observateur.
- **Mode observateur admin** : désactivé par défaut, activé / désactivé avec `/br_spectate` (protégée par permission). Quand il est actif, l'admin n'est pas inscrit, la barrière ne le retient pas et il ne peut ni blesser ni être blessé par les joueurs de la partie. Quand il est inactif, un admin est traité comme n'importe quel joueur (indispensable pour que je puisse tester moi-même).

## 7. Squads

- **Si le serveur a déjà un système de squads / groupes de joueurs** (repéré à l'étape 0 ; pas les jobs ni les gangs), **réutilise-le via le bridge** au lieu d'en créer un second, et signale-le dans ton plan. La taille max et le figeage au lancement s'appliquent quand même. Sinon, crée le système décrit ci-dessous.
- **Taille max configurable** (`Config.Squads.maxSize`, ex. 4 ; `1` = mode solo). Un joueur sans squad joue seul (squad d'un seul joueur).
- **Gestion avant le lancement** (dehors, ou dans la zone pendant `WAITING` / `COUNTDOWN`) : créer une squad, inviter un joueur par son ID, accepter / refuser, quitter, exclure (chef de squad). Via commandes + menu propre (ox_lib context ou équivalent du serveur).
- **Au lancement, les squads sont figées** : les membres qui ne sont pas dans la zone sont retirés de la squad.
- S'il ne reste pas assez de places pour toute une squad, les premiers entrés sont inscrits et les suivants sont refusés avec un message clair.
- **Pas de dégâts entre coéquipiers** (configurable).
- **Coéquipiers visibles** : blip sur la minimap + marqueur discret au-dessus de la tête, et un HUD squad avec pour chaque membre : nom, état (debout / au sol / éliminé), vie.
- **Squad éliminée** : quand tous ses membres sont au sol ou éliminés. Tous ses membres sont alors éliminés de la partie (plus aucune réanimation possible), mais ceux qui sont au sol **restent au sol, lootables**, et gardent la possibilité de retourner au spawn.

## 8. Mise au sol (KO) au lieu de la mort

Pendant `RUNNING`, un joueur inscrit **ne meurt jamais** : quand ses PV tombent à 0 (balle, arme blanche, explosion, chute, véhicule, zone…), il est **mis au sol (KO)**. Avant le lancement, le lobby est sûr : les inscrits ne prennent **aucun dégât** (joueurs, chute, PNJ…), donc personne ne peut tomber ni mourir.

- **Animation au sol** propre (blessé), synchronisée pour tous les joueurs. Option pour ramper lentement (`Config.Downed.canCrawl`).
- **Au sol** : pas d'arme en main, pas de tir, pas d'accès à son inventaire ni à la hotbar, pas de véhicule, pas de sprint. Passe par l'**API de l'inventaire**, pas seulement par des natives, sinon l'arme équipée et la hotbar se désynchronisent (ex. ox_inventory : event client `ox_inventory:disarm` + `LocalPlayer.state.invBusy` / `invHotkeys` ; qb-inventory : `LocalPlayer.state.inv_busy`). Même chose pour le porteur.
- **Mis au sol dans un véhicule** → éjecté et posé à côté. **Dans l'eau** → maintenu à la surface ou posé sur la rive la plus proche (choisis le plus fiable).
- **Le système de mort / ambulance du serveur est ignoré** pour les joueurs en partie : pas d'écran de mort, pas de timer de mort, pas de laststand du framework, pas de respawn à l'hôpital, pas d'appel EMS. Hors de la partie, tout redevient normal.
- **Invulnérable au sol** : personne ne peut l'achever pour l'instant. Prévois un point d'accroche propre pour ajouter un achèvement plus tard.
- Enregistre **qui l'a mis au sol** (attaquant, arme, ou « zone ») → event `playerDowned`, pour un futur kill feed / des stats.
- **HUD au sol** : « Vous êtes au sol », état de la squad, temps restant avant que le retour au spawn soit disponible, et qui est en train de te réanimer / porter / looter.
- **Technique** : choisis l'approche la plus fiable pour intercepter la mort (par ex. empêcher les PV de descendre sous le seuil de mort, ou réanimer instantanément le joueur avec `NetworkResurrectLocalPlayer` au moment du décès puis le passer en KO) et **justifie ton choix**. Pièges à gérer :
  - si ton approche laisse le ped mourir, même un instant, les handlers de mort des autres ressources se déclenchent quand même (`gameEventTriggered` / `CEventNetworkEntityDamage`, `esx:onPlayerDeath` émis par es_extended lui-même, `baseevents`…), et leur ordre d'exécution **n'est pas garanti** : tu ne peux pas « passer avant ». Dans ce cas, c'est le système de mort qui doit **ignorer** les joueurs en partie : expose un state bag (ex. `Player(src).state.brInGame`) et ajoute **un seul test en tête de son handler** (modification minimale, réversible, signalée dans le plan et documentée dans le README) ;
  - certains systèmes enregistrent l'état « mort » en base (`is_dead` d'esx_ambulancejob, métadonnées `isdead` / `inlaststand` de QBCore) et re-tuent le joueur à sa reconnexion : aucun de ces états ne doit rester à `true` pour un joueur de la partie.

### Réanimation

- **Seul un membre de sa squad**, debout et proche, peut le réanimer.
- **Action maintenue** (touche ou target) avec barre de progression (durée configurable, ex. 8 s) et animation (accroupi, soins).
- Annulée si le réanimateur relâche, s'éloigne ou est mis au sol. Une seule réanimation à la fois sur un même joueur.
- Impossible tant que le joueur au sol est porté (il faut le poser d'abord).
- Le joueur réanimé se relève avec un pourcentage de PV configurable, retrouve tous ses mouvements, reste dans la partie et dans la zone.
- **PV GTA** : un ped joueur a en général 200 PV max et il est mort / à terre sous ~100 ; le max peut varier selon le ped ou le serveur. Calcule tous les pourcentages (réanimation, HUD squad) sur la plage utile `100 → GetEntityMaxHealth(ped)`, jamais sur 0 → 200. Exemple : `healthPercent = 50` → `100 + (max - 100) * 0.5`.

## 9. Porter un joueur au sol

- **N'importe quel joueur debout de la partie** peut porter un joueur au sol, **coéquipier ou ennemi**.
- Prise / dépose avec une touche ou le target. **Animation de portage synchronisée** des deux côtés (type « fireman carry »).
- **OneSync** : un client ne peut attacher / déplacer que les entités qu'il possède. C'est donc le **client du joueur porté** qui attache son propre ped à celui du porteur (`AttachEntityToEntity`) sur ordre du serveur ; chaque client joue l'animation sur son propre ped. Le détachement se fait aussi de son côté, avec un filet de sécurité : si le ped du porteur devient introuvable, il se détache et se pose au sol.
- Le porteur : vitesse réduite (`Config.Carry.speedMultiplier`), sprint selon `Config.Carry.canSprint`, et **jamais** de tir ni de véhicule.
- **Dépose automatique** si : le porteur est mis au sol, se déconnecte ou tente d'entrer dans un véhicule ; le joueur porté retourne au spawn ou se déconnecte ; la partie se termine.
- Un joueur ne porte qu'une seule personne, et un joueur au sol n'est porté que par une seule personne à la fois.
- Le portage respecte la barrière (impossible de sortir de l'arène en portant quelqu'un).
- Pas de loot ni de réanimation pendant le portage.

## 10. Loot

- Quand un joueur est au sol, un **ennemi debout** qui se place **juste à côté de lui** (distance configurable) peut le **looter** : il ouvre l'inventaire du joueur au sol et récupère ce qu'il veut, objet par objet **ou tout d'un coup** (action « Tout prendre »).
- **Tout ce que le joueur a sur lui dans son inventaire** est lootable : objets, armes, munitions (et argent liquide si activé en config). Sur ESX sans ox_inventory, les armes sont dans le **loadout** et l'argent dans les **comptes** : ils doivent aussi être lootables.
- **Utilise le système d'inventaire du serveur**, ouvert **par le serveur** (ex. ox_inventory : `exports.ox_inventory:forceOpenInventory(looter, 'player', cible)` ; qb-inventory récent : `OpenInventoryById`). Attention : ox_inventory peut refermer la fouille d'un autre joueur si la cible n'est ni morte, ni menottée, ni mains en l'air, ou si elle est trop loin (environ 1,8 m). Vérifie ce comportement dans la version installée et adapte l'animation du KO et `Config.Loot.distance`, **sans modifier l'inventaire**. Sans fouille native utilisable, crée une interface de loot propre, avec le transfert des objets fait par le serveur.
- **« Tout prendre »** n'existe pas dans les interfaces natives des inventaires : ne modifie pas leur NUI. Ajoute-le comme **action séparée** (option du target ou touche) qui transfère côté serveur tout ce que le looter peut porter.
- **Les règles du mode priment sur la fouille native** : si elle ne permet pas d'appliquer la liste noire, le « un seul looter » et l'interdiction entre coéquipiers (par ex. via les hooks d'ox_inventory), crée l'interface dédiée. Vérifie aussi que les fouilles déjà présentes sur le serveur (joueur mort, menotté, mains en l'air) ne s'appliquent pas aux participants pendant la partie, via les hooks ou exports disponibles.
- Le serveur vérifie que le joueur est bien au sol et que personne d'autre n'est déjà en train de le looter avant de transférer quoi que ce soit.
- Respecte les limites de poids / slots du looter : ce qui ne rentre pas reste sur le joueur au sol.
- **Liste noire** configurable d'objets non lootables, **vide par défaut** : tout est lootable. Je pourrai y ajouter des objets plus tard (ex. carte d'identité, téléphone).
- Option pour autoriser les coéquipiers à se looter entre eux (`Config.Loot.allowSquadmates`, désactivée par défaut).
- **Un seul looter à la fois** par joueur au sol. L'inventaire se ferme si le looter s'éloigne ou est mis au sol, ou si le joueur au sol est réanimé, porté ou retourne au spawn.
- Le joueur au sol voit qu'il est en train d'être looté.
- Le loot reste possible pendant `ENDED`, jusqu'au reset.

## 11. Retour au spawn

- **30 s après être tombé** (configurable), le joueur au sol voit apparaître l'option **« Retourner au spawn »** (touche à maintenir quelques secondes pour éviter les erreurs).
- **Ce n'est pas obligatoire** : il peut rester au sol pour attendre d'être réanimé par sa squad, aussi longtemps que la partie dure.
- S'il l'utilise : il **abandonne la partie** (éliminé, plus réanimable, ne peut plus revenir dans la zone pendant cette partie), il est téléporté au spawn (point configurable, par défaut le spawn du serveur) et soigné.
- **Son stuff reste sur place** : tout ce qui n'a pas encore été looté est déposé à l'endroit où il était au sol, dans un **sac lootable** (`Config.Respawn.dropLootOnLeave = true` par défaut ; `false` = il garde son stuff). Sinon, il suffirait d'attendre 30 s et de partir pour ne jamais se faire looter.
- La **déconnexion au sol** suit exactement la même règle : le serveur crée le sac au moment de la déconnexion.
- Le sac se loote avec les mêmes règles qu'un joueur au sol (ennemis de la partie, distance, un seul looter à la fois, poids / slots). Il est supprimé au reset de la zone et au restart de la ressource.
- Sa squad continue sans lui si elle a encore des membres debout.

## 12. Cas limites à gérer

- **Déconnexion** : debout → retiré, compteurs recalculés, condition de victoire vérifiée ; au sol → traité comme un retour au spawn (sac de loot compris) ; en train de porter / réanimer / looter → action annulée proprement pour l'autre joueur. À la reconnexion : debout, vivant, au point de spawn (voir section 6).
- **Joueur qui se connecte en cours de partie** : il n'est pas participant, ne peut pas entrer, mais voit la zone correctement.
- **Actions simultanées** (deux joueurs veulent porter, looter ou réanimer la même personne ; plusieurs joueurs entrent alors qu'il reste 1 place) : le serveur attribue au premier, les autres reçoivent un message.
- **Restart de la ressource** en pleine partie : tous les joueurs au sol sont réanimés, les portages détachés, les inventaires de loot fermés, les sacs supprimés, blips / threads / HUD / effets nettoyés, état propre.
- Chevauchement avec la redzone / bluezone.

## 13. Architecture et qualité du code

Arborescence indicative (adapte les noms aux conventions du serveur) :

```
<nom_ressource>/
├── fxmanifest.lua
├── config.lua
├── locales/fr.lua
├── bridge/                -- adaptateurs framework / inventaire / mort / notifs / target / squads
├── shared/utils.lua       -- maths du cercle (2D), interpolation
├── server/main.lua        -- machine à états, inscriptions, phases, fin de partie
├── server/damage.lua      -- filtrage des dégâts entre joueurs
├── server/squads.lua
├── server/downed.lua      -- KO, réanimation, retour au spawn
├── server/carry.lua
├── server/loot.lua        -- loot, « Tout prendre », sacs
├── server/commands.lua
├── client/main.lua        -- synchro de l'état, détection
├── client/render.lua      -- murs, blips
├── client/barrier.lua
├── client/downed.lua
├── client/carry.lua
├── client/loot.lua
├── client/hud.lua         (+ html/ si NUI)
└── README.md
```

Exigences :

- **Pas d'anti-cheat** : ne code **aucun** système anti-triche (détection de téléportation, de noclip ou de godmode, surveillance périodique des positions, bannissements, kicks, logs de triche…). Je le ferai plus tard. Les contrôles qui font partie des **règles du jeu** restent gérés par le serveur : état du joueur (debout / au sol / éliminé), squad, place disponible, une seule action à la fois sur un même joueur. Pour brancher mon anti-cheat plus tard, les exports et events listés plus bas suffisent : n'ajoute rien d'autre.
- **Synchronisation** via GlobalState / state bags ou events ciblés, jamais de boucle réseau par frame. Ne compare jamais l'horloge du client à celle du serveur : le serveur garde l'instant de début de phase avec son propre `GetGameTimer()`. À son chargement (connexion en cours de partie, restart), le client **demande l'état** et reçoit le temps écoulé calculé au moment de la réponse, puis interpole localement avec son `GetGameTimer()`. Ne stocke jamais un « temps écoulé » figé dans GlobalState (il serait périmé). `AddStateBagChangeHandler` ne se déclenche pas pour les valeurs déjà présentes : lis aussi `GlobalState` au démarrage du client. Les clés GlobalState / state bags **survivent au restart de la ressource** : réinitialise-les au démarrage.
- **Performance** : threads adaptatifs (`Wait(1000)` ou plus loin de la zone, `Wait(0)` uniquement quand il faut dessiner ou gérer une interaction). Objectif resmon : < 0.05 ms hors zone, le plus bas possible en zone (vise < 0.3 ms).
- Lua 5.4, variables `local`, pas de globales inutiles, logs uniquement si `Config.Debug`.
- **Validation de la config au démarrage** (min ≤ max, rayons décroissants, valeurs positives…) avec messages d'erreur clairs. Hors mode test, avertissement si `Config.Players.min <= Config.Squads.maxSize` (une seule squad pourrait suffire à lancer la partie).
- **Bridge** : n'écris que les adaptateurs des systèmes réellement utilisés par mon serveur, mais structure-les pour pouvoir en ajouter d'autres facilement.
- Encapsule la logique d'une arène dans un objet pour pouvoir gérer **plusieurs arènes plus tard** (n'en implémente qu'une pour l'instant).
- **Touches** déclarées avec `RegisterKeyMapping` (modifiables par le joueur dans les paramètres GTA).
- **Extensibilité** — exports et events publics documentés, pour que je branche kits / récompenses / classement / anti-cheat sans toucher au cœur :
  - exports : `GetGameState()`, `IsPlayerInGame(src)`, `GetPlayerSquad(src)`, `IsPlayerDowned(src)`, `GetAlivePlayers()`, `RevivePlayer(src)`, `ForceStart()`, `StopGame()` ;
  - events serveur, **tous préfixés par le nom de la ressource** (ex. `br_zone:playerDowned`, de même pour les callbacks et les clés de state bag), pour éviter tout conflit avec les events natifs (`playerJoining`, `playerDropped`) et ceux des autres ressources : `playerJoined (src)`, `countdownStarted`, `countdownCancelled`, `gameStarted`, `phaseChanged (phase)`, `playerDowned (src, attacker, weapon, cause)` (dégâts de zone : `attacker = nil`, `cause = 'zone'`), `playerRevived (src, reviver)`, `playerLooted (target, looter)`, `playerLeft (src, reason)` (`reason` : `'spawn'` ou `'disconnect'`), `squadEliminated (squadId)`, `gameEnded (winningSquad)` (`nil` si pas de vainqueur).
- Tous les textes en jeu en **français**, dans le fichier de locale.

## 14. Configuration attendue (exemple)

```lua
Config = {}

Config.Debug = false

Config.Zone = {
    center = vector3(0.0, 0.0, 0.0), -- à placer en jeu avec /br_setcenter
    radius = 250.0,                  -- rayon initial (m)
    wallHeight = 150.0,
    wallMinRadius = 15.0,            -- mode 'wall' uniquement : rayon minimal du mur (jamais 0)
}

Config.Players = { min = 20, max = 30 }

Config.Countdown = {
    duration = 60,   -- secondes avant le lancement une fois le minimum atteint
    whenFull = nil,  -- optionnel : zone pleine → décompte ramené à X s s'il en reste plus (nil = désactivé)
}

Config.ShrinkMode = 'damage' -- 'damage' | 'wall'

-- duration (s) OU speed (m/s) pour chaque phase
Config.Phases = {
    { wait = 0,  duration = 90, radius = 175.0, damage = 1,  moveCenter = true  }, -- rétrécit dès le passage au rouge
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
    border    = { r = 40,  g = 40,  b = 40,  a = 70  }, -- bordure fixe pendant la partie (mode 'damage')
}

Config.Squads = {
    maxSize = 4,           -- 1 = solo
    friendlyFire = false,  -- dégâts entre coéquipiers
}

Config.Downed = {
    canCrawl = true,       -- le joueur au sol peut ramper lentement
}

Config.Revive = {
    duration = 8,          -- secondes d'action maintenue
    distance = 2.0,
    healthPercent = 50,    -- % de la plage de vie utile (100 → max), voir section 8
}

Config.Carry = {
    distance = 2.0,
    speedMultiplier = 0.7,
    canSprint = false,
}

Config.Loot = {
    distance = 1.5,          -- à ajuster selon l'inventaire (ox_inventory referme la fouille au-delà d'environ 1,8 m)
    allowSquadmates = false,
    lootMoney = true,
    blacklist = {},          -- vide = tout est lootable ; ex. { 'id_card', 'phone' } (noms à adapter)
}

Config.Respawn = {
    delay = 30,              -- secondes avant que « Retourner au spawn » soit disponible
    holdTime = 3,            -- secondes à maintenir la touche
    coords = nil,            -- nil = spawn du serveur, sinon vector4(x, y, z, heading)
    dropLootOnLeave = true,  -- retour au spawn ou déconnexion au sol : le stuff non looté reste sur place dans un sac
}

Config.EndResetDelay = 60         -- secondes entre la victoire et le reset (les vainqueurs peuvent looter)
Config.AdminPermission = 'brzone.admin'

Config.TestMode = {
    enabled = false,           -- true = remplace Config.Players, Config.Countdown.duration et Config.Phases
    players = { min = 1, max = 3 },
    countdown = 10,
    disableWinCheck = true,    -- pas de victoire automatique : fin avec /br_stop ou quand plus personne n'est debout
    phases = {
        { wait = 0,  duration = 20, radius = 120.0, damage = 2, moveCenter = true  },
        { wait = 10, duration = 20, radius = 0.0,   damage = 5, moveCenter = false },
    },
}
```

Chaque option doit être commentée en français dans le fichier final.

## 15. Commandes

**Joueurs** :
- `/squad` : ouvre le menu de squad ;
- `/squad invite <id>` (crée la squad si le joueur n'en a pas encore), `/squad accept`, `/squad decline`, `/squad leave`, `/squad kick <id>` (chef de squad uniquement).

**Admins** (protégées par permission) :
- `/br_setcenter` : place le centre de la zone sur ma position. Utilisable seulement en `WAITING` sans inscrit. Appliqué immédiatement (jusqu'au prochain redémarrage de la ressource), et coordonnées affichées dans la console au format `vector3(...)` pour que je les recopie dans `config.lua` ;
- `/br_radius <m>` : change le rayon initial (mêmes règles que `/br_setcenter`) ;
- `/br_start` : lance le décompte même si le minimum n'est pas atteint (ce décompte forcé n'est pas annulé par la règle 5) ;
- `/br_stop` : arrête la partie sans vainqueur (ou vide les inscrits en `WAITING` / `COUNTDOWN`), avec le même nettoyage qu'au reset, puis retour en `WAITING` ;
- `/br_spectate` : active / désactive le mode observateur (pas inscrit, pas retenu par la barrière) ;
- `/br_down <id>` et `/br_revive <id>` : met au sol / réanime un joueur (pratique pour tester) ;
- `/br_debug` : affiche l'état, la phase, le rayon, les joueurs, les squads, les timers.

## 16. Tests

Je ne peux pas réunir 20 joueurs pour tester : prévois le **mode test** de la config (`Config.TestMode`). Je dois pouvoir tester le KO, la réanimation, le portage, le loot et le retour au spawn à 2 ou 3 joueurs (moi, un coéquipier, un ennemi) sans que la partie s'arrête dès qu'une squad tombe. Pour tester la victoire, je passe `disableWinCheck` à `false`.

Fournis une **checklist de tests manuels**, au minimum :

1. J'entre dans la zone (mode observateur désactivé) → je suis inscrit, le compteur se met à jour.
2. J'essaie de sortir à pied, en voiture, en moto rapide → je suis bloqué.
3. Minimum atteint → décompte, zone orange fixe avec la cible de la phase 1 affichée ; un joueur se déconnecte → décompte annulé.
4. Maximum atteint → un non-inscrit ne peut pas entrer.
5. Lancement → mur rouge qui rétrécit tout de suite, de façon fluide, carte à jour, squads figées, bordure fixe visible.
6. Hors de la zone sûre → dégâts + effets ; à 0 PV → mis au sol, sans écran de mort.
7. Mis au sol par un joueur → au sol, pas mort, aucun écran de mort du serveur ; je me déconnecte au sol puis je me reconnecte → je suis debout au spawn, pas re-tué, et un sac avec mon stuff est resté sur place.
8. Un coéquipier me réanime → je me relève avec le bon nombre de PV ; un ennemi ne peut pas me réanimer.
9. Un coéquipier puis un ennemi me portent, me posent ; le porteur est mis au sol → dépose automatique.
10. Un ennemi me loote (objet par objet, puis « Tout prendre ») ; la limite de poids est respectée ; un deuxième looter est refusé.
11. Après 30 s → « Retourner au spawn » apparaît ; si je reste au sol, je peux encore être réanimé ; après un retour au spawn, mon stuff restant est dans un sac et je ne peux pas revenir dans la zone.
12. Un joueur hors de la partie tire dans l'arène (et inversement) → aucun dégât.
13. Toute une squad au sol → squad éliminée ; dernière squad debout → victoire, les vainqueurs peuvent looter, puis reset (joueurs au sol réanimés et renvoyés au spawn, vainqueurs replacés dehors).
14. Déconnexion pendant un portage, un loot, une réanimation → rien ne reste bloqué.
15. Restart de la ressource en pleine partie → tout est propre.
16. Un joueur se connecte en cours de partie → il voit la zone correctement, il ne peut pas entrer.

Relis ton code avant de me le rendre (syntaxe, nil checks, fuites de threads, états jamais nettoyés).

## 17. Livrables

- La ressource complète, prête à être ajoutée avec `ensure` dans le `server.cfg`.
- Un **README en français** : installation, dépendances, explication de chaque option de config, commandes, exports/events, fonctionnement des squads / KO / portage / loot / retour au spawn, modifications éventuelles faites au système de mort, comment brancher de nouvelles fonctionnalités.
- Un **résumé final** : ce qui a été fait, les choix techniques et pourquoi, les limites connues, et des idées pour la suite (achèvement, spectateur, kits, kill feed, classement, routing bucket dédié, multi-arènes…).
