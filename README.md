# 🌍 SoilGrids Italy — Clustering Pedologico e Data Mining Spazio-Temporale

![Python](https://img.shields.io/badge/Python-3.9%2B-blue.svg)
![Scikit-Learn](https://img.shields.io/badge/Scikit--Learn-1.2%2B-orange.svg)
![xarray](https://img.shields.io/badge/xarray-NetCDF4-green.svg)
![LaTeX](https://img.shields.io/badge/LaTeX-PDF%20Report-blueviolet.svg)
![License](https://img.shields.io/badge/License-MIT-brightgreen.svg)

Progetto didattico e di ricerca per il corso di **Machine Learning e Data Mining (MLDM)**.  
L'obiettivo del progetto è l'analisi pedologica ad alta risoluzione del territorio italiano attraverso tecniche di **Data Mining, Riduzione della Dimensionalità e Clustering Non Supervisionato** su dati grigliati 3D provenienti dal database internazionale **SoilGrids 2.0 (ISRIC)**.

---

## 📌 Indice delle Sezioni

- [Panoramica del Progetto](#-panoramica-del-progetto)
- [Architettura della Repository](#-architettura-della-repository)
- [Pipeline Metodologica](#-pipeline-metodologica)
  - [1. Data Ingestion & Integrazione 3D](#1-data-ingestion--integrazione-3d)
  - [2. EDA & Feature Engineering](#2-eda--feature-engineering)
  - [3. Riduzione della Dimensionalità (PCA)](#3-riduzione-della-dimensionalità-pca)
  - [4. Algoritmi di Clustering](#4-algoritmi-di-clustering)
  - [5. Validazione Diagnostica (Yellowbrick & Optuna)](#5-validazione-diagnostica-yellowbrick--optuna)
- [Risultati & Benchmark Numerici](#-risultati--benchmark-numerici)
- [Requisiti e Installazione](#-requisiti-e-installazione)
- [Esecuzione del Codice](#-esecuzione-del-codice)
- [Compilazione della Relazione LaTeX](#-compilazione-della-relazione-latex)
- [Autore & Licenza](#-autore--licenza)

---

## 🎯 Panoramica del Progetto

I suoli giocano un ruolo fondamentale nella regolazione del ciclo dell'acqua, nel sequestro del carbonio organico e nel supporto alla produzione agricola. Il dataset **SoilGrids 2.0** fornisce stime delle proprietà fisico-chimiche del suolo a varie profondità standard (da 0 a 200 cm).

Questo progetto implementa una pipeline completa in Python per:

1. **Riconciliare e fondere raster 3D / file NetCDF** riguardanti pH, capacità di scambio cationico (CEC), sostanza organica (SOC/OCD), azoto totale, densità apparente (BDOD) e tessitura (argilla, sabbia, limo).
2. **Definire i profili chimico-fisici del suolo italiano** aggregati sull'intero profilo verticale ($0-200\text{ cm}$) tramite media pesata sulle campionature di profondità.
3. **Identificare macro-regioni pedologiche (zone pedoclimatiche coerenti)** tramite l'applicazione e il confronto critico di molteplici algoritmi di clustering non supervisionato.
4. **Analizzare le performance computazionali e la scalabilità** dei metodi tramite strategie di Mini-Batch e selezione iperparametrica automatizzata (Optuna).

---

## 📁 Architettura della Repository

```text
Progetto2026/
├── Project.py                 # Pipeline Python principale (EDA, PCA, Clustering, Benchmark, Export)
├── README.md                  # Documentazione del progetto
├── soilgrids_italy/           # Dataset e matrici geospaziali
│   ├── data/                  # File NetCDF (.nc) e geotiff (.tif) multilivello
│   └── README.md              # Documentazione interna del dataset
└── relazione/                 # Relazione tecnica in LaTeX
    ├── relazione.tex          # Sorgente LaTeX principale (28 pagine)
    ├── relazione.pdf          # Report PDF compilato finale
    └── figure/                # Grafici ed esportazioni ad alta risoluzione
        ├── distribuzioni_eda.png
        ├── matrice_correlazione.png
        ├── elbow_curve.png
        ├── silhouette_curve.png
        ├── pca_clusters.png
        ├── mappa_kmeans.png
        ├── mappa_bisecting.png
        ├── mappa_dbscan.png
        ├── mappa_gmm.png
        ├── profili_cluster.png
        ├── bic_aic_gmm.png
        └── ...
```

---

## 🛠️ Pipeline Metodologica

### 1. Data Ingestion & Integrazione 3D

- **Sorgente Dati**: Matrici geospaziali NetCDF (`soil_matrices_3d_with_means.nc`) integrate con raster `.tif` per i 6 livelli standard di profondità (`0-5cm`, `5-15cm`, `15-30cm`, `30-60cm`, `60-100cm`, `100-200cm`).
- **Allineamento Spaziale**: Correzione del sistema di riferimento delle coordinate (CRS) e ricampionamento `interp_like` con librerie `xarray` e `rioxarray`.
- **Pulizia Fisico-Chimica**: Mascheramento automatico delle celle marine e "NoData". Verifica del vincolo granulometrico ($\text{Sabbia} + \text{Limo} + \text{Argilla} = 100\%$).

### 2. EDA & Feature Engineering

- **Profilazione Verticale**: Calcolo della media pesata sugli spessori dei vari strati per rappresentare le proprietà nell'intervallo $0-200\text{ cm}$.
- **Variabili Analizzate**: pH ($\text{pH}_{H_2O}$), Cation Exchange Capacity ($\text{CEC}$), Organic Carbon Density ($\text{OCD}$), Nitrogen, Bulk Density ($\text{bdod}$), Sand, Silt, Clay, Water Retention ($\text{wv0010}$, $\text{wv0033}$, $\text{wv1500}$).
- **Normalizzazione**: Standardizzazione Z-Score (`StandardScaler`) per bilanciare variabili con unita di misura eterogenee.

### 3. Riduzione della Dimensionalità (PCA)

- Analisi delle componenti principali per identificare le direzioni di massima varianza e ridurre il rumore background.
- **PC1 (24.4% varianza)**: Associata alla capacità di ritenzione idrica e al contenuto di argilla/sostanza organica.
- **PC2 (14.5% varianza)**: Associata all'alcalinità, al pH e alla presenza di frazione sabbiosa.

### 4. Algoritmi di Clustering

Vengono implementati e confrontati quattro paradigmi fondamentali:

- **K-Means Clustering**: Partizionamento sferico basato sulla distanza euclidea. Scelta di $K=6$ ottimale.
- **Bisecting K-Means**: Approccio gerarchico divisivo top-down con scissioni binarie sequenziali.
- **DBSCAN**: Clustering basato sulla densità per individuare strutture non sferiche e filtrare il rumore pedologico.
- **Gaussian Mixture Models (GMM)**: Clustering probabilistico con matrici di covarianza intere (`full`).

### 5. Validazione Diagnostica (Yellowbrick & Optuna)

- **KElbowVisualizer & SilhouetteVisualizer**: Diagnostic plots avanzati con la libreria Yellowbrick.
- **Intercluster Distance Maps**: Proiezione multidimensionale delle distanze tra i centroidi dei cluster.
- **Optuna Hyperparameter Tuning**: Ottimizzazione Bayesiana per la ricerca automatizzata dei parametri ottimali.

---

## 📊 Risultati & Benchmark Numerici

I principali risultati quantitativi ottenuti dalla pipeline sull'intero dataset del territorio italiano:

### 1. Metriche di Qualità dei Cluster

| Algoritmo             | Silhouette Score ($\uparrow$) | Davies-Bouldin ($\downarrow$) | Calinski-Harabasz ($\uparrow$) | Note / Caratteristiche                          |
| :-------------------- | :---------------------------: | :---------------------------: | :----------------------------: | :---------------------------------------------- |
| **K-Means ($K=6$)**   |          **0.1041**           |           **2.012**           |           **17,548**           | Partizioni bilanciate e interpretabili          |
| **Bisecting K-Means** |            0.0739             |             2.153             |             14,892             | Struttura ad albero divisiva                    |
| **DBSCAN**            |            0.0869             |             2.341             |             3,105              | Riconosciuti 5.570 punti di rumore (6.20%)      |
| **GMM ($K=6$)**       |            0.0658             |             2.289             |             12,410             | Modello probabilistico (BIC: 2.78M, AIC: 2.77M) |

### 2. Concordanza tra le Partizioni (ARI & NMI)

- **K-Means vs. Bisecting K-Means**: $\text{ARI} = 0.458$ (Concordanza moderata-elevata sulla macro-struttura).
- **K-Means vs. GMM**: $\text{ARI} = 0.387$, $\text{NMI} = 0.462$ (Buona sovrapposizione nei core pedologici).

### 3. Efficienza Computazionale & Scalabilità

| Metodo                 | Tempo Esecuzione (s) |           Speedup Factor           |
| :--------------------- | :------------------: | :--------------------------------: |
| **K-Means Standard**   |   $0.437\text{ s}$   |       $1.0\times$ (Baseline)       |
| **Mini-Batch K-Means** | **$0.141\text{ s}$** | **$\approx 3.1\times$ più veloce** |

---

## 💻 Requisiti e Installazione

### Dipendenze Python

È consigliato l'uso di un ambiente virtuale (`conda` o `venv`) con Python 3.9+:

```bash
# Clone del repository
git clone https://github.com/tuo-username/Progetto2026.git
cd Progetto2026

# Installazione delle dipendenze
pip install numpy pandas xarray rioxarray rasterio netcdf4 scikit-learn matplotlib seaborn yellowbrick kneed optuna
```

---

## 🚀 Esecuzione del Codice

### 1. Esecuzione della Pipeline Completa

Per eseguire l'analisi completa, generare le metriche e salvare tutti i grafici nella cartella `relazione/figure/`:

```bash
python Project.py
```

### 2. Struttura dell'Output

Al termine dell'esecuzione, il programma:

- Stampa a terminale la profilazione dei dati e le metriche di validazione.
- Genera i file di riepilogo e le **18 figure ad alta risoluzione** pronte per la relazione.

---

## 📄 Compilazione della Relazione LaTeX

La documentazione formale del progetto è scritta in **LaTeX** con layout moderno professionale.

```bash
cd relazione
pdflatex -interaction=nonstopmode relazione.tex
pdflatex -interaction=nonstopmode relazione.tex
```

Il PDF finale `relazione.pdf` è composto da **28 pagine** ricche di tabelle, formulazioni matematiche, mappe geografiche e grafici diagnostici.

---

_Progetto realizzato con l'ausilio di dati trasparenti SoilGrids 2.0 / ISRIC World Soil Information._
