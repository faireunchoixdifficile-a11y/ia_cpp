# IA C++ locale — architecture V1

Interface Python/Tkinter pilotant un modèle Ollama local pour générer du C++
dans un dossier projet, sans dépendance Python externe.

## Choix validés

| Sujet | Décision |
|---|---|
| Modèles | architecture multi-modèles configurable : `qwen2.5-coder:7b` pour le code, `gemma4:e2b` pour résumés/titres, `nomic-embed-text` pour les embeddings |
| GUI | Tkinter/ttk (zéro dépendance, cohérent avec les autres outils internes) |
| Périmètre V1 | complète : RAG, manifeste projet, revue de diff, commits git, résumé glissant, journalisation |
| Savoirs C++ | 5 packs markdown rédigés dans le projet, extensibles par dépôt de fichiers |

## Modules

```
core/config.py          configuration portable JSON (IA_CPP_HOME)
core/db.py              SQLite + migrations (user_version)
core/ollama_client.py   client HTTP urllib : chat streaming, embeddings
core/protocol.py        parsing des blocs ===FILE: path ACTION: x===
core/files.py           écriture confinée, sauvegardes, diff, manifeste
core/gitrepo.py         dépôt git du projet, commit, revert
core/rag.py             découpage markdown, embeddings, cosinus pur Python
core/errors.py          parsing MSVC / GCC / Clang / CMake / linker
core/context.py         budget de contexte, résumé glissant
core/engine.py          orchestration (testable sans Tkinter)
ui/app.py, ui/dialogs.py
```

## Schéma de données

`projects`, `conversations`, `messages`, `summaries`, `generated_files`
(versions + sauvegardes), `build_errors`, `knowledge_packs`,
`knowledge_chunks` (embedding BLOB float32), `project_packs`, `settings`.

## Protocole d'échange

Blocs balisés plutôt que JSON — un petit modèle local casse régulièrement un
JSON strict à cause des échappements dans le code :

```
===FILE: src/main.cpp ACTION: create===
<contenu complet>
===END===
```

Actions : `create`, `replace`, `delete`. Contenu entier obligatoire, jamais
de diff ni de « reste inchangé ». Repli accepté sur les blocs annotés
` ```cpp file="..." `.

## Garde-fous d'écriture

Chemins confinés sous la racine (refus des chemins absolus et des `..`),
liste blanche d'extensions, sauvegarde horodatée dans `.ia_cpp/backups/`,
revue de diff obligatoire par défaut, un commit git par application.

## Contexte envoyé au modèle

Prompt système (fiche projet + protocole) → extraits RAG → manifeste du
projet (arborescence + déclarations extraites par regex) → résumé glissant →
N derniers messages → erreurs de compilation en attente avec extraits de
code → prompt courant. Budget en tokens configurable, résumé automatique
au-delà du seuil.

## Validation effectuée

Pipeline testé de bout en bout hors Ollama : parsing d'une réponse à
4 fichiers, écriture, commit git, manifeste du tour suivant, compilation
réelle `g++ -std=c++20 -Wall -Wextra` du code généré (retour 0, exécution
correcte), puis boucle d'erreurs avec réinjection des extraits.

## Reste à faire (V2)

1. Boucle agent : plan → fichiers → attente compilation → correctif, avec
   plan de tâches visible.
2. Pré-vérifications automatiques `clang-format` et `cppcheck` avant
   présentation du diff.
3. Compilation intégrée optionnelle (détection g++/MSVC).
4. Export/import du contexte projet.

# Démarrage

## Prérequis
1. Avoir ollama d'installer et de lancer
2. Avoir les modèles gemma4:e2b, qwen2.5-coder:7b, nomic-embed-text

## Installation

1. Dézipper dans le répertoire de développement
2. Ouvrir quatre invite de commande, dans chacune d'elle (individuellement, une commande = une invite de commande), tapez les commande :
    - ollama pull qwen2.5-coder:7b
    - ollama pull gemma4:e2b
    - ollama pull nomic-embed-text
On va vérifier Ollama et la présence des 3 modèles
    - python main.py --check 
puis :    
4. python main.py --index → indexe les 5 packs de savoirs C++
5. python main.py ou lancer.bat