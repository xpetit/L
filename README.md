⚠️ ne pas trop tenir compte de la syntaxe des exemples dans ce doc, faudra la définir, c'est plus du pseudo-code inconsistant pour le moment.

# objectives

Là faudrait qu'on se mettre d'accord, après avoir lu des critiques des langages actuels par quelques devs qui me semblent honnêtes, les langages sont encore trop complexes.
Si on fait un langage simple dont la spec est clairement définie, ça veut dire qu'on va encourager plusieurs implémentations et innovations (comme ça a été le cas en C).
Pour ça il faut éviter d'avoir beaucoup de features haut niveau. Dans cette optique je suggère notamment qu'on définisse en dernier tout ce qui est code concurrent/asynchrone. Parce que c'est complexe à implémenter/appréhender et ça peut se rajouter plus tard assez facilement en réutilisant les structures de base.

Donc est-ce qu'on est ok pour viser d'abord la simplicité ? Il y a peu de langages simples qui ont été conçus avec le recul sur un demi-siècle de complexité croissante

# types

## number

Avoir un type de base : nombre à virgule fixe de taille arbitraire
On pourrait choisir la précision lors de la déclaration
Exemples :

```
a = 0
b = 7
c = 0.1
π = 3.141592653589793238462643383279502884197169399375105820974944592307816406286
b * c == 0.7 // true
```

C'est un mélange de deux idées qui IMO rendent le dev et les implémentations soft & hard plus simples.

### fixed-point number

Le fixed-point permet d'avoir une précision absolue, c'est grosso modo comme utiliser un int en rajoutant une virgule quelque part au milieu. Ça permet d'éviter : `7 * 0.1 != 0.7`

Il y a de la littérature sur fixed vs float, et des raisons (plus historiques que techniques) sur pourquoi les float sont devenus privilégiés. Quelques liens :

https://www.microcontrollertips.com/difference-between-fixed-and-floating-point
https://en.wikipedia.org/wiki/Fixed-point_arithmetic
https://stackoverflow.com/questions/7524838/fixed-point-vs-floating-point-number

### arbitrary-precision arithmetic (big numbers)

Et le bignum évite les overflows et permet de ne pas trop se soucier de la taille du nombre, elle reste limitée par la quantité de RAM mais on est à d'autres ordres de grandeurs que des nombres 64/128 bits.

#### performance

Les calculs sur ces nombres vont moins vite mais on peut rajouter un mécanisme de limitation de taille du nombre (pour de la sûreté (bound checking, analyse statique de code) ou de la performance), et dire que implicitement une limitation de taille qui tombe sur une taille de nombre de processeur permet d'utiliser le type natif (il faut voir alors quel comportement adopter en cas d'overflow (le même que lors d'un OOM sur un bignum ?)).

```
a -128..127 // fast
b 0..2^32 - 1 // fast
```

Ou alors créer un type dédié ou une indication explicite qu'on utilise un type natif (avec la possiblité de signer l'int ?).

```
a i8
b u32
```

Mais a-t-on besoin de laisser le dév se concentrer sur les perf des nombres ? Si ça se trouve avec suffisamment d'optis côté implémentation, les nombres pourraient aller vite dans la plupart des cas donc j'attendrais que ça devienne un souci de perf avéré dans un cas concret avant de trop se concentrer là dessus. Enfin les performances de cette proposition ne m'inquiètent pas, pour un premier jet de langage il vaut mieux écrire une spec simple des features que l'on veut avoir, tout en gardant à l'esprit que maintenir un niveau de perf acceptable ne sera pas infaisable.

## array (list, sequence, vector) <= pick one

À vue de nez il nous faudrait au minimum un autre type : le tableau pour pouvoir faire une chaîne de nombre.
Si ensuite on fait les strings comme des suites de caractères, et les caractères sont des nombres (des octets), le tableau pourrait être réutilisé pour implémenter les strings.
Les tableaux pourraient aussi servir aux I/O de fonctions (arguments en entrée, valeurs de retours en sortie).

### bonus

Autre questionnements :

-   un tableau est indexé à 0 par défaut, est-ce qu'on ne permettrait pas de changer ça ?
-   puisque qu'un tableau est par définition associatif avec des entiers naturels en tant que clefs, est-ce qu'on ne mixerait pas ce concept (associative array, map, dictionnary, hashmap, key-value store, c'est la même chose essentiellement) avec le tableau ?

Il y a des similitudes entre un tableau classique et une map :

```
a = [2, 4, 8, 16]
b = {0:2, 1:4, 2:8, 3:16}
a[0] == b[0] && a[3] == b[3] // true
```

Lua l'a fait avec la notion de table. À voir ce que ça implique mais cette idée de généraliser le concept me tente bien.

## pointeur

Histoire de laisser le dev choisir s'il fait une copie ou une ref ? Si on l'implémente est-ce qu'on n'interdit pas `null` ? Je ne vois pas l'utilité de `None` / `undefined` / `null`, etc

## fonction

Indispensable de pouvoir gérer une fonction en tant que valeur n'est-ce pas ?

## symbol / enumeration

Dans la plupart des langages cela se fait avec des constantes (comme en Go), mais permet de mélanger les concepts (c'est au dev de faire gaffe ?) :

```
color [white, black] // 0, 1
theme [dark, light] // 0, 1

Colorize(dark) // valid but wrong
```

Je propose de ne pas créer de nouveau type, un enum c'est en fait un tableau de labels/identifiants (indexé par défaut à 0, mais on peut réindexer chaque élément)

### liens

https://en.wikipedia.org/wiki/Enumerated_type

## error

Est-ce qu'on ne dirait pas que c'est un label qui doit être défini qu'on throw/raise ? Avec possibilité d'ajouter de la donnée (comme ça tout est permis).

```
// ceci est un enum, donc juste un tableau d'identifiants
IO_errors [disk_full, network_unreachable, write_error]

[...]

  // error message
  raise write_error, `not written: ` & written

  // just give the data and let the caller format their own error message
  raise write_error, to_write - written
```

Ensuite le caller peut faire le tri dans l'exception qu'il choppe (en plus bien sûr de la tester directement) :

```
if error in IO_errors then
  ...
```

## bool

C'est un peu lié à la notion au dessus (enum), car dans pas mal de langages, vrai et faux sont un enum.
Est-ce qu'on fait que le true/false est une notion à part ? ou des alias vers des nombres 0 et 1 ?
Si le bool est un nombre, alors on implémenterait peut-être la notion de truthiness comme plein de langages ?

Genre :

```
a = 1
if a then
  print `100% chance to be printed`
fi
```

On pourrait même ne jamais faire apparaître `true` ou `false` dans le langage mais ce serait peut-être un peu moins clair d'utiliser `0` et `1`.
Voir => https://en.wikipedia.org/wiki/Truth_value

Si le bool est un type séparé (comme en Go / Ada), cela permet de mettre de l'implicite que sur ce type, ce code serait illégal :

```
a = 1
if a then
  ...
```

Mais celui-ci serait valide :

```
a = 1
b = a == 1
if b then
```

De mémoire en Lua ils sont radicaux au point de l'interdire, de forcer l'explicite `if b == true then`

Il y a des limites à l'explicite, permettre de ne pas mettre d'égalité pour les booléens, ça permet d'écrire du code très lisible :

```
if abort_mission then
```

On peut aussi dire que 0 est falsy et 1 est truthy comme en C / Bash ?

### liens

https://en.wikipedia.org/wiki/Boolean_data_type

## alias / déclaration

À voir si c'est indispensable mais le Go et le C permettent de faire des alias de types (compatibles entre deux, c'est du renommage, ce qui peut être cool pour la lisibilité) et des déclarations de type (types dérivés incompatibles entre deux)

## string

Alors là c'est assez compliqué, on pourrait se dire qu'on transmet juste les octets sans se préoccuper de ce que c'est, si la donnée est en UTF-8 et que le terminal le comprend, tant mieux. Mais du coup la longueur de cette chaîne : `lol 💖` est 8 et non 5... On pourrait alors se dire qu'on fait de l'UTF-32 (équivalent de la rune en Go), mais là encore il existe des graphèmes qui font plus que 4 octets : `lol 👌🏻` (qui a du mal à s'afficher dans sublime mais fonctionne sur vscode), ça fait 12 octets, mais 6 caractères d'après `wc -m`, alors qu'il n'y en a toujours que 5.
Alors je ne connais que le Swift qui se soit attaqué à ce souci en créant une abstraction de plus haut niveau qui comprend la notion de graphème si j'ai bien suivi : https://developer.apple.com/documentation/swift/character
Mais bon y a encore pas mal de caractères unicodes bizarres qu'on ne compterait pas dans la longueur de la chaîne mais qui pourtant peuvent être là : https://en.wikipedia.org/wiki/Universal_Character_Set_characters#Special-purpose_characters

On a le choix :

-   ne rien faire : `lol 💖` fait une longueur de 8, car une `string` est un tableau d'octets, et l'encodage on considère que c'est des soucis d'interfaçage, on ne peut pas facilement connaître la vraie longueur du string et on ne peut pas facilement accéder à un caractère particulier. Si on code de manière générique la lib de string, elle pourra être réutilisée quand on gérera les "grapheme clusters". Mais c'est compliqué : un graphème peut contenir plusieurs lettres dans pas mal de langues (dont le coréen) ! Ce qui donne une situation vraiment tricky : la longueur d'une string coréenne c'est le nombre de graphèmes, mais si on veut comparer/trier il faut se baser sur les lettres 🤦
-   bien faire : à la limite on peut le faire plus tard en créant un autre type `text` où on prend la chose au sérieux mais ça demande de se renseigner sur tous les langages écrits de la terre, de bien comprendre toutes les implications et la faisabilité de ça. Il faudra sûrement tester le Swift pour voir comment ils font et est-ce que ça marche même en Arabe leur type `Character`.
-   être nazi : on autorise qu'un subset de l'unicode, ça peut même aller jusqu'à interdire les emojis blacks, les signes diacritiques et les accents. Tout le monde écrit de l'anglais et comme ça tout le monde se comprend, on a un processing de texte super rapide et compact avec des caractères limités. Je ne plaisante qu'à moitié, y a énormément de caractères unicodes qui ne seront jamais utilisés (les pièces d'échec, c'est illisible tellement c'est petit : ♔♕♖♗♘♙♚♛♜♝♞♟). Cette solution devrait sûrement être laissée au choix du dev.

La première solution me tente plus dans un premier temps. Et dans tous les cas on pourrait build le type `string` sur celui de l'array

## conclusion des types

En fait je me demande si on peut ne pas tout ranger derrière deux concepts : nombre et liste

C'est un peu ça de la donnée au fond, une suite de nombres qui fait sens pour ce sur quoi il tourne (hard ou soft).

Un pointeur/une ref de fonction ? c'est une adresse donc un nombre, un enum ? c'est aussi une suite de nombres labelisés dans le code, une string ? suite de codepoints, donc nombres again. Un booléen ? Par définition 0 ou 1 c'est encore un nombre.

## weak / strong

Je tendrais vers le weak avec la possibilité de fortifier (rajouter des contraintes)

## liens

https://en.wikipedia.org/wiki/Type_system

# fonctions de base

## stdout

comme on ne refait pas encore totalement le monde informatique, on va afficher dans un terminal dans un premier temps donc faut trouver un nom de fonction (triées par ordre alphabétique)

```
display
echo
log
out
print
put
say
show
write
```

Je suis biaisé vers `put` (Ada oblige et je trouve que ça se tape vite), voilà ce que je préfère (dans l'ordre, tu peux faire une liste aussi comme ça on prend le premier en commun) :

```
put
out
log
echo
show
print
write
```

# syntaxe

C'est un sujet important j'aimerais qu'on essaie de simplifier au max (pas de `;` en fin de ligne, a-t-on vraiment besoin de parenthèses ?), qu'on privilégie les touches accessibles directement en QWERTY : /\\[]\`'=-;., toutes les lettres (pour les mots réservés, ce serait pas mal qu'il y en ait le moins possible pour permettre au dev de nommer les choses comme il veut)

Aussi si on est d'accord on peut se dire que la syntaxe est relativement stricte et qu'un outil type `gofmt` est à disposition pour aider à formater le code.

## commentaires

Un commentaire par ligne du coup ? On utilise quoi comme token pour signaler un début de commentaire (jusqu'à la fin de ligne), le classique `//` ou `--`, autre ?

## ranges

Sucre syntaxique : `1..4` génère `1, 2, 3, 4`

## in

C'est franchement trop pratique, vérifier qu'une valeur est dans un tableau, ça devrait faire partie du langage sans avoir à faire d'appel de fonction.

```
2 in 1..6 // true

c = `r`
c in `string` // true
```

## string

Si on utilisait \` pour délimiter les strings ? Ça permet d'utiliser `'` et `"` dans les strings qui reviennent plus souvent.
Donc multi-line ? Ces deux bouts de code affichent la même chose :

```
print `one
two
three
`
```

et

```
      print `one
      two
      three
      `
```

sortie :

```
one
two
three
```

Ou alors on aligne sur le début du texte plutôt que le niveau d'identation (ça me paraît bcp plus lisible) ?

```
      print `one
             two
             three
             `
```

## affectation / égalité

Là on a plusieurs solutions :

-   deux symboles différents (habituellement `=` et `==`, parfois `:=`, `=` (Ada))
-   même symbole (`=`) mais la différence est liée au contexte, si on décide de ça, la syntaxe ne doit permettre aucune ambiguïté là-dessus
-   pas de symbole pour l'affectation (comme en asm), là encore en se basant sur le contexte, on peut qu'après un espace une valeur est attendue et sera affectée.

    ```
    a 1
    b 2
    if a + b = 3 then
      print `obviously`
    fi
    ```

    ```
    robin [
      name `Robin`
      age 11
      height 1.68
      weight 51
    ]
    ```

Je serais plus pour mettre deux symboles différents, ou en tout cas d'éviter à tout prix la réutilisation du symbole (qui n'a de fondement mathématique) pour éviter de perdre les débutants qui risquent d'encore moins discerner la distinction entre affecter et tester l'égalité.

Proposition bold&dynamic : utiliser `is` pour affecter

```
address is `localhost`
port is 8080
c is a + b
lost is lives < 0
```

On interdit les multi-affectations par ligne ? Type ce genre de choses :

```
a = 1, b = 2, c = 3 // JS style
d, e, f = 4, 5, 6 // Go style

i, j = j, i
```

Obligé d'écrire :

```
a = 1
b = 2
c = 3
d = 4
e = 5
f = 6

k = i
i = j
j = k
```

50 caractères vs 54 (8% de code en plus), osef ? En plus c'est rare d'inverser des valeurs donc le joli trick `i, j = j, i` on peut s'en passer non ?

## opérateurs logiques

Je préfère `and`, `or` et `not` à `&&`, `||` et `!` d'abord parce que ce sont des mots rarement utilisés pour déclarer quoi que ce soit, donc si c'est réservé c'est pas trop gênant, ensuite c'est également très court à écrire donc c'est pas plus difficile et ça permet d'utiliser un nom plutôt que des symboles dont il faut connaître le sens, de plus si on utilise des mots clefs les symboles deviennent éventuellement libres pour exprimer autre chose, car il va bien falloir en utiliser, à moins de vouloir écrire du Ada (déclaration d'un pointeur sur entier) :

```Ada
Ptr : access Integer := I'access;
```

## structures de contrôle

`if ...` ? Et pour le bloc qui suit `do ... done` ? `then end` ?

`loop` ? On oublie `for`, `while` et `until` qui ne font gagner que quelques maigres lignes (mais pas de caractères en général, donc la densité d'information est la même)

## concaténation

Si on utilisait le `&` plutôt que `+` pour concaténer ?
`+` signifie additionner donc on pourrait garder ce sens pour additionner des listes :

```
[1, 2, 3] + [4, 5, 6] = [5, 7, 9]
[1, 2, 3] * [4, 5, 6] = [4, 10, 18]
[1, 2, 3] & [4, 5, 6] = [1, 2, 3, 4, 5, 6]
```

Ou alors une autre idée, pas de caractère pour concaténer ?

```
[1, 2, 3] [4, 5, 6] = [1, 2, 3, 4, 5, 6]
```

Est-ce qu'il ne faudrait pas mettre des règles de concaténation genre dès que dans une concaténation on a des nombres et chaînes qui se mélangent les nombres sont systématiquement convertis en chaîne ?

```
`J'ai ` & 30 & ` ans` = `J'ai 30 ans`
```

## séparateur

J'ai l'impression que la virgule est inutile pour le parseur qu'on la met pour des questions de lisibilité, est-ce qu'on la laisse ?

```
[1, 2, 3] + [4, 5, 6] = [5, 7, 9]
[1, 2, 3] * [4, 5, 6] = [4, 10, 18]
[1, 2, 3] & [4, 5, 6] = [1, 2, 3, 4, 5, 6]
print(`wesh j'ai `, 30, `ans`)
vrai := add(2, 2) = 4
```

vs

```
[1 2 3] + [4 5 6] = [5 7 9]
[1 2 3] * [4 5 6] = [4 10 18]
[1 2 3] & [4 5 6] = [1 2 3 4 5 6]
print(`wesh j'ai ` 30 `ans`)
vrai := add(2 2) = 4
```

Bon ça me paraît être une vraie pas bonne idée mais je voulais en discuter.

## nombre

Pouvoir l'exprimer dans n'importe quelle base, avec `_` en guise de séparateur libre.
Par ex, en Ada c'est BASE#NOMBRE# : `2#101# = 5`

## identifiant

On autorise `π` ? Est-ce que c'est utile/souhaitable de permettre ça ou on s'en tient à des lettres et des chiffres ?

### case

Je suis contre le camelCase, et les quelques études sur le sujet montre un manque de lisibilité, je suis plus team `_` en tant que séparateur de mots :

```
a
i
another_identifier
last_check
PID
```

Je suis aussi contre le fait que les majuscules aient un sens, explicite pour le compilateur ou implicite pour le dev comme c'est le cas en Go, Python, Lua, etc
Genre utiliser la casse de la première lettre pour exporter, mettre les constantes tout en majuscules, je préfère les mots-clefs `export` et `constant` à vue de nez.

Et je suggère que le code soit case insensitive, comme ça on évite les variables qui se ressemblent, les devs qui veulent jouer avec ça.

#### liens

https://en.wikipedia.org/wiki/Naming_convention_(programming)
