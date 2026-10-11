# Conversion de coordonnées géodésiques ⇄ cartésiennes

Petit programme Python (tkinter) qui convertit une position entre deux écritures, dans les deux sens, et la dessine sur un schéma 3D et sur un globe.

| Écriture | Ce que c'est |
|---|---|
| **φ, λ, h** (géodésique) | latitude, longitude, hauteur au-dessus de l'ellipsoïde |
| **X, Y, Z** (cartésienne) | la même position, en mètres, à partir du centre de la Terre |

- **→** convertit (φ, λ, h) en (X, Y, Z)
- **←** convertit (X, Y, Z) en (φ, λ, h)

Chaque conversion ajoute un point sur le globe, qu'on peut faire tourner à la souris.

## Lancer le programme

Il faut Python 3 avec tkinter (inclus dans l'installation standard sur Windows et macOS).

```bash
python conversion_geodesique.py            # thème par défaut (clair)
python conversion_geodesique.py sombre     # autres thèmes : clair, sombre, foret, violet, chaud
```

## Utilisation

1. Choisir l'ellipsoïde en haut (Clarke 1880 par défaut, celui du système Merchich au Maroc). Avec « Personnalisé », on tape soi-même a et b.
2. Remplir un côté de la carte, puis cliquer sur la flèche qui va vers l'autre côté (ou appuyer sur Entrée).
3. Les angles se tapent en degrés décimaux (`35.7595`) ou en degrés-minutes-secondes (`35 45 34.2 N`, `5°50'01"W`).
4. Pour ←, choisir la méthode **Directe** ou **Itérative** (même résultat, deux façons d'y arriver).
5. Le panneau de droite propose deux vues : **Schéma géodésique** et **Terre**. Cliquer sur une ligne de la liste tourne la vue vers ce point.

## Les principes du calcul

### 1. L'ellipsoïde

La Terre est un peu aplatie aux pôles. On la modélise par un ellipsoïde de demi-grand axe **a** (équateur) et de demi-petit axe **b** (pôles) :

```
e² = (a² − b²) / a²                 excentricité au carré (0 = sphère parfaite)
N  = a / √(1 − e² · sin²φ)          rayon de courbure du premier vertical
```

N est la distance entre un point de la surface et l'axe des pôles, mesurée le long de la verticale du lieu.

### 2. Conversion → : (φ, λ, h) vers (X, Y, Z)

Formule directe, sans boucle :

```
X = (N + h) · cos φ · cos λ
Y = (N + h) · cos φ · sin λ
Z = (N · (1 − e²) + h) · sin φ
```

### 3. Conversion ← : (X, Y, Z) vers (φ, λ, h)

La longitude est simple : `λ = atan2(Y, X)`. La difficulté est que φ et h dépendent l'un de l'autre. Deux solutions sont proposées.

**Méthode directe (formule fermée)**, avec `p = √(X² + Y²)`, `r = √(X² + Y² + Z²)` et `f = (a − b) / a` :

```
µ = atan( Z/p · ((1 − f) + a·e²/r) )
φ = atan( (Z·(1 − f) + e²·a·sin³µ) / ((1 − f) · (p − e²·a·cos³µ)) )
h = p·cos φ + Z·sin φ − a·√(1 − e²·sin²φ)
```

**Méthode itérative** : on part d'une latitude approchée `φ = atan2(Z, p)`, puis on répète jusqu'à stabilisation (4 ou 5 tours suffisent) :

```
φ = atan( (Z/p) / (1 − e²·N / (N + h)) )
N = a / √(1 − e²·sin²φ)
h = p / cos φ − N
```

### 4. Cas particulier

Si X = Y = 0, le point est exactement sur l'axe des pôles et la longitude n'a plus de sens. Le programme refuse ce cas avec un message.

## Les principes de l'affichage

- **Radians en interne, degrés à l'écran.** Les formules utilisent des radians, l'utilisateur voit des degrés.
- **Schéma géodésique.** Pour un point P, il montre l'ellipsoïde, les axes X Y Z, la normale, les angles φ et λ, la hauteur h et les projections de P sur les axes. L'aplatissement est exagéré ×40 pour être visible. Seul le dessin est exagéré, les nombres affichés sont exacts.
- **Point intéressant du schéma.** La normale ne passe pas par le centre de la Terre : elle coupe l'axe des pôles à `z = −N·e²·sin φ`, et c'est de ce point que φ est mesurée. La latitude géocentrique ψ (angle du rayon du centre à P) est un peu différente de φ.
- **Globe 3D.** Les continents sont des contours simplifiés, transformés en points régulièrement espacés. Chaque point est tourné, incliné, puis projeté à l'écran. La profondeur dit s'il est devant ou derrière.
- **Animation.** Une boucle (environ 30 images par seconde) met à jour la rotation et redessine tout.
- **Couleurs.** Cinq thèmes (`sombre`, `clair`, `foret`, `violet`, `chaud`) donnent une couleur à chaque élément. L'océan est en dégradé, les continents sont colorés par type de terrain (glace, désert, forêt...).

## Organisation du code

Le fichier `conversion_geodesique.py` est commenté pas à pas et se lit de haut en bas :

| Partie | Contenu |
|---|---|
| 1. Les calculs | ellipsoïdes, `e2_of`, `N_of`, `geo_to_cart`, `cart_to_geo_directe`, `cart_to_geo_iterative`, `parse_angle`, `dms` |
| 2. Le globe | contours des continents, `_inside`, `land_cat`, `build_land_dots` |
| 3. Les couleurs | thèmes, `blend`, couleurs des continents et de l'océan |
| 4. La fenêtre | classe `App` : interface, logique (`to_xyz`, `to_geo`), dessin (`draw_schema`, `draw_terre`), animation (`tick`) |

## Limites

- Les contours des continents sont dessinés à la main et très approximatifs : ils servent à se repérer, pas à mesurer.
- Les ellipsoïdes fournis sont ceux de la liste du programme ; pour un autre, utiliser « Personnalisé ».
- Le programme ne change pas de système de coordonnées (par exemple de Merchich vers WGS 84) : il convertit seulement entre deux écritures du même système.
