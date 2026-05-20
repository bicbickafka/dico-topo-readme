# Liage INSEE des dictionnaires topographiques départementaux de la France

L'application [DicoTopo](https://dicotopo.cths.fr/) a pour vocation de réunir en une base de données unique l'ensemble des dictionnaires topographiques départementaux publiés depuis le XIXe siècle. Ce dépôt regroupe les scripts qui enrichissent ces données par liage avec les codes du Code officiel géographique (COG) de l'INSEE. Ce liage permet de cartographier les toponymes et d'assurer l'interrogation et l'interopérabilité du corpus à travers sa dimension spatiale.[^1] À titre d'exemple, la chaîne de traitement est illustrée par le Dictionnaire topographique du département de la Mayenne, disponible sur [Gallica](https://gallica.bnf.fr/ark:/12148/bpt6k204189z/f55.item).

![chain](chain.svg)

Vue d'ensemble de la chaîne de traitement :

| Étape | Input | Output |
| :--- | :--- | :--- |
| parsage | src_DT53.xml | 01_DT53_parsed.xlsx |
| categroisation | 01_DT53_parsed.xlsx | 02_DT53_classified.xlsx |
| reconnaissance d'entités nommées | 02_DT53_classified.xlsx | 03_DT53_recognized.xlsx |
| liage | 03_DT53_recognized.xlsx<br>ref_COG_2011.xlsx | 04_DT53_matched.xlsx |
| validation #1 (responsable scientifique) | 04_DT53_matched.xlsx | 05_DT53_validated.xlsx |
| enrichissement/injection | 05_DT53_validated.xlsx<br>src_DT53.xml | 06_DT53_enriched.xml |
| validation #2 (06_DT53_enriched.xml aka output6.xml vs. src_DT53.xml) | XXX | XXX |
| générer des nouveaux ids | XXX | XXX |
| injecter les nouveaux ids | XXX | XXX |
| chargement en base | XXX | XXX |
| validation #3 (output7.xml vs. la db dicotopo.dev) | XXX | XXX |

## 01_parse.py

Ce premier script transforme la structure XML en tableau Excel. Il extrait cinq champs par article : l'identifiant, la vedette (le nom du lieu), la définition, la typologie et la localisation (au format JSON).

Par exemple, l'extrait

```xml
<article id="DT53-23764" pg="334">
    <vedette><sm>Villeneuve</sm>,</vedette>
    <definition><typologie>hameau</typologie>, <localisation>commune de Bazouges</localisation>,réuni à la
    <localisation>ville de Château-Gontier</localisation> en <date>1862</date>.</definition>
</article>
```

devient :

| Champ          | Contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |

## 02_classify.py

**classify.py** distingue les communes des autres toponymes (fermes, bois, rivières, etc.). Cette distinction est nécessaire car les communes s'apparient directement au COG par leur vedette, tandis que les autres lieux doivent être géolocalisés indirectement via les communes mentionnées dans leur localisation. Ce script analyse la typologie de chaque entrée : si elle mentionne « arrondissement », « canton », « chef-lieu » ou « commune », l'entrée est identifiée comme une commune (`is_commune: true`). Dans tous les autres cas, elle ne l'est pas (`is_commune: false`). Cette règle ne couvre cependant pas tous les cas (notamment lorsque les typologies sont vides), une validation experte reste nécessaire.

![classify](classify.svg)

alternative: check if the dico mentions a list of communes in its introduction

Exemple : Pour l'entrée « Villeneuve », la typologie indique « hameau ». Le script enregistre donc `is_commune: false`, ce qui signale que ce lieu devra être géoréférencé via les communes mentionnées dans sa localisation.

| Champ          | Contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |
| `is_commune`   | false                                                                    |

## 03_recognize.py

Il nous faut maintenant extraire les noms de lieux qui permettront l'appariement avec le COG. Pour les communes `is_commune` est vrai, le nom est la vedette elle-même. Pour les autres lieux, il faut identifier la ou les communes mentionnées dans le champ `localisation`.

#### Reconnaissance d'entités nommées

La bibliothèque SpaCy permet de reconnaître automatiquement les entités géopolitiques (GPE) et les lieux géographiques (LOC). Un système de secours basé sur des expressions régulières capture les mots à majuscule initiale en cas d'échec.

#### Normalisation des toponymes

Les toponymes extraits sont normalisés (suppression des articles, apostrophes, tirets et accents, conversion en minuscules) puis les doublons sont éliminés. Les noms normalisés sont enregistrés dans la colonne `commune_norm` au format JSON.

| Champ          | Contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |
| `is_commune`   | false                                                                    |
| `commune_norm` | ["bazouges", "chateau gontier"]                                          |

## 04_match.py

Ce script associe chaque toponyme normalisé (clé ou _key_) à un nom du COG (également normalisé) en trois étapes. Chaque correspondance est annotée avec le nom officiel du COG (`NCCENR`), le code du COG (`INSEE`) et la méthode d'appariement utilisée (`match`).

Trois étapes d'appariement :

| Étape                               | Exemple                                                                  | Match |
| ----------------------------------- | ------------------------------------------------------------------------ | ----- |
| key = COG                           | "chateau gontier" correspond à "chateau gontier" ("chateau gontier" = "chateau gontier")                      | exact |
| first_token(key) = first_token(COG) | "couesmes" correspond à "couesmes  vauce" ("couesmes"  = "couesmes" )    | fuzzy |
| key ∈ tokens(COG)                   | "vauce" correspond à "couesmes  vauce" ("vauce" ∈ ["couesmes", "vauce"]) | fuzzy |

Pour chacune de ces trois étapes, un écart d'une lettre est toléré (distance de Levenshtein ≤ 1), permettant par exemple de relier "bazouges" à "bazougers". Lorsqu'une clé correspond à plusieurs communes (par exemple, "Saint-Loup" renvoie à "Saint-Loup-du-Dorat" et "Saint-Loup-du-Gast"), toutes les correspondances sont référencées avec un match fuzzy également.	

(nuille sur ouette	-->	soulge sur ouette)

(Javron --> Javron-les-Chapelles)
(Chapelles --> Javron-les-Chapelles)

Exemple de résultat :

| Champ          | Contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |
| `is_commune`   | false                                                                    |
| `commune_norm` | ["bazouges", "chateau gontier"]                                          |
| `NCCENR`       | ["Bazougers", "Château-Gontier"]                                         |
| `INSEE`        | ["53025", "53062"]                                                       |
| `match`        | ["fuzzy", "exact"]                                                       |

## 06_enrich.py

**enrich.py** enrichit le XML en fonction du statut `is_commune` de chaque article. Lorsque `is_commune` est vrai, l'article reçoit un attribut `type="commune"` et une balise enfant `<insee>` contenant le code COG. Pour les autres articles (`is_commune = false`), chaque commune identifiée dans `<localisation>` est encapsulée dans une balise `<commune insee="...">` avec son code COG en attribut.

![inject](inject.svg)

**to-do/tbc : ajout de precision="approximatif" dans le cas de localisations multiples, else precision="certain"**

## validation #2 : Tests après enrichissement (output6.xml vs. DT03.xml)

https://github.com/chartes/dico-topo/blob/enrichissement_xml_dt/data/_OUTPUT6_VALDATION_PROCEDURE.md :

Décomptes
Vérification des vedettes les plus longues
Vérification de l’ordre des mots
Tests xpath
Gestion des @precision
Formes anciennes
Validation XML
Contrôle du formatage

// marguerite

for $a in //article, $c in $a//commune return concat($a/@id, '&#9;', $c/@insee, '&#9;', $c/@precision, '&#9;', $c/text())

checks:
insee → NCCENR
NCCENR_norm → commune_dt_norm

## générer des nouveaux ids

pilot.py
dt2db.py

## injecter les nouveaux ids

insert_new_ids.py

input: output6.xml
output: ouput7.xml

## chargement en base (dicotopo.dev.sqlite)

renseigner le fichier bibl_gallica.tsv avec la biblio du nouveau dictionnaire → injecter dans la table "bibl"

## validation #3 : Tests après insertion en base (output7.xml vs. dicotopo.dev)

https://github.com/chartes/dico-topo-app/blob/dev/db/utils/db_manual_check.md :

bibl
id_register
place
place_comment
place_description
place_feature_type
place_old_label
responsability



[^1]: Le géoréférencement permet notamment de trier et filtrer les résultats selon un découpage administratif, de regrouper les lieux par commune d'appartenance, et de travailler aisément à l'échelle nationale, ce que la fragmentation en dictionnaires départementaux rendait complexe et laborieux.
