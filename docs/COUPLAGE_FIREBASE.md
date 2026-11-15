# Coupler deux appareils (Firebase)

*Version d'essai 12.1.1 (couplage inchangé depuis la 11.1.1).* Le couplage permet à deux appareils, ou davantage, de **retrouver les mêmes fiches presque en temps réel** : un téléphone ou une tablette au chevet, un ordinateur au bureau.
C'est une fonction **facultative** : tant qu'aucun couplage n'est créé, ADP ne fait aucune requête réseau (hors dictée vocale).

Menu **Plus › Couplage entre appareils…** (icône en forme de maillon). Une pastille dans la barre d'état rappelle l'état (« couplé · seul », « couplé · 2 appareils », « hors ligne », « accès refusé »).

---

## 1. Ce qui est synchronisé

| | |
|---|---|
| **Oui** | toutes les fiches qui portent un **numéro de dossier**, champ par champ ; la suppression d'une fiche |
| **Non** | les fiches sans numéro de dossier (brouillons) ; l'**identité** (nom, prénom, date de naissance) **sauf si la case est cochée** ; les champs calculés (rang d'examen, âge en mois) que chaque appareil recalcule ; les réglages ; les photos des pièces jointes ; un champ de plus de 28 000 caractères (compté dans « trop longs, non envoyés ») |

- **Fiche ouverte** : une modification faite sur un appareil arrive dans la fiche ouverte sur l'autre, en une ou deux secondes. Si vous n'avez rien modifié, la fiche n'est pas marquée « non enregistrée » ; la question *« Cette fiche est déjà enregistrée.
  Êtes-vous sûr de vouloir la modifier ? »* n'est pas posée pour une modification **reçue**. Pendant une saisie dans un champ, l'affichage attend que vous ayez fini.
- **Ce qui part est enregistré** : tant que le couplage est en place, la copie enregistrée de la fiche que vous modifiez suit ce qui est envoyé. Une fiche abandonnée sans enregistrer ne laisse donc jamais, sur l'appareil, une version plus ancienne que celle des autres.
  Le bouton **Enregistrer** reste celui qui écrit le fichier JSON dans le dossier de données.
- **Hors ligne** : tout continue ; les modifications partent à la reconnexion. À chaque connexion, l'état de la base est lu **d'abord** et réconcilié avec l'appareil, **avant** tout envoi.
- **Deux appareils modifient la même fiche** : champs différents → les deux modifications sont gardées ; même champ → la plus récente l'emporte. Une copie périmée (fiche ouverte longtemps après, appareil resté hors ligne) ne remplace **jamais** une valeur plus récente.
- **Suppression** : supprimer une fiche sur un appareil la supprime sur les autres (elle est rangée dans `fiches/supprimees`, comme d'habitude). Une fiche ouverte et en cours de modification n'est jamais supprimée sous vos mains.
- **Plus de deux appareils** : possible, avec le même code.

---

## 2. Sécurité et données de santé

- **Chiffrement de bout en bout.** Chaque champ est chiffré **dans le navigateur** (AES-GCM 256 bits) avant d'être envoyé ; la clé est dérivée du **code de couplage** (PBKDF2, 100 000 itérations). Le **nom de chaque champ** est haché (HMAC). Le code **ne quitte jamais l'appareil**.
  Ce que Firebase voit : des textes illisibles, leur taille, leur nombre, l'heure de chaque modification, les identifiants de fiche (ceux d'ADP, sans rapport avec le dossier), le nombre d'appareils connectés et leurs adresses IP.
- **Le salon.** Le nœud de la base qui porte le couplage est dérivé du code par SHA-256 (32 caractères) : sans le code, on ne peut ni le deviner, ni le lire, ni déchiffrer. Les règles de sécurité n'ouvrent que `/adp/<salon>`, jamais `/adp` lui-même.
- **Gardez le code secret** : qui le connaît lit les fiches du couplage. Il est conservé sur l'appareil (jamais dans les réglages exportés ni dans les sauvegardes). Perdu, il ne se retrouve pas : créez un nouveau couplage.
- **L'identité n'est pas envoyée par défaut.** Cochez « envoyer aussi l'identité du patient » pour la partager ; chaque appareil décide pour lui-même (un appareil qui ne l'envoie pas ne reçoit pas non plus celle des autres).
  Décocher n'efface pas ce qui est déjà en ligne : utilisez « Interrompre et effacer les données en ligne ».
- **Texte libre.** Les champs de texte libre (histoire, notes…) partent tels quels, chiffrés : si vous y écrivez un nom, il part.
- **Hébergement de données de santé.** La base Firebase **n'est pas un hébergeur certifié** pour les données de santé. Le chiffrement protège le contenu, pas le fait qu'un échange a lieu. **Testez avec des fiches fictives**, et faites valider l'usage réel par le référent
  sécurité de l'établissement avant de coupler des fiches de patients.

---

## 3. Mise en place

### Étape 1 — Installer les règles de sécurité (une seule fois, dans la console Firebase)

1. Console Firebase → votre projet → **Realtime Database** → onglet **Règles**.
2. Remplacez le contenu par celui du fichier **`docs/regles_firebase.json`** (ou, dans ADP : *Plus › Couplage entre appareils… › Règles Firebase à copier › Copier les règles*), puis **Publier**.
3. Si cette base sert déjà à autre chose, **n'écrasez pas vos règles** : ajoutez seulement le bloc `"adp": { … }` à l'intérieur de `"rules"`.
4. Facultatif : le **Simulateur de règles** de la console. Lecture de `/adp/` suivi de 32 lettres ou chiffres : **autorisée** ; lecture de `/adp` ou de `/` : **refusée**.

Les règles disent : tout est fermé, sauf `/adp/<salon>` quand `<salon>` fait exactement 32 caractères (lettres, chiffres, `-`, `_`) ; sous un salon, seuls quatre nœuds existent (`h` le contrôle, `d` les champs, `m` la méta des fiches, `p` la présence), chacun de la forme
`{t : nombre, c : texte de 30 000 caractères au plus}`, et rien d'autre.

### Étape 2 — Vérifier la base

Sur l'appareil : *Plus › Couplage entre appareils… › Tester la connexion*. Trois lignes doivent s'afficher : base joignable et lecture autorisée ; écriture de contrôle autorisée ; flux temps réel qui fonctionne. L'adresse de la base est
`https://lesacacaisson-default-rtdb.europe-west1.firebasedatabase.app` par défaut ; elle se change avant la création du couplage.

### Étape 3 — Créer le couplage (premier appareil)

*Créer un couplage* : un **code de couplage** de 24 caractères s'affiche (par groupes de quatre). **Notez-le** (bouton *Copier*, ou *Afficher* puis recopiez-le) : il n'est montré que sur demande ensuite.

### Étape 4 — Rejoindre (deuxième appareil)

*Plus › Couplage entre appareils…*, saisissez le code (majuscules ou minuscules, avec ou sans tirets) puis *Rejoindre*. Un code erroné ne trouve aucun couplage et rien n'est conservé.

### Étape 5 — Vérifier

Les deux appareils affichent « **couplé · 2 appareils** ». Enregistrez une **fiche fictive** (avec un numéro de dossier) sur l'un : elle apparaît sur l'autre en quelques secondes.

---

## 4. Au quotidien

- **Donner un nom à l'appareil** (« Tablette caisson », « PC bureau ») : il s'affiche sur les autres appareils.
- **Synchroniser maintenant** : relit l'état de la base et renvoie ce qui attend (utile après une longue coupure).
- **Interrompre le couplage** : cet appareil cesse d'échanger ; ses fiches restent ; la base garde les données chiffrées.
- **Interrompre et effacer les données en ligne** : supprime en plus le salon dans la base. Les **autres appareils** le voient (« couplage effacé sur un autre appareil ») et n'envoient plus rien ; interrompez-le aussi chez eux.
- La première connexion d'un appareil qui a déjà beaucoup de fiches envoie tout : comptez quelques secondes pour quelques dizaines de fiches.

---

## 5. Dépannage

| Message | Cause probable | Que faire |
|---|---|---|
| **Accès refusé** | les règles ne sont pas installées, ou sont trop strictes | étape 1, puis *Tester la connexion* |
| **Hors ligne, nouvel essai** | pas de réseau, pare-feu, proxy qui coupe les flux | les modifications attendent ; le flux se rétablit seul. *Tester la connexion* dit où ça bloque (ligne 1 : réseau ; ligne 3 : flux) |
| **Aucun couplage ne porte ce code** | faute de frappe, ou couplage non créé, ou effacé | recopier le code ; vérifier l'adresse de la base |
| **Code de couplage : 24 caractères** | le code comprend 24 lettres et chiffres, sans 0, 1, I ni O | recopier |
| **Ce navigateur ne permet pas le chiffrement** | page ouverte en `http://` non sécurisé, ou navigateur trop ancien | ouvrir par `https://` (GitHub Pages) ou en double-clic sur `index.html` |
| « trop longs, non envoyés : N » | un champ dépasse 28 000 caractères (photo collée dans un texte, par exemple) | raccourcir le champ ; il reste sur l'appareil |
| Une fiche n'arrive pas | pas de numéro de dossier, ou fiche supprimée ailleurs | saisir le numéro de dossier ; vérifier la liste de l'autre appareil |

---

## 6. Pour les développeurs

- Code : section « COUPLAGE ENTRE APPAREILS (Firebase Realtime Database) » de `index.html` (préfixe `fb…`, objet `FB`). Aucune bibliothèque Firebase : API REST (`fetch`, PATCH à chemins multiples) et flux SSE (`EventSource`).
- Données : `/adp/<salon>/h` (contrôle chiffré), `/adp/<salon>/d/<fiche>/<haché du champ>` = `{t, c}`, `/adp/<salon>/m/<fiche>` (méta ou suppression), `/adp/<salon>/p/<appareil>` (présence, rafraîchie toutes les 20 s).
  `t` = heure de la **modification** du champ (heure locale corrigée de l'écart avec l'horloge du serveur) ; `c` = IV (12 octets) + texte chiffré de `{f : fiche, k : champ, v : valeur}` (ou `x : 1` pour un champ effacé), en base64url. Un nœud dont le contenu ne correspond pas à son emplacement est ignoré.
- Fusion : pour chaque champ, `ft` = heure de la valeur de l'appareil, `ct` = heure du champ dans la base. Valeurs différentes : la plus récente l'emporte, **à égalité la base** ; un champ n'est envoyé que si `ft > ct` ; l'absence d'un champ ici ne supprime rien ailleurs sans suppression explicite.
- Essais : un émulateur local de Firebase (`fbemu.py`) et deux instances d'Edge sans fenêtre (`run_fb.py`, boîte à outils de test) ; jamais contre la vraie base.
