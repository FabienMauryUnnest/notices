# 📒 Notice — Pipeline attribution Alterna

## En résumé, que fait ce pipeline ?

Ce pipeline reconstruit le **parcours marketing complet** de chaque visiteur qui finit par souscrire un contrat Alterna. L'objectif : identifier quelles campagnes publicitaires (Google, Meta, Bing…) ont contribué à la souscription, même si le visiteur est revenu plusieurs fois sur le site par des chemins différents avant de convertir.


## Le problème qu'on résout

Un visiteur type ne souscrit pas en un seul clic. Il peut :

1. Cliquer sur une pub Google "marque pure" un lundi
2. Revenir via une recherche naturelle (SEO) le mercredi
3. Revenir en tapant directement l'URL (trafic direct) le vendredi
4. Souscrire le vendredi

Sans ce pipeline, Piano Analytics (notre outil de collecte) n'attribue la souscription qu'au **dernier canal** — ici "trafic direct". La pub Google qui a initié le parcours serait invisible. Ce pipeline corrige ça en reconstruisant tout le parcours.


## Comment ça fonctionne ?

Le pipeline s'appuie sur une série de tables intermédiaires dans BigQuery, recalculées quotidiennement via Dataform. Voici ce qu'elles font, regroupées par grande fonction.

### 1. Nettoyage et préparation des données brutes

On récupère les données envoyées par Piano Analytics — visites, souscriptions, clics publicitaires — et on les nettoie en plusieurs passes :

- **Dédoublonnage technique** : Piano peut envoyer plusieurs fois le même événement (rechargement de page, synchronisation…). On élimine les doublons exacts pour ne garder qu'un seul exemplaire.
- **Exclusion des agents commerciaux** : certains visiteurs sont en réalité des agents Alterna qui souscrivent pour le compte de clients. On les identifie (via le marqueur `funnel_expert` ou un seuil de 4+ souscriptions converties) et on neutralise leurs données marketing pour ne pas fausser l'analyse.
- **Normalisation des sources** : les noms de sources marketing sont harmonisés. Par exemple, si un `fbclid` (identifiant de clic Facebook) est présent mais la source est vide, on attribue "Facebook". Les pages blog sont identifiées, les visites converties normalisées.

### 2. Construction des touchpoints

Chaque visite nettoyée est transformée en **touchpoint marketing** — un moment précis où un visiteur a été en contact avec un canal. Par exemple : "Le 15 mars à 14h32, le visiteur X a cliqué sur la campagne Google 'marque-pure'."

On crée deux visions complémentaires :

- **Par visite** : les touchpoints rattachés à une session de navigation spécifique
- **Par visiteur** : les touchpoints rattachés à un individu, regroupant potentiellement plusieurs sessions sur la même journée

Chaque touchpoint reçoit un identifiant unique stable pour éviter les doublons dans les étapes suivantes.

### 3. Raccordement d'identité (Identity Stitching)

C'est le cœur du pipeline. Un même visiteur peut interagir avec le site Alterna via différents appareils, navigateurs, ou sessions. Piano lui attribue à chaque fois un identifiant différent. Le raccordement d'identité consiste à **relier ces identifiants entre eux** pour reconstituer le parcours complet d'une seule personne.

On utilise quatre clés de raccordement, par ordre de fiabilité :

1. **Email haché** — le plus fiable, résiste au changement d'appareil
2. **Identifiant utilisateur** — fiable, lié au compte connecté
3. **Identifiant visiteur** — le cookie navigateur, limité à un seul navigateur
4. **Identifiant de visite** — la session, le plus éphémère

Pour chaque souscription, on collecte toutes les identités connues et on les utilise pour retrouver les touchpoints marketing associés, même ceux captés sur un autre appareil ou un autre navigateur.

### 4. Attribution et flags de canal

C'est l'étape finale qui alimente le dashboard Looker Studio. Pour chaque souscription, on regarde **en arrière sur 30 jours** quels canaux marketing ont été vus en combinant les touchpoints de visite ET de visiteur grâce aux identités résolues.

On obtient pour chaque souscription :

- **La source finale Piano** : le dernier canal avant la souscription (ex : "Direct traffic")
- **Les flags de canal** : pour chaque canal (Google, Meta, Bing, SEO, Direct, Blog), est-ce qu'il est intervenu dans les 30 jours précédant la souscription ?
- **Le nombre de visites par canal** : combien de fois le visiteur a interagi avec chaque canal
- **Le délai depuis le dernier contact** : combien de jours entre le dernier touchpoint de ce canal et la souscription
- **Le consentement** : le visiteur avait-il consenti au tracking Google Ads / Meta Pixel ?

C'est ce qui permet de dire : **"Cette souscription attribuée au trafic direct a en fait été initiée par une campagne Google 5 jours plus tôt."**



## Concepts clés

### Source finale vs. canal contributeur

- **Source finale** : le dernier canal vu par Piano au moment de la souscription (ex : "Direct traffic")
- **Canal contributeur** : un canal qui a participé au parcours sans être le dernier (ex : Google Ads 5 jours avant)

Le dashboard montre les deux : la source finale que verrait Piano seul, et les canaux contributeurs identifiés par le pipeline d'attribution.

### Fenêtre d'attribution

On regarde **30 jours en arrière** à partir de la date de souscription pour déterminer quels canaux ont contribué. Un touchpoint Google vu il y a 45 jours ne sera pas comptabilisé.

### Flag de canal ("Google dans le parcours")

Un flag comme **"Google dans le parcours = OUI"** signifie qu'au moins un touchpoint Google a été identifié dans les 30 jours précédant la souscription, quelle que soit la source finale.
