# Walkthrough : le challenge « victor-cesar »

- **Thème :** la cryptographie
- **Niveau :** débutant
- **Ce qu'il te faut :** un terminal (Linux, WSL ou Git Bash) et le site https://cryptii.com/ (conseillé dans `hint/read_me_first.txt`)

Ce guide résout le challenge étape par étape. Chaque étape suit le même plan :

1. **L'indice** : ce qui met sur la piste.
2. **La notion** : ce qu'il faut comprendre.
3. **La pratique** : comment faire, avec cryptii ou avec le terminal.
4. **Le résultat** : ce qu'on obtient.

Le but n'est pas seulement d'avoir le flag, c'est de comprendre **comment on a eu l'idée**.

---

## Vue d'ensemble

Le message est caché en **trois couches**, et on les enlève une par une :

```
poème.txt
  └─ couche 1 : caractères invisibles en fin de ligne      (stéganographie)
       └─ couche 2 : ces caractères forment du Morse        (codage)
            └─ couche 3 : les lettres obtenues sont décalées de 3  (chiffre de César)
                 └─ HAVE A CRYPTASTIC CHRISTMAS
```

---

## Étape 0 : lire l'énoncé

En CTF, le premier réflexe est de **tout lire attentivement**. Les auteurs cachent souvent des indices dans l'énoncé.

| Ce qu'on lit dans le README | Ce que ça nous dit |
| --- | --- |
| « Le flag est une phrase lisible en anglais » | À la fin, on doit obtenir une vraie phrase. Si ce n'est pas le cas, ce n'est pas fini. |
| Une citation de **Jules César**, et le dossier s'appelle `victor-cesar` | César a inventé une méthode de chiffrement célèbre. |
| « *trois* Rages », « ses ***3*** yeux », « **tri**-force », `poème.3` | Le chiffre **3** revient sans cesse : ce n'est pas un hasard. |
| « **Ouvrez et lisez attentivement** » | Quelque chose est caché **dans** le fichier. |

**Hypothèse de départ :** un message est caché dans le poème et chiffré avec César, avec une clé de 3. Il faut d'abord trouver ce message.

---

## Étape 1 : trouver ce qui est invisible

### L'indice

> `hint_etape1.txt` : « Dans le poème, essayez de voir ce que vous ne pouvez pas voir. »

Dans un éditeur, `poème.txt` a l'air normal. Mais si on sélectionne le texte à la souris, certaines lignes **continuent après le dernier mot** : il y a des « blancs » en fin de ligne.

### La notion : un texte, ce sont des nombres

Un ordinateur ne stocke pas des lettres, il stocke des **nombres**. Chaque caractère a son numéro :

- l'**ASCII** est le vieux système, avec 128 caractères. Par exemple, `A` = 65 et l'espace = 32 ;
- l'**Unicode** est le système moderne, avec plus de 150 000 caractères (accents, emojis, tous les alphabets…). On note ces numéros en hexadécimal, sous la forme `U+XXXX` ;
- l'**UTF-8** est la façon d'enregistrer l'Unicode dans un fichier. Un caractère y prend de 1 à 4 **octets**.

Le point important : Unicode contient **plusieurs espaces différents**. À l'écran, ils sont tous invisibles, mais pour l'ordinateur ce sont des caractères distincts. C'est une cachette idéale. Cacher un message de façon à ce qu'on ne remarque même pas qu'il existe s'appelle la **stéganographie**.

### La pratique : regarder les octets avec `xxd`

`xxd` affiche le contenu brut d'un fichier, octet par octet :

```bash
xxd poème.txt
```

```
00000060: 6575 7269 6573 2ce2 8080 e2a0 80e2 8080  euries,.........
00000070: 20e2 8080 e2a0 80e2 a080 20e2 8080 e2a0   ......... .....
```

Comment lire ce résultat :
- **à gauche**, la position dans le fichier ;
- **au milieu**, les octets en hexadécimal ;
- **à droite**, le texte, où un `.` remplace chaque octet qu'on ne peut pas afficher.

Après `euries,`, seuls trois motifs se répètent :

| Octets | Caractère | Nom |
| --- | --- | --- |
| `e2 80 80` | U+2000 | EN QUAD, un espace large |
| `e2 a0 80` | U+2800 | BRAILLE PATTERN BLANK, une case de braille vide |
| `20` | U+0020 | l'espace normal |

![Sortie de xxd sur poème.txt](image-1.png)

### Le résultat

On a trouvé **deux caractères invisibles spéciaux**. Ils forment des petits groupes de 1 à 4, séparés par des espaces normaux.

Deux symboles, des petits groupes, des séparateurs : ça ressemble beaucoup au Morse.

---

## Étape 2 : décoder le Morse

### L'indice

> `hint_etape2.txt` :
> ```
> .       = (U+2800)
> -       = (U+2000)
> [space] = (espace)
> \n      = /
> ```

Ce qu'il faut en retenir :
- U+2800 est un **point** ;
- U+2000 est un **trait** ;
- l'espace normal sépare les **lettres** ;
- chaque retour à la ligne correspond à un `/`, qui sépare les **mots** en Morse.

*Sans l'indice, on pouvait aussi essayer les deux sens. Dans l'autre sens, on obtient des codes qui n'existent pas en Morse, comme `----`.*

### La notion : le code Morse

Le Morse a été inventé au XIXᵉ siècle pour le télégraphe. Chaque lettre devient une suite de points (signal court) et de traits (signal long) :

```
A .-     B -...   C -.-.   D -..    E .      F ..-.   G --.
H ....   I ..     J .---   K -.-    L .-..   M --     N -.
O ---    P .--.   Q --.-   R .-.    S ...    T -      U ..-
V ...-   W .--    X -..-   Y -.--   Z --..
```

### La pratique

#### Méthode 1 : Mathieu sensei (flemmard d'utiliser python)

Une seule commande transforme les caractères invisibles en Morse lisible :

```bash
sed -e 's/\xe2\x80\x80/-/g' -e 's/\xe2\xa0\x80/./g' po├¿me.txt | grep -o '[-.][-. ]*$' | paste -sd/
```

Ce que fait chaque morceau :
- `sed` remplace les octets `e2 80 80` (U+2000) par `-` et les octets `e2 a0 80` (U+2800) par `.`. Ce sont exactement les motifs repérés avec `xxd` ;
- `grep -o '[-.][-. ]*$'` ne garde que la fin des lignes, là où se trouve le Morse ;
- `paste -sd/` colle les lignes ensemble avec des `/` entre elles, comme le dit l'indice 2.

Résultat :

```
-.- -.. -.-- ..../-../. --- essaie par toi même pour avoir la suite ;)
```

Ensuite, dans **cryptii** :
1. colle cette ligne dans le bloc texte tout en haut ;
2. ajoute un bloc **Morse code** et choisis **DECODE**.

#### Méthode 2 : un script Python

Pour tout faire soi-même et comprendre chaque étape :

```python
import re

# Table de traduction Morse -> lettre
MORSE = {'.-':'A','-...':'B','-.-.':'C','-..':'D','.':'E','..-.':'F','--.':'G',
         '....':'H','..':'I','.---':'J','-.-':'K','.-..':'L','--':'M','-.':'N',
         '---':'O','.--.':'P','--.-':'Q','.-.':'R','...':'S','-':'T','..-':'U',
         '...-':'V','.--':'W','-..-':'X','-.--':'Y','--..':'Z'}

texte = open('poeme.txt', encoding='utf-8').read()
mots = []

for ligne in texte.split('\n'):
    # On cherche les caractères invisibles en fin de ligne
    fin = re.search('[\u2000\u2800][\u2000\u2800 ]*$', ligne)
    if not fin:
        continue
    # U+2000 devient un trait, U+2800 devient un point
    morse = fin.group(0).replace('\u2000', '-').replace('\u2800', '.')
    mots.append(''.join(MORSE.get(code, '?') for code in morse.split(' ')))

print(' '.join(mots))
```

### Le résultat

```
KDYH D FUBSWDVWLF FKULVWPDV
```

Ce sont bien des lettres… mais ce n'est pas de l'anglais. Le message est **encore chiffré** : on passe à la couche suivante.

---

## Étape 3 : déchiffrer le César

### L'indice

> `hint_etape3.txt` : « César à chiffré en 3. Cette phrase ne fait pas de sens ? Relisez là. »

Cet indice confirme notre hypothèse de l'étape 0 : un chiffre de César avec un décalage de 3.

### La notion : le chiffre de César

Jules César chiffrait ses messages en **décalant chaque lettre** dans l'alphabet. Avec un décalage de 3 :

```
Alphabet normal : A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Chiffré (+3)    : D E F G H I J K L M N O P Q R S T U V W X Y Z A B C
```

Pour **déchiffrer**, on **recule de 3**. Au début de l'alphabet, on repart de la fin : A − 3 = X.

Ce chiffre est très facile à casser, car il n'existe que 25 décalages possibles. On peut tous les essayer en une seconde.

### La pratique

**À la main :**

```
KDYH        ->  HAVE         (K->H, D->A, Y->V, H->E)
D           ->  A
FUBSWDVWLF  ->  CRYPTASTIC
FKULVWPDV   ->  CHRISTMAS
```

**Avec cryptii :** sous le bloc Morse, ajoute un bloc **Caesar cipher** et règle-le ainsi :
- **DECODE** sélectionné ;
- **Shift** = `3` ;
- **Alphabet** = `abcdefghijklmnopqrstuvwxyz` (la valeur par défaut).

![Bloc Caesar cipher dans cryptii](image.png)

Attention : le texte chiffré ne va **pas** dans le champ Alphabet. Ce champ contient la liste des lettres utilisées pour le décalage. Si on y colle le message, cryptii affiche une erreur « duplicate characters ».

**En Python :**

```python
chiffre = "KDYH D FUBSWDVWLF FKULVWPDV"
clair = ''.join(chr((ord(c) - 65 - 3) % 26 + 65) if c.isalpha() else c for c in chiffre)
print(clair)   
```

### Le résultat


> Tadda ! Merci Victor pour ce chouette challenge !!  👻

---
---

## Note : écrire le flag (en ctf)

Les règles du README sont : tout en minuscules, des `_` à la place des espaces, et le tout entouré de `EN{}`.

```
EN{xxxx}
```

---

## Récapitulatif

| Étape | Indice | Action | Résultat |
| --- | --- | --- | --- |
| 0 | README | Repérer César et le chiffre 3 | Hypothèse : César avec une clé de 3 |
| 1 | « voir ce que vous ne pouvez pas voir » | `xxd poème.txt` | Deux caractères invisibles, U+2000 et U+2800 |
| 2 | Table point/trait | Remplacer, puis décoder le Morse | `KDYH D FUBSWDVWLF FKULVWPDV` |
| 3 | « César a chiffré en 3 » | Décaler chaque lettre de −3 | `HAVE A CRYPTASTIC CHRISTMAS` |
| 4 | Format du README | Minuscules, `_`, `EN{}` | `EN{xxxx}` |

## Ce que tu as appris

- La **stéganographie** cache l'existence même d'un message. La **cryptographie** rend un message illisible. Ici, les deux sont combinées.
- Un fichier texte, ce sont des **octets**. Quand quelque chose a l'air bizarre, regarde-les avec `xxd`.
- Un signal à **deux symboles** doit faire penser au **Morse** ou au **binaire**.
- Les challenges empilent souvent **plusieurs couches**. Si le résultat est encore illisible, ce n'est pas fini.

---

*Note pour l'auteur du challenge : le README parle de `poème.3`, mais le fichier s'appelle `poème.txt`. Il y a aussi deux coquilles : « cryptogtaphie » et « englais ».*
