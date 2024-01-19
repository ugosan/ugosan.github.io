---
layout: post
title: Using virtualenv on Jupyter Notebook
excerpt: Using virtualenv on Jupyter Notebook
tags: virtualenv Jupyter Notebook
---



1. Create a virtual environment: `virtualenv venv`

2. Activate it: `source ./venv/bin/activate`

3. Install ipykernel: `python -m pip install ipykernel`
    
4. Add a Jupyter kernel: `python -m ipykernel install --user --name venv`
    
5. Launch Jupyter Notebook: `jupyter notebook`

```
virtualenv venv
source ./venv/bin/activate
python -m pip install ipykernel
python -m ipykernel install --user --name venv
jupyter notebook

```