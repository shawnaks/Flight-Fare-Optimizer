# ✈️ Flight Fare Optimizer

**Finding the cheapest, shortest, and smartest U.S. airline routes with graph search — built from scratch in Python.**

![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![Jupyter Notebook](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-data%20analysis-150458?logo=pandas&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-computing-013243?logo=numpy&logoColor=white)
![Matplotlib](https://img.shields.io/badge/Matplotlib-visualization-11557C)

> A course project for **MATH 3012 — Applied Combinatorics** at **Georgia Institute of Technology** (Fall 2024).

---

## Overview

When you fly from one city to another, the fastest itinerary is not always a direct flight — and the cheapest itinerary is rarely the shortest. **Flight Path Optimizer** turns U.S. domestic flight data into a weighted graph and uses classical graph algorithms to answer a practical question:

> *Given a departure airport and a destination, what is the best route — and what does "best" actually mean?*

The project models **airports as vertices** and **flights as weighted edges**, where edge weights represent average ticket fares. It then implements and benchmarks four pathfinding strategies entirely from scratch:

- **Breadth-First Search (BFS)** — minimizes the number of flights.
- **Depth-First Search (DFS)** — a baseline that illustrates why uninformed search fails on cyclic graphs.
- **Dijkstra's Algorithm** — minimizes total ticket cost.
- **Modified Dijkstra's Algorithm** — adds a fixed penalty per leg so that itineraries with fewer layovers are preferred, capturing the real-world cost of a stopover.

All four algorithms are evaluated on **25,000 randomly generated airport pairs** using three metrics: runtime, number of flights, and total fare.

---

## At a Glance

| Metric | Value |
|---|---|
| Dataset | U.S. Airline Flight Routes and Fares (Kaggle) |
| Records analyzed | ~246,000 quarterly route observations (2021 – Q1 2024) |
| Airports in graph | 176 |
| Algorithms implemented | 4 (BFS, DFS, Dijkstra, Modified Dijkstra) |
| Benchmark trials | 25,000 random routes per algorithm |
| Languages / tools | Python, pandas, NumPy, Matplotlib, Jupyter/Colab |
| Deliverables | Executable notebook + 25-page technical report |

---

## Table of Contents

- [Overview](#overview)
- [At a Glance](#at-a-glance)
- [How It Works](#how-it-works)
- [Algorithms](#algorithms)
- [Benchmark Results](#benchmark-results)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Key Findings](#key-findings)
- [Limitations & Future Work](#limitations--future-work)
- [Report & References](#report--references)
- [Authors](#authors)
- [License](#license)

---

## How It Works

The pipeline converts a raw fare dataset into a searchable graph in four stages.

### 1. Clean and reshape the data

The raw CSV contains many columns that are irrelevant to routing, such as airline market-share fields and geocoded city identifiers. The notebook:

- Drops unused columns.
- Filters to records after 2017 (the analyzed slice spans 2021 through Q1 2024).
- Reorders the remaining fields for readability.

The retained schema is:

| Column | Meaning |
|---|---|
| `Year`, `quarter` | Observation period |
| `airport_1`, `airport_2` | Origin and destination airport codes |
| `city1`, `city2` | Associated cities / metro areas |
| `fare` | Average ticket fare for the route |
| `fare_lg`, `fare_low` | Fares from the largest and lowest-fare carriers |
| `carrier_lg`, `carrier_low` | Corresponding carriers |
| `nsmiles` | Nonstop distance in miles |
| `passengers` | Passenger volume |

### 2. Make routes bidirectional

The source dataset stores each market pair only once. Since travelers need paths in both directions, the notebook duplicates every row with the origin and destination swapped, assuming fares are symmetric. This doubles the edge set and makes the graph strongly connected in both directions.

### 3. Collapse repeated routes

The same city pair appears across multiple years and quarters at different prices. Routes are grouped by `(airport_1, airport_2)` and aggregated:

- `fare`, `fare_lg`, `fare_low`, and `nsmiles` → **mean**
- `carrier_lg`, `carrier_low`, `city1`, `city2` → **first**

The result is one representative edge per directed airport pair.

### 4. Build three graph representations

| Graph | Structure | Edge weight | Purpose |
|---|---|---|---|
| `graph_dict` | Adjacency list | Unweighted | BFS and DFS |
| `weighted_graph_dict` | Adjacency map | Average fare ($) | Cost-minimizing Dijkstra |
| `layover_graph_dict` | Adjacency map | Average fare + $80 | Layover-aware Dijkstra |

~~~text
flight_data.csv
      │
      ▼
clean columns ──► filter years ──► duplicate reversed routes ──► average by airport pair
      │
      ▼
┌─────────────────────┬──────────────────────────┬─────────────────────────────┐
│ graph_dict          │ weighted_graph_dict      │ layover_graph_dict          │
│ (unweighted)        │ (fare weights)           │ (fare + $80 per leg)        │
└─────────┬───────────┴────────────┬─────────────┴──────────────┬──────────────┘
          │                        │                            │
        BFS / DFS              Dijkstra                Modified Dijkstra
          │                        │                            │
          └──────────────► 25,000-route benchmark ◄─────────────┘
                                   │
                                   ▼
                     Runtime · Path length · Route cost charts
~~~

---

## Algorithms

| Algorithm | Optimizes for | Optimal? | Data structure | Role in the study |
|---|---|---|---|---|
| **DFS** | Nothing (explores depth-first) | No | Stack / recursion | Baseline; demonstrates poor behavior on dense, cyclic graphs |
| **BFS** | Number of flights | Yes (fewest edges) | `collections.deque` | Fastest itinerary in terms of legs |
| **Dijkstra** | Total fare | Yes (non-negative weights) | `heapq` priority queue | Cheapest itinerary regardless of layovers |
| **Modified Dijkstra** | Fare + convenience | Yes, under modified weights | `heapq` priority queue | Balanced, practical recommendation |

### Dijkstra's Algorithm

Each node starts with a distance of infinity except the origin, which is set to zero. A min-heap repeatedly extracts the unvisited node with the smallest tentative distance, relaxes its outgoing edges, and records the predecessor that produced the improvement. Once the heap is exhausted, the shortest path is reconstructed by backtracking through the predecessor map.

Complexity: **O((V + E) log V)** with a binary heap.

### The Modified Dijkstra (Layover-Aware) Variant

The core insight of the project: a pure cost-minimizing route may include an unreasonable number of layovers simply because each leg is cheap. To make long itineraries less attractive, the notebook adds a constant **c = $80** — an estimate of the opportunity cost of a layover — to every edge weight:

```text
w' = w + c
```

For two paths with the same base cost `w`, one with `m` legs and one with `n` legs where `m > n`:

```text
w'_m = w + c·m
w'_n = w + c·n
w'_m > w'_n   because   m > n
```

Adding a constant therefore penalizes every additional leg equally, so the algorithm prefers fewer stops while still accounting for fare. The benchmark confirms the effect: the modified variant produces paths almost as short as BFS while keeping costs close to pure Dijkstra.

### DFS Implementation Detail

DFS shuffles each node's neighbor list before recursing. This prevents the search from always following the dataset's insertion order and produces varied paths across runs. It also makes DFS's poor performance visible across many random trials.

---

## Benchmark Results

Every algorithm was run on the **same 25,000 randomly selected origin–destination pairs**. Start and end airports were always distinct to avoid zero-length routes skewing the averages.

| Algorithm | Mean runtime | Mean flights per route | Mean route fare |
|---|---:|---:|---:|
| **DFS** | 1.50 ms | 45.97 | $10,557.98 |
| **BFS** | 1.80 ms | **2.00** | $440.98 |
| **Dijkstra** | 1.15 ms | 2.26 | **$333.24** |
| **Modified Dijkstra** | **1.03 ms** | 2.08 | $339.00 |

> Averages over 25,000 trials. Runtimes are per query.

The notebook renders horizontal bar charts for total runtime, mean runtime, mean path length, and mean route cost, including zoomed-in comparisons of BFS, Dijkstra, and Modified Dijkstra.

### Most Connected Airports

The graph's largest hubs, by number of outgoing nonstop destinations:

| Airport | Connections |
|---|---:|
| DFW (Dallas/Fort Worth) | 112 |
| ORD (Chicago O'Hare) | 105 |
| EWR (Newark) | 105 |
| DAL (Dallas Love Field) | 105 |
| SNA (Orange County) | 104 |
| PHX (Phoenix) | 104 |
| LGA (New York LaGuardia) | 104 |
| LAX (Los Angeles) | 104 |
| JFK (New York JFK) | 104 |

---

## Features

- **End-to-end data pipeline** — from raw Kaggle CSV to a query-ready graph, including cleaning, symmetrization, and route aggregation.
- **Three graph representations** — unweighted adjacency lists plus two weighted adjacency maps for different optimization goals.
- **Four algorithms written from scratch** — no NetworkX or external graph library; BFS, DFS, and Dijkstra are implemented directly with `deque` and `heapq`.
- **Layover-aware routing** — a principled, provable modification that trades a small fare increase for significantly fewer stops.
- **Large-scale evaluation harness** — 25,000 randomized queries per algorithm with automated timing and metric collection.
- **Publication-quality visualizations** — Matplotlib bar charts comparing runtime, path length, and route cost.
- **Reproducible notebook** — a single, well-commented Jupyter notebook (32 cells) that runs top-to-bottom.
- **Full technical write-up** — a 25-page report covering mechanisms, pseudocode, proofs, results, and future work.
- **Colab-ready** — designed to run in Google Colab with a simple file-upload cell.

---

## Tech Stack

| Layer | Tools |
|---|---|
| Language | Python 3 |
| Data processing | pandas, NumPy |
| Graph algorithms | Standard library — `heapq`, `collections.deque`, `random`, `time` |
| Visualization | Matplotlib |
| Environment | Jupyter Notebook / Google Colab |
| Documentation | LaTeX-style PDF report |

No third-party graph libraries are used — every traversal and shortest-path routine is implemented explicitly.

---

## Project Structure

~~~text
MATH-3012-Project/
├── Flight_Optimizer.ipynb     # Main notebook: data pipeline, algorithms, benchmark, charts
├── MATH_3012_Report.pdf       # 25-page technical report with methodology and analysis
├── README.md                  # Project documentation
└── .gitignore                 # Standard ignore rules
~~~

### Notebook Walkthrough

| Cells | Section |
|---|---|
| 1–2 | Imports and CSV upload |
| 3–7 | Data loading, column cleanup, year filtering, reordering |
| 8–10 | Airport inventory, route reversal, aggregation by airport pair |
| 11–13 | Construction of the unweighted, weighted, and layover-penalized graphs |
| 14–15 | Hub analysis and connection lookup |
| 16–19 | BFS, DFS, Dijkstra, and path-cost implementations |
| 20–24 | Individual algorithm tests and direct-vs-layover comparisons |
| 25–31 | 25,000-route benchmark and result visualizations |

---

## Getting Started

### Prerequisites

- Python 3.9 or newer (or a Google account for Colab)
- `pip`
- The `flight_data.csv` dataset (not bundled with the repository)

### Dataset

Download **US Airline Flight Routes and Fares 1993–2024** from Kaggle:

<https://www.kaggle.com/datasets/bhavikjikadara/us-airline-flight-routes-and-fares-1993-2024>

Place `flight_data.csv` in the project root or upload it when prompted by the notebook.

### Option A — Run in Google Colab (recommended)

1. Open [Flight_Optimizer.ipynb](https://github.com/shawnaks/MATH-3012-Project/blob/main/Flight_Optimizer.ipynb) in Colab.
2. Select **Runtime → Run all**.
3. When the upload prompt appears, choose `flight_data.csv`.
4. The notebook processes the data, runs every algorithm, and renders the benchmark charts.

The notebook imports `google.colab.files`, so it works out of the box in Colab.

### Option B — Run Locally

~~~bash
git clone https://github.com/shawnaks/MATH-3012-Project.git
cd MATH-3012-Project

python -m venv .venv

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# macOS / Linux
source .venv/bin/activate

pip install numpy pandas matplotlib jupyter
jupyter notebook Flight_Optimizer.ipynb
~~~

Because the notebook was authored in Colab, replace the upload cell with a local file read:

~~~python
# Replace:
# from google.colab import files
# uploaded = files.upload()

import pandas as pd
flight_data = pd.read_csv("flight_data.csv", low_memory=False)
~~~

Then run the remaining cells in order.

---

## Key Findings

- **DFS is unsuitable for flight planning.** On a dense graph full of cycles, it produced itineraries averaging **46 flights** and over **$10,500** in fares — routes that could be flown in two or three legs. Its runtime also exceeded Dijkstra's, contradicting naive complexity expectations because it wanders through much of the graph before reaching the goal.
- **BFS delivers the shortest itineraries.** It guarantees the minimum number of flights and is ideal when time or simplicity matters more than price, such as emergency travel. However, its routes cost about **$102 more** on average than the modified Dijkstra.
- **Dijkstra finds the cheapest routes.** It produced the lowest mean fare (**$333.24**), making it the right choice for budget-focused travelers who do not mind extra layovers.
- **Modified Dijkstra is the best general-purpose strategy.** It was the **fastest** algorithm in the benchmark, kept path length within **0.08 flights of BFS**, and saved roughly **$102 per route** versus BFS while costing only **$5.76 more** than pure Dijkstra. In effect, it behaves like a hybrid of BFS and Dijkstra — a realistic model of how travelers actually weigh price against convenience.

---

## Report & References

The full analysis — including algorithm pseudocode, correctness discussion, the layover-penalty proof, charts, and conclusions — is available in [`MATH_3012_Report.pdf`](./MATH_3012_Report.pdf).

1. Miao Bin. *Graph Traversal in Python: Breadth First Search (BFS).* Medium.
2. Shimon Even. *Graph Algorithms.* Cambridge University Press, 2008, pp. 46–48.
3. Bhavik Jikadara. *US Airline Flight Routes and Fares 1993–2024.* Kaggle.
4. Kurt Mehlhorn. *Algorithms and Data Structures: The Basic Toolbox.* Springer, 2008, Chap. 10.
5. Mitchel T. Keller and William T. Trotter. *Applied Combinatorics.* 1986.

---

## Authors

**MATH 3012 — Applied Combinatorics**  
Georgia Institute of Technology · November 2024

- Shawnak Samaddar
- Sean Lee
- David Xibille
- Riley Jocham

---