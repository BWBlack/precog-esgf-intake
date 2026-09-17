---
title: "precog-esgf-intake: Automated discovery, validation, and optimized download management of Earth system model 
data from Earth System Grid Federation nodes."
tags:
  - Python
  - Earth System Modelling
  - Biogeochemical Cycles
  - Earth Sciences
authors:
  - name: Leonardo Bertini^[corresponding author]
    orcid: 0000-0003-3920-4476
    affiliation: 1 # (Multiple affiliations must be quoted)
  - name: Sam Ditkovsky
    orcid: 0000-0002-4759-9829
    affiliation: 1
  - name: Benedict Blackledge
    orcid: 0000-0003-0894-6776
    affiliation: 1
  - name: Jamie D. Wilson
    orcid: 0000-0001-7509-4791
    affiliation: 1
affiliations:
  - name: Department of Earth, Ocean and Ecological Sciences, University of Liverpool, UK
    index: 1
date: 11 Feb 2026
bibliography: paper.bib
---

# Summary

`precog-esgf-intake` is a Python command-line-interface (CLI) software for automated discovery, checking,
validation, and download of Earth System Model outputs from Earth System Grid Federation (ESGF) archives. The
software is designed for research workflows that require reproducible access to large, distributed climate-model
datasets and is particularly aimed at bulk screening of Earth System archives before downstream analysis. The
software builds on inherited ESGF access functionality from `intake-esgf` [@Collier_intake_esgf_2026], while
introducing interactive CLI workflows for archive interrogation, file-availability checks, grid-consistency
cross-checks, temporal validation, export of search results in tabular formats, and optimised download management of
shortlisted Earth System Model data products.

# Statement of need

Modern Earth System science workflows frequently rely on climate-model data distributed across Earth System Grid
Federation [(ESGF)](https://esgf.github.io/index.html) nodes [@ESGFAggregation]. For example, a Coupled Model
Intercomparison Project (CMIP) analysis to constrain future projections in ocean carbon inventories requires
standardized output of ocean biogeochemical variables (carbon, oxygen, nutrients) across many models and
experiments[@Wilson2022].
These archived data are often distributed across many federated storage nodes, including duplicated,
incomplete or corrupted versions. Although ESGF provides a federated infrastructure for searching and accessing these
archives, practical research workflows often require additional automation to determine whether a given combination of
variables,
experiments, ensemble members, and grid configurations is both scientifically suitable and operationally downloadable
before substantial time is spent retrieving files.

This issue is especially important for CMIP-style analyses in which researchers may need to confirm that the same Earth
System Model provides paired pre-industrial, historical and ssp-scenario simulations, that multiple variables of
interest are available and whether these are separately
archived on compatible grids for downstream analyses; and that temporal coverage is continuous across archived files.
Manual inspection of catalogue search results can become slow, repetitive, and error-prone when screening many candidate
models or variables across multiple ESGF nodes.

Among its features, `precog-esgf-intake` provides an interactive workflow for shortlisting Earth System Model datasets
that satisfy compound criteria across experiments and variables. The software validates temporal coverage, grid
consistency, and file availability across published ESGF archives before download. Users can export search results as
tabular summaries, inspect and refine shortlisted datasets, and initiate batch downloads or trigger file integrity
checks on locally assigned output paths. Retrieved data are then organised into a directory structure suitable for
reproducible downstream analysis. This workflow is particularly useful when researchers need to screen many candidate
models and variables before selecting datasets that are both scientifically appropriate and operationally accessible
within their computational and storage constraints.

Furthermore, by automating the retrieval and validation of CMIP grid metrics like `volcello` , `areacello` as default
(and enabling further bespoke searches for `deptho` or `thkcello` as a validated substitutes when
`volcello` is unavailable) — `precog-esgf-intake` reduces the need for users to understand CMIP metadata and ESGF
directory conventions. By wrapping ESGF discovery in a simple Python CLI and allowing users to tailor searches by
editing a straightforward `search_criteria.toml` configuration file, the
package supports both accessible default workflows and more bespoke dataset selection. This lowers the barrier for
researchers
new to CMIP while enabling experienced users to specify models, experiments, variables, frequencies, and other search
constraints. Its lightweight design also supports terminal-based HPC workflows, allowing data discovery and ingestion
from visualisation nodes and Jupyter sessions rather than interactive web portals or manual downloads.

# State of the field

CMIP data are currently accessible through the ESGF metagrid web application, which provides web-based discovery and
download services for CMIP5, CMIP6, CMIP6Plus, and the upcoming CMIP7 phases. Recent development efforts have focused on
analysis-ready, cloud-optimised approaches based on in-memory object storage [@Mizielinski2026], with catalogue sweep
tools enabling more scalable access through Python and xarray, while community evaluation frameworks like
ESMValTool[@ESMValTool]
supporting integrated diagnostics for benchmarking model outputs.

`precog-esgf-intake` builds on the ESGF catalogue node sweeping implementation from `intake-esgf`
[@Collier_intake_esgf_2026], but targets a different level of the workflow: rather than focusing only on data access
primitives and caching data-responses to in-memory use, it provides an interactive command-line environment for ample
discovery, screening, validating and managing downloads of published ESGF archive products that satisfy scientific
constraints relevant to Earth system and ocean biogeochemistry applications. The software is intended for situations in
which researchers need to move efficiently from broad discovery and catalogue searches to a smaller, analysis-ready
subset of model output that satisfies practical and scientific constraints.

# Key functionality

The software provides several features tailored to archive-scale Earth system data workflows:

- Definition of ESGF search criteria by modifying a [`search_criteria.toml`](scripts/search_criteria.toml) configuration file.
- Automated search through the ESGF catalogue using project, variable, experiment, and temporal filters (inherited 
  from [intake-esgf](https://github.com/esgf2-us/intake-esgf)).
- Ensemble-aware screening mode for selecting one validated `piControl` anchor branch and retaining all compatible `Historical` ensemble members on the same grid.
- Single-branch screening mode for selecting one internally consistent `piControl/Historical` branch pairing per 
  accepted model/grid.
- Combined-variable screening for retaining models that provide all requested variables simultaneously across both 
`piControl` and `historical` runs, producing a single shortlist for downstream download.
- Verification and logging of continuity of date stamps and availability of `piControl` and `Historical` runs on 
  consistent grids (e.g., regular grid `gr` and native grid `gn`).
- Export of simple Dataframes for realised ESGF catalogue searches.
- Verification of server responses and flagging shortlisted ESM outputs as 'Downloadable'.
- Parallelized URL checks for fastest connection in case same data are available across different nodes.
- Parallel batch downloading of files from multiple ESGF nodes with retry and integrity checks.
- Parallel batch search and download of grid cell measures (e.g., `areacello` and `volcello`) with archive snapshot of
  relaxed regex matches for later inspection.
- Local directory layout optimised for downstream analysis.
- Example Jupyter notebook workflow covering TOML setup, catalogue screening, and download preparation.

# Software design and workflow overview

`precog-esgf-intake` implements a staged workflow for archive-scale ESGF data discovery, screening, and retrieval.
Rather than moving directly from catalogue search to cached download, the software separates archive interrogation,
shortlist generation, branch and grid validation, downloadability checks, and file retrieval into distinct command-line
steps, allowing users to inspect and validate intermediate results before proceeding. 

In a typical workflow (Figure 1), the user first edits a user-facing `search_criteria.toml` configuration file to define
the intended ESGF search criteria, including project, activity, experiment, frequency, variables (optional), and grid
labels. This externalised configuration makes search intent easier to reproduce than editing hard-coded dictionaries in
source code, while still allowing interactive entry of `variable_id` when that field is intentionally left
blank. The catalogue search stage then queries ESGF holdings for matching CMIP products (`CMIP6` by default), exports
tabular search summaries for inspection, and applies screening logic to retain
scientifically relevant combinations such as paired `piControl` and `historical` simulations on common valid
grids.

A key feature of the workflow is the ability to **combine** multiple requested variables into a single conditional
shortlist. Rather than screening each variable independently, users can require that the same model provides all
requested variables simultaneously, for both `piControl` and `historical`, before that model progresses to the
downloadable shortlist. This combined screening is particularly useful for multi-field diagnostics where consistent 
co-availability of variables is required for downstream interpretation.

The workflow supports both single-branch and ensemble-aware catalogue screening. In single-branch mode, the 
search retains one internally consistent `piControl`/`historical` branch pairing for each accepted model-grid combination. In
ensemble-aware mode, the workflow first identifies a validated `piControl` anchor branch and then retains all compatible
`historical` ensemble members on the same model/grid pair, provided that the requested variables are complete and
continuity checks are satisfied. Grid-level diagnostics are generated from catalogue
metadata so that invalid model-grid combinations can be rejected before download preparation, and validation outputs are
preserved for later auditing.

Subsequent stages verify whether shortlisted files are reachable on remote ESGF nodes, assign local destination paths,
and download both target variables and required supporting grid-cell measures such as `areacello` and
`volcello`. By treating search results, validation outputs, and downloadable file lists as explicit intermediate
artefacts, the software supports both interactive exploratory use and more reproducible archive-based Earth system workflows in
which dataset selection decisions can be inspected, traced, and repeated at a later time.

![](precog-esgf-intake-diagram.png)
{width=90%}
*Figure 1. precog-esgf-intake toolkit overview and directory structure of an example ESGF download. The top-level
directory
contains search outputs and model-specific CMIP6 data organised by model, experiment, variable, and annual files.*

# Research impact and applications

A representative use case is the identification of CMIP6 models that simultaneously provide both pre-industrial
and historical outputs for ocean biogeochemical variables such as `expc` and `epc100`, together with auxiliary or
supporting variables and the associated grid-cell measures required for downstream analyses. The repository
accompanying this software includes
a [workflow example](https://github.com/LeoBertini/precog-esgf-intake/blob/main/Workflow_Example_POCflux.ipynb)
demonstrating this type of archive screening and retrieval process.

This is particularly relevant for ocean biogeochemistry and carbon-cycle studies, where analyses often
depend on coherent combinations of physical and biogeochemical fields rather than isolated variables (e.g., retrieval of
carbonate system fields as well as ocean state physical variables, [@Wilson2022]). In such cases, the time spent
screening archive holdings, checking consistency, and organising downloads can be substantial, and purpose-built
automation improves both efficiency and reproducibility. This data discovery and retrieval functionality by
`precog-esgf-intake` is also complementary to established CMIP preprocessing packages such as xMIP [@xMIP] and
ESMValTool [@ESMValTool] , which provide tools for harmonising,
cataloguing, and preparing model outputs for analysis; together, these packages support a reproducible workflow from
archive discovery and dataset selection
through to preprocessing and scientific analysis

Therefore, `precog-esgf-intake` fills a workflow gap between catalogue access and scientific analysis. The software
capabilities support workflows in which data access is itself a significant part of the scientific process,
particularly when analyses depend on assembling coherent, scientifically consistent, and analysis-ready
ensemble subsets of Earth system model output from distributed archives.

# AI usage disclosure

AI assistance was used solely to improve readability of selected software documentation (Claude Sonnet 5 by Anthropic).
No AI tools were used to design, implement, or validate the software. All AI-assisted documentation was reviewed and
edited by the authors, who retain full responsibility for its accuracy and content. Authors made all the core design and
architectural decisions.

# Acknowledgements

This work is part of the [PRECOG - Predicting Biological Carbon in the Ocean Globally](https://precog-ocean.github.io)
project, funded by UK Research and Innovation (UKRI) through a Future Leaders Fellowship Award (Project Reference
MR/Y016629/1). The authors thank the developers of `intake-esgf` and the wider ESGF infrastructure for making
climate-model archives programmatically accessible to the research community. `precog-esgf-intake` builds directly on
the core implementation by `intake-esgf`, and this dependency is gratefully acknowledged.

# References

