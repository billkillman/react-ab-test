# NUM-50 — Prompt A contre prompt B : pas à pas

**À la fin** : deux campagnes A et B dans **Experiments**, un tableau de comparaison, une décision (adopter B ou garder A).

**Deux fichiers** :

- `prompts_b.py` : le prompt de l'API, plus une consigne ;
- `num50_ab.py` : fait répondre A et B aux mêmes réclamations, fait juger les deux réponses, compare.

```
  réclamation ─┬─ prompt A (API) ─► réponse A ─┐
               └─ prompt B (A + 1 consigne) ─► réponse B ─┴─► juge (NUM-49) ─► tableau A/B ─► décision
```

**Seul le prompt système change** : le modèle, les paramètres, les modèles de réponse, le seuil de confiance et la mise en forme sont ceux de l'API. Un test le vérifie.

---

## Étape 0 — Installer (5 min)

Copiez à la racine du projet, à côté de `app.py` : `prompts_b.py`, `num50_ab.py`, `test_num50.py`, et `lot_C_promesses.json` (NUM-110). Rien n'est remplacé.

```bash
python -m pytest -q test_num50.py          # attendu : 11 passed
```

Lancez-les dans l'environnement de l'API, puisque `num50_ab.py` importe `app.py`.

---

## Étape 1 — Écrire B (15 min, avec le Tech Lead)

Dans `prompts_b.py`, changez la `CONSIGNE`. **Une seule** : avec deux changements, on ne saura pas lequel a produit l'effet.

L'exemple livré vise le risque métier majeur :

> Ne **jamais** promettre ni annoncer un remboursement, un geste commercial, un délai ou une action qui ne figure pas explicitement dans les modèles de réponse.

Notez aussi, avant de lancer :

- **l'hypothèse** : « B réduit les défauts C4 sans dégrader les autres critères » ;
- **les critères à lire** : ceux que NUM-51 a jugés fiables (option `--criteres`).

---

## Étape 2 — Essai sur 3 réclamations (10 min)

```bash
python num50_ab.py --fichier lot_C_promesses.json --nombre 3
```

**Vérifier** : un tableau A/B s'affiche, et `num50_resultats/` contient un `.csv` et un `.md`. En cas d'erreurs 429, réglez `JUGE_INTERVALLE_S=7` dans le `.env` : ce réglage espace tous les appels au service.

---

## Étape 3 — Le lot C : montrer l'effet (≈ 40 min)

Le lot C contient 40 réclamations construites pour pousser à la promesse. C'est là que A fera assez de défauts C4 pour que l'effet de B se voie.

```bash
python num50_ab.py --fichier lot_C_promesses.json --criteres C1,C3,C4,C6
```

---

## Étape 4 — Le jeu de mesure, dans Arize : vérifier que rien ne se dégrade (≈ 1 h)

```bash
python num50_ab.py --jeu recla-jugements-mesure --criteres <critères fiables de NUM-51>
```

Cette commande lance deux campagnes, `num50-…-A-v1.0.1` puis `num50-…-B-v1.0.1-b1`. Pour les voir côte à côte : **Datasets → recla-jugements-mesure → Experiments**, puis sélectionnez les deux (libellés exacts à confirmer sur votre instance). Le tableau et le détail sont aussi écrits en local.

Facultatif, pour vérifier que B s'abstient toujours là où il faut :

```bash
python num50_ab.py --jeu recla-abstentions --criteres C6
```

Lisez-y la ligne « Abstentions » : A et B doivent être à égalité.

---

## Étape 5 — Lire et décider (30 min)

**Le tableau**

| Critère | Défauts A | Défauts B | Corrigées par B | Dégradées par B | p | Verdict |
|---|---|---|---|---|---|---|
| C4 | 12/40 | 2/40 | 10 | 0 | 0.002 | B meilleur |

- **Corrigées** : défaut avec A, plus de défaut avec B.
- **Dégradées** : l'inverse.
- **p** : probabilité d'observer un tel écart si B n'avait aucun effet. Sous 5 %, l'effet est démontré.
- **Ordre de grandeur** : il faut **au moins 6 réclamations corrigées sans aucune dégradée** pour passer sous 5 %. C'est pourquoi l'effet se montre sur le lot C (étape 3), et non sur les 44 réclamations du jeu de mesure.

**La décision**, affichée en bas du rapport :

| Constat | Décision |
|---|---|
| un critère « B moins bon » | garder A |
| un critère « B meilleur », aucun « B moins bon » | adopter B |
| même cas, mais B s'abstient plus souvent que A | à trancher : une réponse évitée n'est pas une réponse améliorée |
| aucune différence démontrée | garder A |

**En croisant les deux jeux :**

| Lot C | Jeu de mesure | Conclusion |
|---|---|---|
| B meilleur | rien de dégradé | **adopter B** |
| B meilleur | un critère dégradé | **garder A**, documenter la dégradation |
| pas de différence | — | **garder A** |

Le fichier `.csv` place les réponses A et B côte à côte : relisez-y les réclamations dégradées.

**Si B est adopté** : reportez la consigne dans `prompts.py` par une PR, avec une nouvelle `PROMPT_VERSION`.

**Une règle** : si vous retouchez B, faites-le en le testant sur `--fichier recla_jugements_calibrage.csv`, jamais après avoir vu le jeu de mesure. Sinon, on choisit la variante qui colle à ces 44 réclamations.

---

## DoD (part A contre B)

- [ ] hypothèse et consigne écrites avant de lancer ;
- [ ] lot C et jeu de mesure comparés ;
- [ ] deux campagnes visibles dans **Experiments** ;
- [ ] décision consignée, réclamations dégradées relues ;
- [ ] relecture par le Tech Lead.
