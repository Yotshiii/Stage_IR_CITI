# Benchmark Cross-Dataset - MLP 

Les notebooks Colab servent à entraîner un
MLP sur un dataset et le tester sur un autre, dans les deux sens, pour voir comment il généralise. Un notebook compare un dataset généré avec ID2T à TonIOT, l'autre compare un dataset généré avec ID2T
à CICIDS2017. **Les deux sur des attaques Portscan**<br>
À part les noms de fichiers et de variables, les deux notebooks
sont identiques donc tout ce qui suit s'applique aux deux.

| Notebook | Dataset A (référence) | Dataset B (comparé) |
|---|---|---|
| `05_benchmark_cross_dataset_TonIOT.ipynb` | ID2T_Portscan | TonIOT_Portscan |
| `05_benchmark_cross_dataset_CICIDS.ipynb` | ID2T_Portscan | CICIDS2017_Friday_Portscan |

## Ce qu'il vous faut

Un compte Google avec accès à Colab, et les CSV du dataset que vous voulez
tester (voir juste en dessous). Rien à installer en local pour l'usage normal,
tout tourne sur Colab. Si vous voulez quand même le faire tourner en local, il
y a une section dédiée plus bas, mais ce n'est pas comme ça que les notebooks
ont été pensés à la base.

## Comprendre le dossier DATASETS

C'est le point le plus important à saisir avant de commencer, donc autant être
clair dessus : pour chaque dataset (CICIDS ou TonIOT), il y a deux sous-dossiers.

```
DATASETS/
├── CICIDS-2017/
│   ├── Datasets - AVANT_PREP/
│   │   ├── Friday-WorkingHours_labeled (CICIDS2017).csv
│   │   └── Thursday-WorkingHours-converted_clean_20260623-153608_labeled (ID2T).csv
│   └── Datasets - POST_PREP/
│       ├── CICIDS2017_Friday_Portscan_clean.csv
│       └── ID2T_Portscan_clean.csv
└── TonIOT/
    ├── Datasets - AVANT_PREP/
    │   ├── normal_IoT_2_final_labeled - (ID2T).csv
    │   └── normal_scanning1_labeled - (TonIOT).csv
    └── Datasets - POST_PREP/
        ├── ID2T_Portscan_clean.csv
        └── TonIOT_Portscan_clean.csv
```

**AVANT_PREP**, ce sont les CSV bruts tels qu'exportés par NFStream, avant
tout nettoyage. C'est ce qu'on upload si on veut rejouer tout le pipeline de
préprocessing depuis le début (Partie 1 du notebook).

**POST_PREP**, ce sont les CSV déjà nettoyés, ils sont reconnaissables grâce au suffixe
`_clean.csv`. <br>
C'est exactement ce que produit la Partie 1 à la fin. Autrement
dit, si vous chargez ces fichiers-là, vous pouvez sauter toute la Partie 1 et
aller direct à l'entraînement du modèle et aux visualisations (Partie 2).

L'intérêt de partir des fichiers déjà préprocessés, c'est surtout le temps :
le nettoyage et l'encodage OneHot sur des gros CSV peuvent prendre plusieurs
minutes. Si vous voulez juste relancer les tests, changer un hyperparamètre du
MLP, ou refaire les graphiques t-SNE/PCA, il n'y a aucune raison de repasser
par les données brutes à chaque fois.

## Comment le notebook est organisé

### Partie 1 - Préprocessing

Le notebook commence par les imports et une cellule de configuration (noms des
datasets, colonne de label, ce qui compte comme "normal", colonnes à
supprimer, colonnes à encoder, etc.). Ensuite on upload les deux CSV bruts
(Dataset A et Dataset B) via l'interface de fichiers de Colab.

Une fois les deux datasets chargés, le notebook détecte automatiquement les
catégories à encoder en OneHot en prenant l'union des valeurs des deux
datasets - histoire que l'encodage produise exactement les mêmes colonnes des
deux côtés. Vient ensuite le nettoyage proprement dit (fonction
`clean_dataset`) : binarisation du label, suppression des colonnes inutiles,
encodage, gestion des NaN et des valeurs infinies.

Comme deux datasets nettoyés séparément peuvent finir avec des colonnes
légèrement différentes, il y a une étape d'alignement qui garde uniquement
l'intersection des colonnes des deux côtés. Le notebook supprime aussi les
features trop corrélées entre elles (seuil à 0.95 par défaut), en se basant
sur le Dataset A.

La Partie 1 se termine par l'export des deux CSV nettoyés, qui sont
automatiquement téléchargés. Ce sont ces fichiers-là qui vont dans
`Datasets - POST_PREP`.

### Partie 2 - Entraînement et benchmark

La première cellule de cette partie recharge les datasets préprocessés. Si
vous venez d'exécuter la Partie 1 dans la même session, c'est presque une
formalité - les données sont déjà en mémoire, cette cellule les relit juste
depuis le disque. Mais si vous arrivez directement ici sans passer par la
Partie 1, c'est le moment d'uploader manuellement les fichiers de
`Datasets - POST_PREP` dans Colab (sous les noms attendus, du genre
`ID2T_Portscan_clean.csv`) avant de lancer cette cellule.

Suit la configuration du MLP (deux couches de 256 neurones, ReLU, Adam, early
stopping), puis les deux expériences : entraînement sur A et test sur B, puis
l'inverse. Le `StandardScaler` est toujours réajusté uniquement sur les
données d'entraînement de chaque expérience - jamais sur le test, pour ne pas
fausser les résultats.

Le notebook affiche ensuite les matrices de confusion des deux expériences
côte à côte, un graphique comparant precision/recall/f1-score dans les deux
sens, et un résumé chiffré (avec faux positifs et faux négatifs) exporté en
`benchmark_summary.csv`.

### t-SNE et PCA

En fin de notebook, deux visualisations viennent compléter l'analyse. L'idée
est de vérifier si les attaques générées artificiellement par ID2T
ressemblent, en termes de distribution, aux attaques réelles du dataset
comparé. Le t-SNE projette les flux en 2D pour un premier aperçu visuel ; la
PCA fait la même chose mais en séparant flux normaux et flux d'attaque.

Ces cellules rechargent elles aussi les CSV préprocessés directement depuis
Colab - il faut donc que les fichiers `_clean.csv` soient déjà présents,
que ce soit parce que vous venez de faire tourner la Partie 1 ou parce que
vous les avez uploadés vous-même.

## Deux façons de s'en servir

**Pour tout rejouer depuis les données brutes** : ouvrez le notebook, lancez
les imports et la configuration, puis uploadez les fichiers
`Datasets - AVANT_PREP` correspondants (ID2T pour le Dataset A, TonIOT ou
CICIDS pour le Dataset B). Laissez tourner le reste de la Partie 1 -
nettoyage, alignement, export - puis enchaînez directement sur la Partie 2,
les données sont déjà en mémoire.

**Pour juste refaire les tests rapidement** : pas besoin de toucher à la
Partie 1. Uploadez directement les deux fichiers `_clean.csv` de
`Datasets - POST_PREP` dans l'espace de fichiers de Colab, puis allez droit à
la Partie 2 en commençant par les imports sklearn et la config du modèle. La
cellule de chargement des datasets préprocessés fera le reste.

## Quelques pièges à connaître

Les chemins de fichiers dans la Partie 2 sont codés en dur
(`/content/{NOM}_clean.csv`). Si vous renommez les fichiers en les uploadant,
pensez à ajuster `DATASET_A_NAME` / `DATASET_B_NAME` en conséquence, sinon le
notebook ne les trouvera pas.

Le Dataset A est toujours ID2T (le dataset généré artificiellement) et sert
de référence pour la suppression des features corrélées. Le Dataset B, c'est
le dataset réel auquel on le compare.

Colab efface tout ce qui est dans `/content/` à la fermeture de la session -
donc si vous revenez plus tard, il faudra réuploader les fichiers. Une cellule
(commentée par défaut) permet de monter Google Drive à la place, ce qui évite
de tout réuploader à chaque fois ; il suffit de décommenter
`drive.mount('/content/drive')` et d'ajuster les chemins en conséquence.

Enfin, le MLP a un `random_state` fixé donc son entraînement est reproductible,
mais ce n'est pas le cas du t-SNE ni de l'échantillonnage utilisé pour les
visualisations - les graphiques peuvent donc varier un peu d'une exécution à
l'autre.

## Faire tourner ça en local

Les notebooks utilisent deux fonctions propres à Colab :
`google.colab.files` pour l'upload/download, et `google.colab.drive` pour
monter Google Drive. Pour les faire tourner ailleurs (Jupyter, VS Code), il
faut remplacer ces appels :

| Dans Colab | En local |
|---|---|
| `uploaded = files.upload()` puis lecture depuis mémoire | `pd.read_csv("chemin/vers/fichier.csv")` directement |
| `files.download(path)` | Rien à faire, le fichier est déjà sur le disque |
| `drive.mount('/content/drive')` | Inutile, utilisez directement vos chemins locaux |
| `SAVE_DIR = "/content/"` | Un dossier local, par exemple `"./data/"` |

Concrètement : remplacez les cellules d'upload par un `pd.read_csv(...)` qui
pointe vers votre dossier `DATASETS/` local, enlevez les appels à
`files.download`, et mettez à jour tous les chemins (`SAVE_DIR`, `PATH_A`,
`PATH_B`, `GEN_DATASET_PATH`, `LAB_DATASET_PATH`) pour qu'ils pointent vers
vos fichiers. Le reste du notebook - nettoyage, entraînement, visualisations -
fonctionne sans rien changer, Colab n'intervenant que pour l'I/O.

## Dépendances

```
pandas, numpy, matplotlib, seaborn, scikit-learn
```

Déjà installées sur Colab. En local : `pip install pandas numpy matplotlib
seaborn scikit-learn`.

## En résumé

Pour rejouer toute la chaîne depuis zéro → fichiers `Datasets - AVANT_PREP`,
Partie 1 puis Partie 2. Pour juste refaire les tests ou les graphiques →
fichiers `_clean.csv` de `Datasets - POST_PREP`, direct en Partie 2.