# Australian Weather Rain Prediction — MLOps Deployment

An end-to-end machine-learning application that predicts whether it will rain tomorrow at an Australian weather location. The repository takes historical weather observations through preprocessing and XGBoost model training, packages the prediction UI as a Flask container, and provides a path to deploy it to Google Kubernetes Engine (GKE) through GitHub Actions.

> Deployment focus: **Docker → Artifact Registry → GKE → GitHub Actions CI/CD**.

## Problem statement

Weather-dependent decisions—such as event planning, farming operations, logistics, and day-to-day travel—benefit from an early, data-driven estimate of rainfall. Manual interpretation of many related weather signals is slow and inconsistent.

This project addresses the binary classification problem:

```text
Given today's weather observations for an Australian location,
predict whether it will rain tomorrow: YES or NO.
```

The model uses information including location, temperature, rainfall, evaporation, sunshine, wind direction and speed, humidity, pressure, cloud cover, whether it rained today, and date-derived features. The Flask interface collects the already-numeric, model-ready feature values and returns a `YES`/`NO` prediction.

### Why this is an MLOps problem

A useful model is more than a notebook result. It needs a reproducible way to:

1. prepare data and train a versioned artifact;
2. run the same serving environment on a laptop and in the cloud;
3. deploy multiple replicas reliably;
4. expose the application safely; and
5. rebuild and roll out a new image whenever approved code reaches `main`.

This repository supplies the building blocks for that lifecycle.

## What is in the repository

| Area | Location | Purpose |
| --- | --- | --- |
| Web application | `application.py` | Flask server, model loading, form handling, and prediction response |
| User interface | `templates/index.html`, `static/style.css` | Browser form for the model features |
| Data preparation | `src/data_processing.py` | CSV loading, date features, missing-value handling, encoding, and train/test split |
| Model training | `src/model_training.py` | XGBoost training, metrics logging, and model serialization |
| Training entry point | `pipeline/training_pipeline.py` | Runs processing followed by training |
| Model/data artifacts | `artifacts/` | Raw input, processed splits, and the serving model (created by the pipeline) |
| Container build | `Dockerfile` | Builds the Flask serving image |
| Kubernetes runtime | `kubernetes-deployment.yaml` | Two-replica Deployment and public LoadBalancer Service |
| Continuous delivery | `.github/workflows/deploy.yml` | Builds, pushes, and rolls out an image after a push to `main` |

## Architecture

```text
Historical weather CSV
        │
        ▼
DataProcessing ──► processed train/test artifacts ──► ModelTraining
                                                        │
                                                        ▼
                                              artifacts/models/model.pkl
                                                        │
                                                        ▼
Developer / GitHub Actions ──► Docker image ──► Artifact Registry
                                                        │
                                                        ▼
Internet user ──► GKE LoadBalancer ──► Service ──► 2 Flask pods ──► prediction
```

The container serves the model file at `artifacts/models/model.pkl`. Therefore, an image must contain a compatible, trained model before it is deployed.

## Model workflow

### Input and output

The training dataset is read from `artifacts/raw/data.csv`. `DataProcessing`:

- converts `Date` into `Year`, `Month`, and `Day`;
- fills missing numerical values with column means;
- drops remaining incomplete rows;
- label-encodes `Location`, wind-direction columns, `RainToday`, and `RainTomorrow`; and
- creates an 80/20 train/test split using `random_state=42`.

`ModelTraining` trains an `xgboost.XGBClassifier`, writes `artifacts/models/model.pkl`, and logs accuracy, weighted precision, weighted recall, and weighted F1 score.

Run training locally from the repository root:

```bash
python -m pipeline.training_pipeline
```

The app expects the feature order below. Keep this order unchanged when retraining or integrating a programmatic client:

```text
Location, MinTemp, MaxTemp, Rainfall, Evaporation, Sunshine,
WindGustDir, WindGustSpeed, WindDir9am, WindDir3pm, WindSpeed9am,
WindSpeed3pm, Humidity9am, Humidity3pm, Pressure9am, Pressure3pm,
Cloud9am, Cloud3pm, Temp9am, Temp3pm, RainToday, Year, Month, Day
```

## Local development

Prerequisites: Python compatible with the project metadata, `pip`, and the data/model artifacts.

```bash
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\\Scripts\\activate
pip install -r requirements.txt
python -m pipeline.training_pipeline  # only needed to create/update model.pkl
python application.py
```

Open `http://localhost:5000` and provide numeric encoded values for all fields. The current UI intentionally mirrors the model’s encoded input contract; it does not translate human-readable location or wind labels.

## Docker deployment

Docker makes the serving runtime portable: the same image tested locally is the artifact promoted to GKE.

### Build and run locally

First ensure `artifacts/models/model.pkl` exists. Then:

```bash
docker build -t australian-weatherprediction:local .
docker run --rm -p 5000:5000 australian-weatherprediction:local
```

Visit `http://localhost:5000`.

Useful checks:

```bash
docker image inspect australian-weatherprediction:local
docker logs <container-id>
```

### Tag and publish to Artifact Registry

The Kubernetes manifest is currently configured for Artifact Registry repository `my-repo` in `us-central1` and project `mlops-new-483907`. Replace these sample/project-specific values with your own project values consistently in the manifest and workflow.

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="us-central1"
export REPOSITORY="my-repo"
export IMAGE_NAME="mlops-app"
export TAG="$(git rev-parse --short HEAD)"

gcloud auth login
gcloud config set project "$PROJECT_ID"
gcloud auth configure-docker "$REGION-docker.pkg.dev"

docker build -t "$REGION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE_NAME:$TAG" .
docker push "$REGION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE_NAME:$TAG"
```

Never use an unreviewed `latest` image for a production rollout. An immutable Git SHA tag gives every GKE revision a traceable source commit and makes rollback dependable.

## Kubernetes / GKE deployment

`kubernetes-deployment.yaml` declares:

- a Deployment named `mlops-app` with **two replicas**;
- a matching `app: mlops-app` selector/label pair;
- container port `5000`; and
- a `LoadBalancer` Service named `mlops-service` that maps public port `80` to the Flask port `5000`.

### Create the GCP infrastructure

Enable required APIs and create the container registry repository:

```bash
gcloud services enable container.googleapis.com artifactregistry.googleapis.com
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Australian WeatherPrediction images"

gcloud container clusters create my-cluster \
  --region=us-central1 \
  --num-nodes=2

gcloud container clusters get-credentials my-cluster \
  --region=us-central1
```

For a real environment, prefer a private GKE cluster, Workload Identity, a dedicated namespace, and least-privilege service accounts. The commands above are a simple starting point, not a production security baseline.

### Apply and verify

The manifest’s image field defaults to `:latest`; deploy a specific published version instead:

```bash
kubectl apply -f kubernetes-deployment.yaml
kubectl set image deployment/mlops-app \
  mlops-app=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/mlops-app:$TAG

kubectl rollout status deployment/mlops-app
kubectl get pods -l app=mlops-app
kubectl get service mlops-service --watch
```

When `EXTERNAL-IP` is assigned, open `http://EXTERNAL-IP/`.

### Operational commands

```bash
# Inspect rollout and application output
kubectl describe deployment mlops-app
kubectl logs deployment/mlops-app --tail=100

# Inspect the public endpoint
kubectl get svc mlops-service

# Roll back the immediately preceding release
kubectl rollout undo deployment/mlops-app
kubectl rollout status deployment/mlops-app
```

Deleting the Service stops the cloud load balancer and its associated public IP. Delete only resources you intend to tear down:

```bash
kubectl delete -f kubernetes-deployment.yaml
```

## CI/CD with GitHub Actions and GCP

The workflow at `.github/workflows/deploy.yml` is triggered by pushes to `main`. Its intended release sequence is:

```text
push to main
  → authenticate to Google Cloud
  → build image tagged with Git SHA
  → push image to Artifact Registry
  → fetch GKE credentials
  → apply Kubernetes manifest
  → update Deployment to the Git-SHA image
  → Kubernetes rolling rollout
```

### Required GitHub repository secrets

| Secret | Required value |
| --- | --- |
| `GCP_PROJECT_ID` | Target Google Cloud project ID |
| `GCP_SA_KEY` | Service-account JSON credential used by the current workflow |

The CI service account needs permissions to push to Artifact Registry and deploy to GKE. Start with roles scoped to the target project/cluster, such as Artifact Registry Writer, Kubernetes Engine Developer, and the permissions needed to obtain cluster credentials. Restrict and review these permissions for your organization.

### Pre-deployment checklist

Before enabling automatic deployment, confirm all of the following:

- the Artifact Registry repository, GKE cluster, region, image path, and Kubernetes manifest use the same project/region/repository values;
- the image contains `artifacts/models/model.pkl` and it loads successfully;
- GitHub secrets are present and the CI identity is authorized;
- the deploy image is immutable (Git SHA) rather than relying on `latest`;
- a deployment has readiness/liveness probes and resource requests/limits; and
- a test stage validates the application before an image is promoted.

## Important current deployment notes

This repository includes the deployment skeleton, but it should be hardened before treating it as production-ready:

1. **Python versions need alignment.** `pyproject.toml` specifies Python `>=3.14`, while the Dockerfile uses Python 3.12 Alpine. Align the project requirement and base image, then rebuild the image. Otherwise editable installation can fail.
2. **The current GitHub job runs inside `google/cloud-sdk`.** That image is not a Docker daemon environment. The `docker build`/`docker push` step requires a Docker-capable runner or a replacement such as Cloud Build/Buildx. Validate this before relying on the workflow.
3. **No CI test gate exists yet.** Add unit/integration tests and run them before building and deploying an image.
4. **The GKE manifest is minimal.** Add a namespace, resource requests/limits, non-root security context, a pinned image strategy, rolling-update settings, and readiness/liveness probes. Flask’s development server should be replaced with a production WSGI server such as Gunicorn for production traffic.
5. **Avoid long-lived JSON keys where possible.** The current workflow uses `GCP_SA_KEY`; GitHub-to-GCP Workload Identity Federation is the preferred production authentication approach because it avoids storing a persistent service-account key.
6. **Model compatibility is part of every release.** The UI accepts numeric label encodings, so the preprocessing mappings must stay synchronized with the saved model. Persisting preprocessing artifacts/pipelines alongside the model is recommended.

## Recommended release workflow

1. Make a code, dependency, or model change on a branch.
2. Run training and tests locally; verify the application can load the generated model.
3. Build and smoke-test the Docker image locally.
4. Merge a reviewed pull request to `main`.
5. Let CI build an image tagged with the commit SHA, push it to Artifact Registry, and update GKE.
6. Watch `kubectl rollout status deployment/mlops-app`; verify the load-balancer endpoint.
7. If verification fails, use `kubectl rollout undo deployment/mlops-app`, investigate logs, and release a corrected immutable image.

## Troubleshooting

| Symptom | Likely cause | Check / response |
| --- | --- | --- |
| `FileNotFoundError` for `model.pkl` | Training artifact was not created or copied into the image | Run the training pipeline and confirm `artifacts/models/model.pkl` before `docker build` |
| Pod is `ImagePullBackOff` | Incorrect image path/tag or missing registry access | `kubectl describe pod <pod>`; verify Artifact Registry image and GKE node permissions |
| Service has no external IP | Load balancer is still provisioning or cloud quota/network issue | `kubectl get svc mlops-service --watch` and inspect Service events |
| CI fails at `docker build` | Runner/job container has no Docker daemon | Use a Docker-capable GitHub runner or Cloud Build/Buildx |
| Container build rejects Python version | Dockerfile Python version conflicts with project metadata | Make Dockerfile and `requires-python` consistent |
| Form always produces errors/unknown results | Inputs are not numeric or preprocessing/model mismatch | Use model-ready encoded values and retrain/ship preprocessing artifacts together |

## License

No license has been declared for this repository. Add an explicit license before distributing or accepting external contributions.
