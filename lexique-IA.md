# *Glossaire*

1. Les fondations (données & calcul)

- Données — la matière première brute
- Big Data — données à très grand volume/vitesse/variété
- DataSet — un ensemble de données structuré, utilisé pour entraîner ou tester un modèle
- Algorithme — la suite de règles/instructions qui traite les données

2. La grande famille de l'IA (hiérarchie d'inclusion)

- Machine Learning — les systèmes apprennent des motifs à partir des données plutôt que d'être programmés explicitement
- Deep Learning — sous catégorie du ML utilisant des réseaux de neurones à plusieurs couches
- Réseau de Neurones — la structure mathématique inspirée du cerveau, brique de base du Deep Learning
- Paramètre — les valeurs internes ajustées pendant l'entraînement du réseau (plus il y en a, plus le modèle est "gros")
- Modèle — le résultat entraîné — un réseau de neurones + ses paramètres, prêt à faire des prédictions

3. Domaines d'application du Deep Learning

- NLP (Traitement du Langage Naturel) — la branche dédiée au texte/langage
    - LLM (Large Language Model) — un modèle de NLP à très grande échelle (Deep Learning appliqué au texte)
        - GPT, Qwen — des exemples concrets de LLM

4. Types d'IA selon la fonction

- IA Générative — génère du contenu nouveau (texte, image...) → c'est la catégorie des LLM comme GPT/Qwen
- IA Adaptative — s'ajuste en continu à son environnement/utilisateur
- IA Générale (AGI) — IA hypothétique capable de tout type de tâche cognitive humaine (pas encore atteinte)

5. Le fonctionnement d'un LLM (le cycle d'utilisation)

- Prompt — l'instruction/question envoyée au modèle
- SystemPrompt — un prompt caché, défini en amont, qui fixe le comportement général du modèle
- Token — l'unité de découpage du texte (mot/sous-mot) traitée par le modèle
- Embedding — la représentation mathématique (vecteur) d'un token/texte, qui capture son sens
- Inférence — le moment où le modèle utilise ses paramètres pour produire une réponse à partir d'un prompt (≠ entraînement)
- Hallucination — un défaut de l'inférence : le modèle génère une information fausse avec assurance

6. Construire des applications autour d'un LLM

RAG (Retrieval-Augmented Generation) — technique qui connecte un LLM à une base de données externe pour améliorer sa précision et limiter l'hallucination
Agent IA — un LLM auquel on donne la capacité d'agir (utiliser des outils, enchaîner des actions) de façon autonome
ChatBot — une application conversationnelle, souvent construite sur un LLM

7. Outils pour faire tourner des modèles

- Ollama — un outil pour exécuter des LLM en local
- ModelFile — le fichier de configuration d'Ollama qui définit comment un modèle doit être lancé (paramètres, SystemPrompt inclus...)

8. Risques, limites & cadre

- Biais — distorsions injustes héritées des données d'entraînement
- Hallucination — (déjà vu ci-dessus)
- European AI Act — le cadre réglementaire européen encadrant l'usage de l'IA
- Test de Turing — critère historique pour évaluer si une IA "pense" comme un humain
- Singularité Technologique — hypothèse d'un point où l'IA dépasserait l'intelligence humaine, liée au concept d'IA Générale