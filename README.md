**Demo en ligne :** https://assistant-transactions-sql-ga5nse76ycdlrwbj6cqcbn.streamlit.app/
\# Assistant transactions

> A guarded text-to-SQL assistant that turns natural-language questions into transparent, read-only analytics queries.



Poser une question en francais sur une base de transactions et obtenir la reponse,

sans ecrire de SQL.



Un LLM traduit la question en requete SQL. Le code valide la requete avant de

l'executer. La reponse et la requete generee sont affichees ensemble.



\# Exemple



Question : Combien j'ai depense chez mes fournisseurs en juin ?



```sql

SELECT fournisseur, SUM(montant) AS total

FROM transactions

WHERE date >= '2026-06-01' AND date < '2026-07-01'

GROUP BY fournisseur

ORDER BY total DESC;

```



\## Architecture



\- `db.py` : connexion SQLAlchemy et schema de la table

\- `seed.py` : generation de 5000 transactions synthetiques

\- `prompt.txt` : instructions donnees au modele (schema, valeurs autorisees, regles)

\- `core.py` : generation SQL, validation, execution

\- `app.py` : interface Streamlit

\- `eval.py` : evaluation automatisee, resultats dans RESULTATS.md



La logique est separee de l'interface : `core.py` est testable sans lancer Streamlit.



\## Securite



Le prompt interdit les operations d'ecriture, mais un prompt se contourne.

La barriere reelle est dans `valider\_sql` et s'applique apres le modele :



\- une seule instruction, ce qui bloque `SELECT 1; DROP TABLE ...`

\- la requete doit commencer par SELECT

\- liste noire de mots-cles : INSERT, UPDATE, DELETE, DROP, ALTER, CREATE, TRUNCATE, GRANT, REVOKE, COPY, CALL, MERGE



Resultats des tests :



```

valider\_sql("SELECT 1; DROP TABLE transactions")

(False, 'Plusieurs instructions detectees')



valider\_sql("DELETE FROM transactions")

(False, 'Seules les requetes SELECT sont autorisees')



valider\_sql("SELECT SUM(montant) FROM transactions")

(True, 'SELECT SUM(montant) FROM transactions')

```



\## Evaluation



8 questions testees, 8 requetes correctes. Le detail figure dans `RESULTATS.md`.



Le cas le plus interessant est la moyenne mensuelle : le modele a construit une

sous-requete avec `DATE\_TRUNC('month', date)` au lieu d'un `AVG(montant)` naif,

qui aurait donne la moyenne par ligne et non par mois.



\## Limites connues



\- Echantillon de 8 questions, ecrites par l'auteur, sur une base d'une seule table.

&#x20; Ce resultat ne prejuge pas du comportement sur un schema complexe.

\- Les expressions temporelles ambigues sont tranchees sans alerte : "cette annee"

&#x20; est interprete comme l'annee civile, et non comme les douze derniers mois.

\- Les donnees synthetiques generent le loyer et l'assurance de facon aleatoire au

&#x20; lieu de les rendre mensuels, ce qui gonfle artificiellement ces categories.



\## Installation



```

python -m venv .venv

.venv\\Scripts\\activate

pip install -r requirements.txt

docker run --name pg-transactions -e POSTGRES\_PASSWORD=devpass -e POSTGRES\_DB=transactions -p 5433:5432 -d postgres:16

python seed.py

streamlit run app.py

```



Variables d'environnement a placer dans un fichier `.env` :



```

DATABASE\_URL=postgresql+psycopg2://postgres:devpass@localhost:5433/transactions

GROQ\_API\_KEY=votre\_cle

```



\## Stack



Python, PostgreSQL, SQLAlchemy, Streamlit, pandas, API Groq (modele openai/gpt-oss-120b).

