

This repository documents a workflow, inspired by “DevOps”, “MLOps”, and “DataOps” practices but simplified and adapted specifically to the needs of economists. I hope it will evolve further and prove useful to other researchers by helping them avoid the kinds of serious data infrastructure problems that I encountered during my own projects. 

# Motivation

The problems disrupted my computing environment in the middle of revision and resubmission, complicated communication and implementation of revisions, became increasingly unmanageable as data requirements expanded exponentially. And required substantial time and effort to restore reproducibility and replication of research/project-specific results.

 In my case, these problems included, but were not limited to, the following:

- An R or Python package maintained by a third-party volunteer on GitHub, but unavailable on CRAN or PyPI, could disappear irreversibly from the computing environment during a revise-and-resubmit process after an accidental update of the R version.
- Although storing all project-specific input data within each research project improved replication across different computing environments, projects using the same datasets repeatedly had to redo cleaning and preprocessing from scratch, while codebooks and recoding procedures became fragmented across projects or were eventually lost in most cases.
- Duplicating the same datasets across multiple projects also substantially increased disk usage and cloud storage costs, with regular backups adding even further to these costs.
- When local computing power and memory became insufficient for large-scale text classification, text matching, or NLP tasks involving datasets such as patents and patent claims, deploying working environments and local LLMs to remote servers or serverless platforms from scratch required considerable time.
- Managing cron jobs and automated workflows that continuously “ingested”, preprocessed, transformed, and deployed frequently-updated data sources (often using multiple interpreters such as R and Python) became increasingly difficult.
- Because each dataset came with its own data-sharing agreement, reproducing or replicating research was costly for other researchers who needed to download datasets separately and setup the working environment in the way preferred by the authors.
- External researchers who would like to replicate or reproduce a given result frequently lacked access to or had difficulty in following revisions made to code and LaTeX files, reducing transparency and reproducibility.

# Solution

To address these issues, the workflow consists of (i) a data layer that continuously ingests, transforms, and serves datasets accessible through proper authentication, and (ii) a project layer that performs analysis on data from the data layer across a wide range of computing environments and operating systems.

### Data Layer

In the data layer, the researcher either creates or uses a pre-existing `Docker` container stored in GitHub Container Registry (`GHCR`) and `DockerHub` (Step 1), and  initializes and regularly updates a data-specific GitHub repository (Step 2). The container includes up-to-date `R` and `Python` environments, and key packages such as tidyverse and pandas, making the computing environment reproducible across different operating systems and working environments. 

This layer aims for a reproducible, low-cost "data engineering" system where each incoming income survey such as PSID, SILC-EU, firm-level datasets such as Compustat, or product-level price data such as NielsenIQ are regularly “ingested” as raw immutable data (see Step 3), immediately converted into cleaned, standardized `Parquet` datasets (see Step 4), and organized into versioned "hot" (active) and "cold" (historical, reproducibility-only) folders stored directly as Parquet files in `Dropbox`, with rich metadata and codebooks kept alongside for ease of reference (see Step 5). 

Because Parquet is a self-describing “columnar” format, no database server or object-storage is required: project layer (see below) queries the data as clean tables with `DuckDB`, which reads the Parquet files in place from Dropbox without complex querying. Authentication and access control are handled through Dropbox scoped API tokens, with short-lived tokens generated for replicators, long-lived refresh tokens generated for collaborators, RAs, and external researchers (see Step 6). 

Compute-heavy operations (text classification and text matching, NLP, or large-scale cleaning) can be executed in Docker-based environments that can be quickly deployed to computing instances when needed by pulling pre-existing Docker images, scaling compute up temporarily, and shutting it down immediately after jobs finish. So the layer results in a file-based structure that separates storage (Dropbox + Parquet), compute (Docker/EC2 on-demand), query layer (DuckDB), and governance (Dropbox permissions + metadata registry).

```yaml

┌──────────────────────────────────────────────────────────┐
│                   DATA SOURCES                           │
├──────────────────────────────────────────────────────────┤
│ Raw immutable files stored in standardized structure     │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│                TRANSFORMATION LAYER                      |             
├──────────────────────────────────────────────────────────┤
│(Dev Container + Git)│Harmonization│Validation│ Metadata  │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│                  PARQUET STORAGE                         │
├──────────────────────────────────────────────────────────┤
│ Hot Storage │ Cold Storage │ Versioned Datasets          │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│                   QUERY LAYER                            │
├──────────────────────────────────────────────────────────┤
│ DuckDB │ SQL │ In-place querying from Dropbox            │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│                  PROJECT LAYER                           │
├──────────────────────────────────────────────────────────┤
│ DevContainer │ Docker │ Git │ Analysis Scripts           │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│                 REPRODUCIBLE OUTPUTS                     │
├──────────────────────────────────────────────────────────┤
│ Tables │ Figures │ Papers │ Public Repository            │
└──────────────────────────────────────────────────────────┘
```

### Project Layer

In the project layer, the researcher initializes and regularly updates a research-specific GitHub repository and either creates or uses a pre-existing `Docker` container stored in `GHCR` and Docker Hub. The container includes up-to-date R and Python environments, and key packages such as tidyverse and pandas, making the computing environment reproducible across different operating systems (see Step 2 for all the details).

The project layer then initializes a `.devcontainer` environment and a `devcontainer.JSON` file to call the relevant Docker image from the `GHCR` or the `DockerHub` registry. The researcher uses the `DevContainer` extension available in `VSCode` or `VSCode/Positron` to build a container from the called `Docker` image. The project is run inside the container. 

The project specific input data which is sufficiently small is kept inside the project repo, all the other datasets should be queried from the data layer using `DuckDB` or `SQL`. 

The project version is traced through Git. Each Git commit in the project layer should represent one meaningful shift, with detailed explanations:

- baseline configuration
- baseline sample restriction
- baseline exploratory data analysis
- baseline model specification
- adding/removing controls from model - 1
- changing sample restriction
- modifying estimation method

A revision in response to a reviewer feedback would be made on a different Git branch. The researcher **does NOT silently change any parameters** in scripts. Commit and justify changes any like:

- “Adjust standard-error clustering level from firm to industry”

 An external researcher can then trace which changes were made at each stage, with the corresponding results associated with each revision.

After the analysis is completed, a `run_all` script executes the full workflow end-to-end, generating all tables and figures, which are then exported into a text editor used by collaborators. The repository is synchronized and subsequently made public.

If the data sharing agreement permits, authorized users with the appropriate authentication token can reproduce the full results with a few simple bash commands.

Overall, the workflow ensures that each project version is explicitly mapped to its corresponding data and code state, enabling straightforward deployment, secure access via tokens, and full reproducibility through GitHub and `Docker`-based environments.

# Implementation

[Step 1: Creating a Docker image for data analysis [DL] [PL]](implementation/step1.md)

[Step 2: Initializing and running a layer [DL] [PL] ](implementation/step2.md)

[Step 3: Choosing a storage platform [DL]](implementation/step3.md)

[Step 4: Choosing a data format [DL]](implementation/step4.md)

[Step 5: Metadata management [DL]](implementation/step5.md)

[Step 6: Providing authenticated data access to co-authors, RAs, and external researchers [DL]](implementation/step6.md)

# Implementation in Practice
For a data layer, see https://github.com/enesn/tr-silc-panel. Also see https://github.com/enesn/us-psid-shelf-r/.
For a project layer, see TBD.

