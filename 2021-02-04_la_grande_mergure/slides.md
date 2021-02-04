# La grande mergure: What happened ?

---

## Introduction

Objectif:

- faire les point sur ce qui a été fait
- rappeler les practices à changer

---

## Migration

Ramener les notebooks de validation DE dans `mytraffic-data`

- évite la duplication d'utilities
- permet aux DS de facilement regarder ces notebooks
- permet aux DE de regarder les notebooks d'analyses

---


## The Purge

- tout cramer :fire:
- repartir sur un structure de fichier plus "flat"
  - un dossier par analyse
  - on garde le minimum de code dans master
- garder 2 analyses:
  - pedestrian tile flow
  - behavior


---


## Changements dans seldon

- suppression de tout ce qui était lié à une analyse particulière
- simplification des `error_functions`

---

## Stop mutating

- le cache streamlit n'aime pas les mutations
- pandas non plus:


```python
df = pd.DataFrame({"age": [4, 26, 110], "name": ["Nachos", "Antoine", "Greg"]})
sub_df = df[df.age >= 10]
sub_df["is_old"] = True
```

```
SettingWithCopyWarning:
A value is trying to be set on a copy of a slice from a DataFrame.
Try using .loc[row_indexer,col_indexer] = value instead
```

---

## Stop mutating

- `seldon.caching` résout seulement une partie du problème
- meilleure solution: STOP MUTATING

à la place de ça

```python
df["my_column"] = ...
```

faire ça

```python
df = df.assign(my_column=...)
```

---

## Lancer un notebook

Chacun a sa manière de lancer un notebook, et toutes ne sont pas équivalentes.

Points qu'on cherche à avoir:

- pouvoir importer `seldon`
- avoir du hot reload sur `seldon`
- pouvoir importer du code de mon analyses sans taper le chemin complet

→ Pas simple à faire de manière ad-hoc

---

## Lancer un notebook

2 possiblités:

- depuis la racine, `./launch <chemin vers notebook>`
- depuis n'importe où, `seldon_launch <chemin vers notebook>` (c.f. doc)

---

## Lancer un notebook

notebook:

- expose une fonction `main()`
- pas de code au "top level"
- pas de `if __name__ == '__main__'`


---

## Gestion des dates

Avant:

- dates stockées sous forme de string
- sandwichs de `strptime` / `strftime` à la moindre manipulation

---

## Gestion des dates

Maintenant:

- pour pur python: utilisation des types de `seldon.date` partout
- pour le pandas:
  - stocké sous forme de `datetime`
  - fonctions dans `seldon.pd_dates`
  - parsing automatique des dates lors des queries Athena
  - si df créé par un autre moyen, à faire soit même

---

## Kepler GL

Visu kepler intégrée aux notebooks streamlit :fire:

---

## Autres conventions

- les notebooks commencent par `notebook_`
- on n'importe pas depuis un autre notebook, on extrait dans un fichier la fonctionalité commune
- dataframes prefixés par `df_`

---

## Best practices PR


Pour avoir des reviews de qualité, les PRs sont:

- avec un scope limité
- fréquentes


En contrepartie, les reviewers doivent être réactifs (1 jour = **trop long**)

