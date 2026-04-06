# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Dockerized ML inference API for an advertising sales prediction model. It exposes a Flask REST API that serves predictions from a pre-trained scikit-learn linear regression model.

## Commands

### Train the model (run locally before building the image)
```bash
python model.py
```
This reads `data/Advertising.csv`, trains a pipeline (imputer + scaler + LinearRegression), evaluates it, and serializes it to `ad_model.pkl`.

### Run the API locally (without Docker)
```bash
python app.py
```
Starts Flask on port 5000.

### Build and run with Docker
```bash
docker build -t taller-docker .
docker run -p 5000:5000 taller-docker
```

### Call the API endpoints
```bash
# Health check
curl http://localhost:5000/

# Predict sales from ad spend
curl "http://localhost:5000/api/v1/predict?tv=100&radio=20&newspaper=10"

# Retrain on new data (requires data/Advertising_new.csv to exist)
curl http://localhost:5000/api/v1/retrain
```

## Architecture

**Two-stage design:** model training is decoupled from serving.

- `model.py` — offline training script. Reads `data/Advertising.csv`, builds a `sklearn.pipeline.Pipeline` (mean imputer → standard scaler → LinearRegression), runs 4-fold cross-validation, then fits on the full dataset and pickles the pipeline to `ad_model.pkl`.

- `app.py` — Flask API that loads `ad_model.pkl` at startup (global `model` variable). Three routes:
  - `GET /` — health check
  - `GET /api/v1/predict?tv=&radio=&newspaper=` — runs inference; missing params are imputed by the pipeline's `SimpleImputer`
  - `GET /api/v1/retrain` — hot-reloads the global model by re-fitting on `data/Advertising_new.csv` (in-memory only; pickle serialization is commented out)

- `ad_model.pkl` — the serialized pipeline artifact; must exist before `app.py` starts. It is excluded from `.dockerignore` so it is baked into the image at build time. `model.py` itself is excluded from the Docker image.

- `data/` — contains `Advertising.csv` (training) and `Advertising_new.csv` (retrain endpoint). Both are included in the image. Features are `tv`, `radio`, `newspaper`; target is `sales`.

**Port:** the app listens on `PORT` env var, defaulting to `5000`. Docker Compose maps host `5000` → container `5000`.
