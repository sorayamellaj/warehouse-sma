# Étape 1 : Modélisation théorique et mathématique

## 1. Modèle de l'entrepôt

L'entrepôt est un graphe $G = (V, E)$ :
- $V$ : les nœuds (rayonnages, postes de préparation, stations de recharge, intersections)
- $E = S$ : les segments d'allée (arêtes). Un segment accueille un seul robot à la fois.

Temps discret : $t \in \{1, \dots, T\}$. Traverser un segment dure 1 créneau.

Ensembles :
- $R$ : robots ($N = |R|$), $O$ : commandes, $S$ : segments, $C$ : stations de recharge
- $k$ : nombre de places de recharge simultanées

Variables de décision :
- $y_{r,s,t} \in \{0,1\}$ : vaut 1 si le robot $r$ occupe le segment $s$ au créneau $t$
- $z_{r,t} \in \{0,1\}$ : vaut 1 si le robot $r$ se recharge au créneau $t$

Exemple minimal : $v_0 \overset{s_1}{—} v_1 \overset{s_2}{—} v_2 \overset{s_3}{—} v_3$, avec $s_4$ reliant $v_2$ à $v_4$.
$v_0$ = rayonnage, $v_3$ = poste de préparation, $v_4$ = station de recharge.
Deux robots qui traversent en sens inverse se rencontrent sur $s_2$ : c'est le cas d'interblocage.

## 2. Agents et actions

| Agent | Rôle | Actions possibles |
|---|---|---|
| `AGVAgent` | Transporte une commande, gère sa batterie | choisir un itinéraire (suite de segments), demander la réservation d'un couple (segment, créneau), demander un créneau de recharge, attendre, avancer |
| `OrderAgent` | Représente une commande (origine, destination, priorité) | publier la commande, choisir un robot parmi les propositions, fixer la priorité (normale ou express) |
| `ChargingStationAgent` | Gère les places et la file d'attente d'une station | annoncer une place libre, attribuer la place à l'offre la plus forte, mettre à jour la file |
| `AisleAgent` | Gère les réservations d'un segment | accepter ou refuser une demande $(s,t)$, libérer un créneau, publier la table de réservation |

Chaque agent est un agent à utilité : il choisit l'action qui maximise sa fonction d'utilité.

## 3. Fonction d'utilité

Pour le robot $r$ :

$$U_r = 100 - \alpha\, T^r_{\text{livraison}} - \beta\, T^r_{\text{attente}} - \gamma \max(0,\, B_{\min} - B_r)$$

- $T^r_{\text{livraison}}$ : durée du trajet jusqu'au poste de préparation (sans l'attente)
- $T^r_{\text{attente}}$ : temps passé à attendre (allée occupée, file de recharge)
- $B_r$ : batterie prévue du robot à la fin de l'action, $B_{\min}$ : seuil minimal
- $\alpha, \beta, \gamma \ge 0$ : poids des trois critères

Règle de décision :

$$A_t = \arg\max_{a \in \mathcal{A}_r} U_r(a)$$

Rôle de la base de connaissances (KB) : elle fournit le plan de l'entrepôt (graphe), les priorités de commande, les zones partagées avec des humains et les paramètres de l'utilité.
Exemples : commande express $\Rightarrow$ $\alpha$ plus élevé ; segment en zone humaine $\Rightarrow$ interdit ou pénalisé ; $B_{\min}$ lu dans la KB.

## 4. Contraintes couplantes

Exclusivité des allées (au plus un robot par segment et par créneau) :

$$\sum_r y_{r,s,t} \le 1 \qquad \forall s, t$$

Capacité de recharge (au plus $k$ robots en charge en même temps) :

$$\sum_r z_{r,t} \le k \qquad \forall t$$

Elles sont dites « couplantes » car elles relient les décisions des robots entre eux.
Contrainte individuelle ajoutée (choix de modélisation) : un robot ne roule pas et ne se recharge pas en même temps :

$$\sum_s y_{r,s,t} + z_{r,t} \le 1 \qquad \forall r, t$$

## 5. Jeu de congestion, prix de l'anarchie et consensus

### 5.1 Le jeu
- Joueurs : les robots $R$
- Ressources : les couples (segment, créneau) $(s,t)$ et les places de recharge $(c,t)$
- Stratégie $a_r$ : l'ensemble des ressources utilisées par $r$ (itinéraire, créneaux, recharge éventuelle)
- Gains : $U_r(a_r, a_{-r})$, définie en section 3
- Congestion : si $n_e(a)$ robots visent la ressource $e$, le retard est $d_e(n) = \tau\,(n-1)$ ($\tau$ = durée d'un créneau), donc $T^r_{\text{attente}}(a) = \sum_{e \in a_r} d_e(n_e(a))$

Sans coordination, la contrainte est relâchée en coût de congestion. Le protocole de réservation (5.3) impose ensuite $n_e \le 1$.

C'est un jeu de congestion (Rosenthal, 1973) : il admet un potentiel exact

$$\Phi(a) = 100N - \sum_r \big(\alpha T^r_{\text{livraison}} + \gamma \max(0, B_{\min} - B_r)\big) - \beta \sum_e \sum_{j=1}^{n_e(a)} d_e(j)$$

Un changement de stratégie d'un seul joueur modifie $\Phi$ exactement comme son utilité. Donc la dynamique de meilleure réponse converge vers un équilibre de Nash pur.

### 5.2 Équilibre de Nash et prix de l'anarchie
Équilibre de Nash $a^*$ : pour tout $r$ et toute stratégie $a'_r$, $U_r(a^*_r, a^*_{-r}) \ge U_r(a'_r, a^*_{-r})$.

Bien-être social : $W(a) = \sum_i U_i(a)$. Optimum social : $\max_a W(a)$ sous les contraintes de la section 4.

$$\mathrm{PoA} = \frac{\max_a \sum_i U_i(a)}{\sum_i U_i(a_{\mathrm{Nash}})} \ge 1$$

S'il y a plusieurs équilibres, on prend le pire. Le « 100 » de l'utilité garde les valeurs positives, ce qui rend le rapport interprétable.

### 5.3 Algorithme de consensus (réservation distribuée par meilleure réponse)
1. Chaque `AGVAgent` lit la table des réservations (publiée par les `AisleAgent`) et calcule sa meilleure réponse $a'_r$.
2. Si $U_r(a'_r) > U_r(a_r)$, il envoie une demande de réservation pour chaque $(s,t)$ de $a'_r$.
3. Chaque `AisleAgent` accepte si le créneau est libre (en cas d'égalité : priorité de la commande, puis identifiant) et publie la table mise à jour.
4. Si toutes les demandes sont acceptées, $r$ libère ses anciens créneaux. Sinon il garde son ancien plan.
5. On répète, un robot à la fois (tour de rôle). On s'arrête quand aucun robot ne s'améliore (équilibre) ou quand le délai maximal est dépassé.

Convergence : chaque amélioration stricte augmente $\Phi$ et le nombre de stratégies est fini, donc l'algorithme s'arrête.
Panne : si un robot tombe (message LWT), les `AisleAgent` libèrent ses créneaux et les autres recalculent.

### 5.4 Stations de recharge (Contract Net Protocol)
La `ChargingStationAgent` (manager) annonce une place libre. Les robots (contractors) répondent par une offre liée à leur urgence de batterie (par exemple $\gamma \max(0, B_{\min} - B_r)$). La station attribue la place à l'offre la plus forte.