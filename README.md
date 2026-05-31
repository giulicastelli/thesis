# A Comparative Network Analysis of Professional Tennis

**Structural Differences Between ATP and WTA Circuits Using Neo4j and Graph Data Science**

Bachelor's thesis — LUISS Guido Carli, Department of Business and Management.
Bachelor's Degree in Management & Computer Science, Course of Databases & Big Data.
Candidate: Giulia Castelli (Student ID 300691).
Supervisor: Prof. Blerina Sinaimeri. Academic Year 2025/2026.

## Overview

This project models 2000–2024 main-tour singles matches as two directed, weighted graphs (winner → loser, weight = number of wins) and compares the men's (ATP) and women's (WTA) circuits along two dimensions:

- **Dominance concentration** — degree centrality, weighted PageRank, Gini coefficient
- **Competitive reciprocity** — global reciprocity and per-pair asymmetry

The main finding is that the ATP circuit is slightly more concentrated *and* more reciprocal than the WTA, driven by the long-overlapping Big-3 elite core rather than by greater rivalry balance across the field.

## Repository structure

```
.
├── final_datasets.ipynb      # data ingestion, cleaning, dataset construction
├── network_analysis.ipynb    # Neo4j/GDS network construction, metrics, figures
├── data/
│   ├── matches_all.csv
│   ├── players_all.csv
│   ├── tournaments_all.csv
│   ├── rankings_all.csv
│   └── github_raw_data/      # raw Sackmann ATP/WTA source files
├── output/                   # figures used in the thesis
└── ER_diagram.graphml        # entity-relationship diagram of the graph model
```

## Data

All match, player, tournament, and ranking data used in this thesis come from **Jeff Sackmann's** publicly available ATP and WTA repositories:

- ATP: https://github.com/JeffSackmann/tennis_atp
- WTA: https://github.com/JeffSackmann/tennis_wta

The raw source files are kept in `data/github_raw_data/`. The cleaned, harmonised datasets used by the analysis (`matches_all.csv`, `players_all.csv`, `tournaments_all.csv`, `rankings_all.csv`) are built from those sources in `final_datasets.ipynb`.

I did not collect any of the underlying tennis data myself — full credit for the data goes to Jeff Sackmann. The Sackmann repositories are distributed under the **Creative Commons Attribution-NonCommercial-ShareAlike 4.0** license, and any reuse of the data in this repository is subject to those terms.

## Requirements

- Python 3.10+
- Jupyter
- `pandas`, `numpy`, `matplotlib`, `networkx`, `scipy`, `statsmodels`, `powerlaw`
- `neo4j` (Python driver) and a local Neo4j instance with the **Graph Data Science (GDS)** library installed

Install Python dependencies:

```bash
pip install pandas numpy matplotlib networkx scipy statsmodels powerlaw neo4j
```

## Running the analysis

1. Start a local Neo4j instance with the GDS plugin enabled (default Bolt URI `neo4j://127.0.0.1:7687`, user `neo4j`). The notebook creates a database named `tennisdb`; adjust `NEO4J_URI`, `NEO4J_USER`, and `NEO4J_PASSWORD` at the top of `network_analysis.ipynb` to match your local setup.
2. Run `final_datasets.ipynb` to build the cleaned CSVs in `data/`.
3. Run `network_analysis.ipynb` to load the graphs into Neo4j, compute the metrics, and regenerate the figures in `output/`.

## Citation

If you use this work, please cite:

> Castelli, G. (2026). *A Comparative Network Analysis of Professional Tennis: Structural Differences Between ATP and WTA Circuits Using Neo4j and Graph Data Science.* Bachelor's thesis, LUISS Guido Carli.

## License

The tennis data in this repository is **not mine** — it is redistributed from Jeff Sackmann's ATP/WTA repositories under their **CC BY-NC-SA 4.0** terms (attribution, non-commercial, share-alike).

No license has been chosen yet for the code in this repository; until one is added, default copyright applies.
