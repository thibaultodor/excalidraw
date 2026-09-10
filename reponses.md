# TD DORA — réponses et résultats

Mesures effectuées le **10 septembre 2026**. Le contrat a été figé à
`2026-09-10T16:18:55+02:00`, avant la première collecte automatisée. Les
valeurs sont datées : Excalidraw étant actif, elles ne doivent pas être
comparées chiffre pour chiffre avec celles du guide relevées le 6 septembre.

## Phase 0 — contrat de définitions

Le fichier de référence versionné est [`dora-definitions.yml`](dora-definitions.yml).
Son contenu utile est reproduit ici pour que le rendu soit autonome :

```yaml
application: excalidraw
fige_le: "2026-09-10T16:18:55+02:00"

deploiement:
  compte_comme_deploiement: "premier Deployment Status 'success' sur l'environnement exact 'Production \u2013 excalidraw'"
  exclut: ["environnements Preview", "environnements Production ou Preview des exemples d'integration", "deploiements sans statut success"]
  horodatage: "fin du deploiement (created_at du premier statut success)"

changement:
  point_de_depart: "committer date du commit sur la branche par defaut"

incident:
  definition: "degradation en production necessitant une intervention immediate ; a defaut d'acces a la production, une issue 'bug' n'est qu'un proxy exploratoire"
  source: "incident reel : systeme d'alerte ou issue dediee ; proxy du TD : issue GitHub avec le label 'bug'"
  debut: "horodatage de la detection ; created_at de l'issue uniquement pour le proxy"
  fin: "service retabli ; closed_at de l'issue uniquement pour le proxy"
  rattachement_deploiement: "champ explicite 'caused_by: <deployment_id>' renseigne dans le corps de l'incident"

rework:
  marqueur: "branche hotfix/*"

fenetre_de_reference: "90 jours glissants"
agregation: "mediane (P50) pour les durees, P90 publie en complement ; comptes et taux sur la fenetre"
```

La notation YAML `\u2013` représente bien le tiret demi-cadratin de
`Production – excalidraw`, sans risque de le confondre avec un trait d'union.

## Phase 1 — reconnaissance

### 1. Environnements rencontrés

L'échantillon des 100 déploiements les plus récents contient **sept** valeurs,
et non les six observées quatre jours plus tôt dans le guide :

| Environnement | Nombre dans les 100 derniers |
|---|---:|
| `Preview – excalidraw` | 81 |
| `Preview – excalidraw-package-example` | 2 |
| `Preview – excalidraw-package-example-with-nextjs` | 2 |
| `Production – docs` | 1 |
| `Production – excalidraw` | 2 |
| `Production – excalidraw-package-example` | 6 |
| `Production – excalidraw-package-example-with-nextjs` | 6 |

Cette observation a été faite le 10 septembre 2026. L'apparition de
`Production – docs` explique l'écart avec le guide daté du 6 septembre.

### 2. Déploiements DORA retenus

Seul `Production – excalidraw` représente la mise en production de
l'application mesurée. Les trois environnements `Preview` sont de la
préproduction. `Production – docs` mesure une autre application et quatre
environnements concernent les exemples de package (deux Preview et deux
Production). Ces catégories se recouvrent donc pour deux lignes. Les agréger
violerait le périmètre « une application ou un service ».

### 3. Surestimation de la fréquence

Dans cet échantillon, compter tout donnerait 100 évènements au lieu de 2 :
`100 / 2 = 50`. La deployment frequency serait donc surestimée d'environ
**50 fois**. Ce rapport est un instantané, pas un coefficient stable.

### 4. Erreur correspondante du support

Il s'agit de la ligne « **Compter les builds comme des déploiements** », dont
le symptôme est une fréquence artificiellement élevée et dont la correction
est de filtrer strictement l'environnement de production. Ici, les évènements
indésirables sont surtout des previews et les productions d'autres
applications.

### 5. Statuts d'un déploiement

Le tableau contient l'historique des états du déploiement (`pending`,
`in_progress`, `success`, `failure`, etc.), chacun avec son propre
`created_at`, ainsi que les métadonnées et éventuelles URL associées. Un objet
Deployment exprime une intention ou une tentative ; sans statut `success`, on
ne sait pas si le changement est réellement arrivé en production.

### 6. Horodatage retenu

Je retiens le `created_at` du **premier statut `success`**, c'est-à-dire la fin
réussie du déploiement, et non le `created_at` de l'objet Deployment. Le
contrat de phase 0 avait bien figé cette convention.

## Phase 2 — collecte outillée

Commande utilisée depuis la racine du dépôt pédagogique :

```powershell
py -B .\TD\outils\dora_metrics.py --repo excalidraw/excalidraw --git ..\excalidraw --environment "Production – excalidraw" --window 90 --incident-label bug --json
```

Résultats sur la fenêtre commençant le **12 juin 2026** :

| Métrique | Valeur obtenue |
|---|---:|
| Deployment frequency | **0,1556/jour**, soit 14 déploiements en 90 jours |
| Délai médian entre deux déploiements | **118,24 h**, soit 4,93 jours |
| Change lead time P50 | **43,84 h**, soit 1,83 jour |
| Change lead time P90 | **214,76 h**, soit 8,95 jours |
| Commits analysés / nombre de lots | **75 / 13**, soit 5,77 commits par lot |

Aucun lot n'a été ignoré pour cause de SHA absent.

### 7. Taille moyenne des lots

`75 / 13 = 5,77` commits par lot. C'est un lot de quelques commits, donc un
ordre de grandeur plutôt réduit. Le chapitre 6.1 présente le travail par
petits lots comme le levier technique le plus rentable sur les cinq
métriques ; le lead time et la fréquence sont particulièrement sensibles à
cette taille.

### 8. Écart P50/P90

`214,755 / 43,836 = 4,90`. Le cas défavorable est donc presque cinq fois plus
long que le cas habituel. Cela signale une longue traîne, probablement une
catégorie de changements qui reste bloquée (validation, migration,
dépendance inter-équipe, gros lot). Cela **ne signale pas** que tous les
changements ni que le cas typique se sont dégradés.

### 9. Ordre de grandeur de la deployment frequency

Quatorze déploiements en 90 jours correspondent à environ 4,7 déploiements
par mois : un rythme de l'ordre de l'hebdomadaire, et non un déploiement à la
demande comme l'ordre de grandeur *elite* de 2024. Il serait toutefois
incorrect de classer l'équipe à partir de ce seul chiffre : les clusters sont
des repères mouvants issus d'une enquête par tranches, et nos données
instrumentées ne lui sont pas commensurables au chiffre près.

### 10. Intérêt du délai médian

Le ratio `14 / 90` dépend fortement des limites arbitraires de la fenêtre. Le
délai médian de 4,93 jours exprime directement l'expérience habituelle (« un
déploiement tous les cinq jours environ »), résiste mieux aux intervalles
extrêmes et reste lisible pour une équipe qui déploie peu.

### 11. Métriques indisponibles

Les trois valeurs `n/a` sont :

- le failed deployment recovery time ;
- le change fail rate ;
- le deployment rework rate.

Elles nécessitent toutes une information opérationnelle absente ou non
fiable dans le dépôt public : incidents de production, rattachement causal au
déploiement et marqueur explicite de retravail.

### 12. Maillon faible

Le maillon faible est le **lien entre un incident et le déploiement qui l'a
causé**. Git fournit les commits et l'API fournit les déploiements, mais ni un
outil ni une corrélation temporelle ne peut inventer une causalité qui n'a pas
été saisie. Sans ce lien, change fail rate et recovery time ne sont pas
calculables correctement.

### 13. Règle erronée des 24 heures

Faux positif : deux versions planifiées peuvent être livrées à quelques
heures d'intervalle sans que la première ait échoué. Faux négatif : un incident
peut être détecté tard, réparé après plus de 24 h, ou résolu par une action qui
ne crée aucun nouveau déploiement. La proximité temporelle n'établit ni
l'échec ni la causalité.

## Phase 3 — proxy `bug` et limites

La vérification a utilisé les recherches GitHub avec la date du 12 juin 2026.
Le collecteur a aussi été corrigé : le paramètre `since` de l'API Issues filtre
les tickets **mis à jour**, pas ceux **créés**. La version initiale comptait
ainsi 23 anciens bugs récemment modifiés ; la requête Search et le filtre
local utilisent désormais `created_at`, conformément au sujet.

### 14. Issues `bug` trouvées et rattachées

Le collecteur corrigé trouve **0 issue `bug` créée dans la fenêtre** et donc
**0 issue rattachée** à l'un des 14 déploiements. Le change fail rate reste à
`n/a` : zéro rattachement ne prouve pas zéro incident.

### 15. Compteurs de l'interface GitHub

- toutes catégories créées depuis le 12 juin 2026 : **144** (122 ouvertes et
  22 fermées au moment du relevé) ;
- portant le label `bug` depuis la création du dépôt : **765** ;
- portant le label `bug` et créées depuis le 12 juin 2026 : **0**.

### 16. Confrontation des nombres

Le dépôt reste très actif (144 nouvelles issues) et possède un important
historique de tickets `bug` (765), mais aucun nouveau ticket n'a reçu ce label
pendant la fenêtre. La taxonomie ou le processus de tri a donc changé, ou le
label a cessé d'être appliqué. Le zéro mesure l'abandon du marqueur, pas
l'absence certaine de défauts.

### 17. Quatrième explication à un taux de 0 %

Outre le faible nombre de déploiements, une détection insuffisante et
l'absence de rattachement, une quatrième explication est : **le proxy ou le
marqueur choisi n'est plus alimenté** (label abandonné, renommé ou remplacé
par un autre système).

### 18. Métrique la plus sensible à la saisie

Le support désigne le **deployment rework rate** comme le plus sensible à la
discipline de saisie, puisqu'il faut marquer chaque correctif non planifié.
Le parallèle n'est pas fortuit : un label d'incident ou de bug est lui aussi
un champ conventionnel ; quand la convention n'est plus appliquée, le calcul
reste techniquement valide mais ne mesure plus la réalité souhaitée.

## Phase 4 — production de la donnée manquante

Le workflow prêt à copier sur le fork est fourni dans
[`.github/workflows/deploy.yml`](.github/workflows/deploy.yml). Il crée exactement un évènement de
déploiement, enregistre dans `payload.source_ref` la branche de la pull
request fusionnée (ou le nom `hotfix/*` trouvé dans un message de fusion
local) et publie le statut `success`. Une fusion fast-forward reste détectable
grâce au préfixe conventionnel `hotfix:` demandé par le guide. Le collecteur
reconnaît ce champ pour que le retravail reste mesurable après la fusion sur
`master`.

Cette adaptation corrige une contradiction du guide : avec `ref: context.sha`,
la valeur API est un SHA et ne peut jamais commencer par `hotfix/`. De même,
`NOTES.md` étant un nouveau fichier, la séquence correcte est `git add
NOTES.md`, puis `git commit`, et non `git commit -am` seul.

La phase a été exécutée sur le fork
[`thibaultodor/excalidraw`](https://github.com/thibaultodor/excalidraw). Le
clone local utilise maintenant ce fork comme `origin` et l'amont comme
`upstream`. Cinq pushes sur `master` ont produit cinq statuts `success`, dont
le dernier provient de `hotfix/correctif-urgent`. L'[incident
no 1](https://github.com/thibaultodor/excalidraw/issues/1) contient
`caused_by: 6374499592` et a été clos 140 secondes après son ouverture.

Commande de mesure finale :

```powershell
py -B .\TD\outils\dora_metrics.py --repo thibaultodor/excalidraw --git ..\excalidraw --environment production --window 90 --incident-label incident --rework-prefix "hotfix/" --json
```

| Métrique du fork | Valeur mesurée le 10 septembre 2026 à 14:59 UTC |
|---|---:|
| Deployment frequency | **0,0556/jour**, soit 5 déploiements en 90 jours |
| Délai médian entre deux déploiements | **0,01139 h**, soit 41 s |
| Change lead time P50 | **0,00306 h**, soit 11 s |
| Change lead time P90 | **0,00333 h**, soit 12 s |
| Commits analysés / lots | **5 / 4** |
| Change fail rate | **20 %** (1 déploiement défaillant sur 5) |
| Failed deployment recovery time P50 | **0,03889 h**, soit 2 min 20 s |
| Deployment rework rate | **20 %** (1 hotfix sur 5 déploiements) |
| Incidents trouvés / rattachés | **1 / 1** |

### 19. Nouveaux calculs rendus possibles

Une fois le workflow, le marqueur `hotfix/*` et le champ `caused_by` produits,
les trois valeurs manquantes deviennent calculables : change fail rate,
failed deployment recovery time et deployment rework rate. Les cinq métriques
DORA peuvent alors être publiées ensemble.

### 20. Coût de production de la donnée

Le guide réserve environ **50 minutes** à l'instrumentation contre **40
minutes** à la tentative par proxy. Ici, l'exécution automatisée entre le
premier déploiement et la clôture de l'incident a pris environ **6 min 25 s**,
hors analyse et rédaction. Ce temps court ne représente pas l'effort manuel
d'une équipe ; l'enseignement reste que l'instrumentation coûte au démarrage,
mais crée ensuite un flux causal réutilisable, contrairement au proxy.

### 21. Représentativité du change fail rate

Non. Un incident artificiel parmi cinq déploiements donne 20 %, mais un seul
évènement fait varier le taux de 20 points. Il faut une fenêtre contenant un
volume suffisant de déploiements et d'incidents réels, une convention stable,
un rattachement exhaustif et plusieurs périodes comparables avant d'en tirer
une tendance.

## Phase 5 — lecture critique des outils

État relevé le 10 septembre 2026 sur les dépôts et flux de releases GitHub :

| Outil | Dernière release | Dernier commit | État |
|---|---|---|---|
| Apache DevLake | `v1.0.3-beta17`, 5 septembre 2026 | `43b5728`, 7 septembre 2026 | actif |
| Middleware | `0.3.1`, 30 mai 2025 | `844eb42`, 3 août 2026 | dépôt actif, releases en retard |
| Four Keys | `v1.0.2`, 4 mai 2023 | `6cc642e`, 23 janvier 2024 | archivé et en lecture seule depuis le 23 janvier 2024 |

### 22. Un outil professionnel calculerait-il le change fail rate ?

Seulement s'il reçoit des évènements d'incident et un rattachement fiable aux
déploiements (champ causal, incident manager, convention de labels, etc.). Il
peut ingérer, normaliser et agréger cette donnée, mais pas déduire la causalité
à partir des seuls commits et déploiements publics. La conclusion de la phase
2 ne change donc pas avec un produit professionnel.

### 23. Pourquoi l'outillage arrive en dernier

Le Quick Check établit rapidement une ligne de base et la conversation
d'équipe identifie la contrainte utile ainsi que les définitions communes.
Sans ce travail, on automatise un proxy ambigu et on obtient plus vite un
chiffre faux. L'instrumentation vient ensuite, lorsque l'équipe sait quelle
donnée produire, où la capter et quelle décision elle doit éclairer.

### 24. Habitude à adopter avant de choisir un outil

Toujours vérifier la maintenance réelle : date du dernier commit, date de la
dernière release, statut archivé, issues et pull requests, documentation,
compatibilité et sécurité des dépendances. Il faut aussi réaliser un petit
prototype avant de s'engager. Un tutoriel populaire peut continuer à citer un
outil plusieurs années après son abandon, comme Four Keys.

## Éléments pour la restitution collective

La ligne du contrat qui explique le plus directement un écart avec un autre
binôme est :

```yaml
compte_comme_deploiement: "premier Deployment Status 'success' sur l'environnement exact 'Production \u2013 excalidraw'"
```

Un binôme qui compte les previews, les exemples, la documentation ou le
`created_at` de l'objet Deployment obtient nécessairement une autre série.

Après observation des données, je ne modifierais pas rétroactivement le
contrat figé. Pour la prochaine série, je remplacerais le proxy `bug` par un
label obligatoire `incident`, avec les vrais instants de détection et de
rétablissement, et je conserverais `payload.source_ref` pour le hotfix.

Deux équipes mesurant le même dépôt peuvent donc différer parce que leur
périmètre, leur horodatage, leur fenêtre ou leur convention d'incident ne sont
pas identiques. Leurs séries ne doivent pas être agrégées. Face au P90 élevé,
ma première action serait d'identifier les commits de la longue traîne et de
cartographier leur flux ; je demanderais d'abord s'ils partagent une même
catégorie de blocage. Enfin, ces données publiques décrivent une partie du
système de livraison d'Excalidraw, pas la productivité ni la santé des
personnes qui le maintiennent.


