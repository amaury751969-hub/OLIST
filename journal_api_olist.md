# Journal de l'API Olist — FastAPI

**Nom :** DUCOULOMBIER Amaury
**Dates :** J1 2026/11/05 · J2 2026/11/06
**Fichier rendu :** api_olist_DUCOULOMBIER.zip

> Compétences : WCS C1.6 · C2.4 · C2.1 · C2.5 (niveau 2 · Adapter) — Simplon C6 · C7 (partiel)
> Remplis les parties **dans l'ordre**. Remplace chaque « … ». Une phrase suffit sauf mention contraire.

**Chiffres de référence (export de lundi)**
- Lignes : 110 197 · Commandes livrées : 96 478
- CA total : 15 424 463,65 R$ · Panier moyen : 159,88 R$
- Novembre 2017 : 1 154 559,53 R$ et 7 289 commandes
- SP : 5 771 713,93 R$, 40 501 commandes, 37,42 % du CA

---

# J1 — L'API sur le fichier CSV

<a id="j-m1"></a>
## M1 — Lancer l'API et la tester

- Version de Python : 3.14
- Commande de lancement que j'ai utilisée : `.\.venv\Scripts\python.exe -m uvicorn main:app --reload`
- Code et `nb_lignes` renvoyés par `/sante` : code 200 et 110197 lignes
- Différence entre `/docs` et `/sante` ouverts dans le navigateur :`/docs` affiche la documentation pour consulter et tester les routes de l'API. `/sante` affiche directement la réponse JSON indiquant le statut, la source et le nombre de lignes.
- Problème rencontré et solution (ou « aucun ») : le CSV était dans API_OLIST/Data alors que le code l'attendait dans data/ . Le fichier a été copié dans data/ventes_olist.csv.

<a id="j-m2"></a>
## M2 — Lire une route complète

### Le trajet d'une requête `GET /indicateurs`, en 4 étapes

1. Le navigateur envoie une requête HTTP GET à l'adresse http://127.0.0.1:8000/indicateurs.
2. FastAPI exécute la fonction indicateurs() du fichier main.py, qui appelle donnees.indicateurs() dans le fichier donnees_csv.py.
3. Cette fonction lit les données du fichier `data/ventes_olist.csv` et renvoie un dictionnaire Python contenant le nombre de lignes, le nombre de commandes, le CA total et le panier moyen.
4. Le modèle Indicateurs vérifie les champs et leurs types puis FastAPI renvoie une réponse JSON avec le code 200.

### L'expérience du contrat

- Quand j'ai mis `panier_moyen` en commentaire dans le modèle, la réponse ne contenait plus le champ panier_moyen.
- Conclusion : Le modèle de réponse décide des champs renvoyés au client, même si la fonction de données en renvoie davantage.

### Pourquoi deux fichiers (`main.py` et `donnees_csv.py`) ?

- `donnees_csv.py` lit les données et calcule les indicateurs, tandis que 'main.py' expose les résultats par les routes de l'API. Cette séparation permet de remplacer le CSV par PostgreSQL sans changer les routes ni le format des réponses.

### Comparaison avec lundi

- `nb_commandes` : 96 478 identique au résultat de (lundi : 96 478)
- `ca_total` : 15 424 463,65 identique au résultat de (lundi : 15 424 463,65)

<a id="j-m3"></a>
## M3 — Ventes mensuelles

### Tests

- Sans paramètre : 23 mois, du 2016-09 au 2018-08.
- `debut=2017-01` et `fin=2017-12` : 12 mois ; novembre 2017 = 1 154 559,53 R$ et 7289 commandes
- `debut=2018-02` et `fin=2017-01` : code 400, message Bad Request ("début" doit être antérieur ou égal à "fin")
- `debut=2017-13` : code 422, message Unprocessable Content (le mois 13 ne respecte pas le format autorisé)

### Questions

1. Différence entre un paramètre de chemin et un paramètre de requête, avec un exemple de cette API : 
Un paramètre de chemin fait partie de l'adresse comme "SP" dans /etats/SP. Un paramètre de requête se place après le ?, comme "debut" dans /ventes/mensuelles?debut=2017-01.
2. Qui renvoie le code 400 (toi ou FastAPI) ? Et le code 422 ? Pourquoi cette différence ? 
Je déclenche le code 400 dans mon code lorsque debut est après fin. FastAPI renvoie automatiquement le code 422 lorsqu'un paramètre ne respecte pas les règles déclarées, comme un mois égal à 13.

<a id="j-m4"></a>
## M4 — Top catégories

### Tests

- Sans paramètre : 10 catégories ; la première est health_beauty avec 9,16 %
- Somme des 10 parts : environ 62,4 %
- `n=3` : code 200, la réponse contient les 3 catégories au plus fort CA.
- `n=0` : code 422 · `n=51` : code 422

### Question

- Que reçoit Rafael s'il n'envoie pas `n` ? Où est-ce décidé dans le code ? 
Rafael reçoit les 10 catégories au plus fort CA. C'est décidé dans main.py par la valeur par défaut 10 dans Query(10, ge=1, le=50)

<a id="j-m5"></a>
## M5 — Chiffres d'un État

### Mon modèle `Etat` (copie le code)

```python
class Etat(BaseModel):
    etat: str
    ca: float
    nb_commandes: int
    part_ca: float
```

### Tests

- `/etats/SP` : CA 5 771 713,93 R$, commandes 40501, part 37.42 %
- `/etats/sp` : identique (oui / non) : oui
- `/etats/XX` : code 404, message "detail :"Aucune vente pour l'Etat "XX"."
- `/etats/SPX` : code 422

### Question

- Pourquoi un État sans vente renvoie 404 et pas une réponse vide avec le code 200 ? Le code 404 indique qu'aucune donnée ne correspond à l'Etat demandé. Une réponse vide avec le code 200 indiquerait que la demande a abouti sans signaler clairement cette absence.

<a id="j-m6"></a>
## M6 — Recette complète dans /docs

Pour chaque test : le code obtenu, puis une valeur lue dans la réponse.

1. `GET /sante` (attendu 200) : code 200, "nb_lignes": 110197.

2. `GET /indicateurs` (attendu 200) : code 200, "nb_lignes": 110197,
                                     "nb_commandes": 96478,
                                     "ca_total": 15424463.65,
                                     "panier_moyen": 159.88

3. `GET /ventes/mensuelles` sans paramètre (attendu 200, 23 mois) : code 200, 23 mois, de 2016-09 à 2018-08.

4. `GET /ventes/mensuelles?debut=2018-01` (attendu 200) : code 200, nombre de mois = 8.

5. `GET /ventes/mensuelles?debut=2018-02&fin=2017-01` (attendu 400) : code 400, message « debut » doit être antérieur ou égal à « fin ».

6. `GET /ventes/mensuelles?fin=nov` (attendu 422) : code 422, erreur de validation du format du paramètre `fin`.

7. `GET /categories/top` (attendu 200, 10 catégories) : code 200, 10 catégories : "health_beauty", "watches_gifts", "bed_bath_table", "sports_leisure", "computers_accessories", "furniture_decor", "housewares", "cool_stuff", "auto", "garden_tools".

8. `GET /categories/top?n=50` (attendu 200) : code 200, dernière catégorie de la liste "books_technical".

9. `GET /categories/top?n=0` (attendu 422) : code 422, la valeur de `n` doit être supérieure ou égale à 1.
10. `GET /etats/RJ` (attendu 200) : code 200, part du CA : 13,33 %.
11. `GET /etats/XX` (attendu 404) : code 404, message : Aucune vente pour l'État « XX ».
12. `GET /etats/SPX` (attendu 422) : code 422, le code d'État doit comporter exactement deux lettres.

### Bilan du J1

1. Ce qui fonctionne : L'API lit le CSV et donne les 5 routes demandées. Elle renvoie les indicateurs globaux, les ventes mensuelles, les meilleures catégories et les chiffres demandés par État. Les résultats vérifiés correspondent aux chiffres de référence du brief.

2. Ce qui me reste à comprendre ou à terminer : Je dois gagner en autonomie pour lancer l'API et interpréter les messages du terminal. Je dois aussi consolider ma compréhension du lien entre les routes, les réponses et les fonctions de calcul.

---

# J2 — L'API sur PostgreSQL

<a id="j-m7"></a>
## M7 — Charger les ventes dans PostgreSQL

- Base utilisée : `olist` sur PostgreSQL.
- Résultat du contrôle final du script : 110197 lignes, 96478 commandes, 15424463,65 de CA
- Droit donné à `lecteur_olist` sur la table, et pourquoi seulement celui-là : Le droit SELECT pour lire les ventes sans pouvoir les modifier ni les supprimer car l'API est en lecture seule.

<a id="j-m8"></a>
## M8 — Mettre les secrets dans `.env`

- Résultat de la commande de test : lecteur_olist

- Rôle de `.env` : stocker la configuration et le mot de passe de connexion à PostgreSQL en dehors du code Python

- Rôle de `.env.exemple` : fournir un modèle des variables à renseigner, avec de fausses valeurs, pour configurer le projet.

- Rôle de la ligne `.env` dans `.gitignore` : empêcher Git de suivre le fichier .env afin de ne pas partager les secrets du projet.

- J'ai vérifié qu'aucun `.py` ne contient le mot de passe (oui / non) : oui, la recherche de olist2026 dans les fichiers Python ne renvoie aucun résultat.

<a id="j-m9"></a>
## M9 — Réécrire les fonctions en SQL

### Résultat de `verifier_sources.py`

- Bilan affiché : 8 / 8
- Fonction qui m'a demandé le plus de corrections, et pourquoi : etat() car j'avais oublié de compléter le calcul de part_ca. J'ai également corrigé le mot de passe dans .env qui empêchait la connexion à PostgreSQL pour toutes les fonctions.


### Ma requête de `top_categories()` (copie-la)

```sql
SELECT categorie,
       round(sum(montant), 2) AS ca,
       round(100 * sum(montant) / (SELECT sum(montant) FROM analytique.ventes), 2) AS part_ca
FROM analytique.ventes
GROUP BY categorie
ORDER BY ca DESC
LIMIT %(n)s;
```

### Sécurité

- Pourquoi `%(code)s` plutôt qu'une f-string ? Donne l'exemple d'une valeur dangereuse.
%(code)s permet à psycopg de transmettre la valeur séparément du SQL : elle est traitée comme une donnée et ne peut pas modifier la requête. Avec une f-string une valeur comme SP' OR '1'='1 pourrait modifier la condition et renvoyer toutes les ventes.

- Pourquoi faire calculer PostgreSQL plutôt que Python ?
PostgreSQL effectue directement les filtres, les regroupements et les calculs, puis renvoie uniquement les résultats utiles. Cela évite de transférer les 110 197 lignes dans Python pour les additionner.

<a id="j-m10"></a>
## M10 — Basculer l'API sur PostgreSQL

- `/sante` après la bascule : source "postgresql" , lignes 110197

- Pourquoi il faut relancer l'API après avoir modifié `.env` : 
La configuration est chargée au démarrage et --reload surveille les fichiers Python, pas .env.

- Test de panne (`PGPORT=5999`) : code 503, Error: Service Unavailable  message "Base PostgreSQL injoignable"

- Ce qu'aurait vu Rafael sans le gestionnaire d'erreur `SourceIndisponible` : 
Une erreur 500 Internal Server Error, sans message clair expliquant l'indisponibilité de la base.

- Ce que Rafael a dû changer dans son code pour la bascule : 
Rien, car les adresses, les paramètres et le format des réponses restent identiques.

<a id="j-m11"></a>
## M11 — Écrire le client de Rafael

- Bilan affiché par `client.py` : 10 / 10

- Nombre de lignes de `ca_mensuel_2017.csv`, en-tête compris : 13 lignes

- Différence entre `.status_code` et `.json()` : .status_code donne le code HTTP de la réponse par exemple 200 ou 400. .json() transforme le contenu JSON en dictionnaire ou en liste Python.

- Utilité du paramètre `timeout` : limiter le temps d'attente d'une réponse pour éviter que le script reste bloqué indéfiniment. Ici fixé à 10 secondes.

<a id="j-m12"></a>
## M12 — README et rendu

- Le test de remise à zéro a fonctionné en suivant seulement le README (oui / non) : oui, l’API a redémarré après suppression de `__pycache__` et le client a validé 10/10 contrôles.

- Ce que j'ai dû corriger dans le README après ce test : aucune correction supplémentaire après le test. Les commandes de lancement du README ont permis de redémarrer l'API et d'obtenir 10/10 contrôles réussis avec le client.

- Contenu de mon zip, fichier par fichier : main.py, donnees_csv.py, donnees_pg.py, erreurs.py, client.py, verifier_sources.py, requirements.txt, env.exemple, .gitignore, README.md et olist_ventes.sql.

### Mes choix techniques (3 à 5 lignes)

- Source de données retenue et pourquoi : PostgreSQL pour centraliser les ventes et effectuer les calculs directement dans la base.

- Compte de connexion et pourquoi : 'lecteur_olist', avec les droits de lecture uniquement, car l'API sert à consulter les indicateurs. 

- Gestion des erreurs : validation des paramètres avec FastAPI et réponses adaptées : 400, 404, 422 et 503 lorsque la source est indisponible.  

- Ce que j'ajouterais pour mettre l'API en production : une authentification, une connexion HTTPS, une gestion sécurisée des secrets et une surveillance des erreurs.

### Message à Mariana (3 lignes, sans jargon)

- Ce que Rafael peut faire maintenant : Rafael peut consulter automatiquement les chiffres de ventes depuis le portail, sans attendre l'envoi d'un fichier.

- D'où viennent les chiffres : ils proviennent de la base PostgreSQL Olist et concernent les commandes livrées.

- Le chiffre qui prouve que c'est juste : Le chiffre d'affaires total est de  15 424 463,65 R$, identique au résultat de référence.
