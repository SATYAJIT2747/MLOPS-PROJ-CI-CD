# MLOPS-PROJ-CI-CD

End-to-end MLOps project for sentiment analysis, built around experiment tracking, data versioning, automated testing, containerization, Kubernetes deployment on EKS, and observability with Prometheus and Grafana.

This README documents what was built, how the system flows, how to run it, and the proof artifacts that show the deployment working in AWS.

## Project Flow

This is the main flow of the project from experimentation to deployment:

![Project flow diagram](EKS%20RUNS_img/project_structure.png)


The project takes a text input, normalizes it, converts it to features, runs the latest approved model, and serves the prediction through a Flask web app. The model lifecycle is handled with MLflow and DagsHub, the data and pipeline are managed with DVC, and the production deployment runs on AWS EKS behind a LoadBalancer service.

## Project Structure

The repository currently includes:

- Experiment notebooks for model exploration and comparison.
- A DVC pipeline for ingestion, preprocessing, feature generation, model building, evaluation, and registration.
- A Flask application for inference.
- Docker packaging for local and production use.
- GitHub Actions CI/CD for testing, building, pushing, and deploying.
- Kubernetes deployment and service manifests for EKS.
- Prometheus and Grafana monitoring setup.

The high-level execution path is:

1. Raw data is prepared and versioned through DVC.
2. Data is cleaned and transformed into model-ready features.
3. Experiments are tracked in MLflow via DagsHub.
4. The selected model is registered and promoted.
5. The Flask app loads the latest staged model and serves predictions.
6. GitHub Actions runs tests, rebuilds the container, pushes to ECR, and applies the Kubernetes manifests.
7. EKS exposes the app using a LoadBalancer service.
8. Prometheus scrapes request metrics and Grafana visualizes traffic and latency.

## What Was Done

### 1. Project scaffolding

- Created the repository structure using a data-science style project layout.
- Organized the code under src with separate packages for data, features, model, logger, and visualization.
- Added test modules and helper scripts for CI and promotion.

### 2. Data and model pipeline

- Built data ingestion and preprocessing steps.
- Added feature engineering and model-building code.
- Added evaluation and model registration steps.
- Wired the flow through dvc.yaml and params.yaml so the pipeline can be reproduced consistently.

### 3. Experiment tracking and model registry

- Connected experiment runs to DagsHub and MLflow.
- Tracked notebooks and training runs for comparison.
- Loaded the production app model from the MLflow registry instead of hardcoding a local pickle model as the source of truth.

### 4. Flask inference service

- Built a simple web interface for sentiment prediction.
- Added text preprocessing inside the app before inference.
- Exposed a prediction endpoint and a Prometheus metrics endpoint.
- Added request count, latency, and prediction counters for observability.

### 5. Docker and containerization

- Packaged the Flask app into a production container.
- Exposed port 5000 and used Gunicorn for the deployment command.
- Copied the trained vectorizer into the image so the app can transform text consistently at runtime.

### 6. CI/CD with GitHub Actions

- Added a workflow that installs dependencies, verifies AWS CLI, runs DVC, executes tests, promotes the model, builds the Docker image, pushes it to ECR, and deploys to EKS.
- Used GitHub Secrets for AWS credentials and the DagsHub token.

### 7. AWS and EKS deployment

- Created an EKS cluster and node group.
- Applied a Kubernetes deployment with 2 replicas.
- Exposed the app through a LoadBalancer service.
- Stored the DagsHub token as a Kubernetes secret and injected it into the app at runtime.

### 8. Monitoring

- Added Prometheus scraping for the app metrics.
- Created Grafana dashboards to visualize request traffic and latency over time.

## Repository Layout

- src/data: ingestion and preprocessing logic.
- src/features: feature engineering and feature building.
- src/model: model building, evaluation, and registration.
- flask_app: web application, templates, and runtime helpers.
- tests: unit tests for the model and Flask app.
- scripts: utility scripts, including model promotion.
- dvc.yaml: reproducible pipeline definition.
- params.yaml: model and pipeline parameters.
- deployment.yaml: Kubernetes deployment and service manifest.
- .github/workflows/ci.yaml: CI/CD workflow.

## Local Setup

### 1. Create the environment

	conda create -n atlas python=3.10
	conda activate atlas

### 2. Install dependencies

	pip install -r requirements.txt

If you are working with the Flask app separately, install its local requirements too:

	cd flask_app
	pip install -r requirements.txt

### 3. Run the DVC pipeline

	dvc repro
	dvc status

### 4. Run the Flask app locally

	cd flask_app
	python app.py

The app runs on port 5000.

### 5. Build and run the Docker image

	docker build -t capstone-app:latest .
	docker run -p 8888:5000 -e CAPSTONE_TEST=your_token_here capstone-app:latest

Replace the token with your own secret value. Do not commit real credentials.

## AWS and Deployment Setup

Before deployment, configure the following according to your own account:

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_REGION
- AWS_ACCOUNT_ID
- ECR_REPOSITORY
- CAPSTONE_TEST as a GitHub Secret and Kubernetes Secret

The application and workflow expect the model registry auth token to be available as CAPSTONE_TEST or dagshub_token.

### EKS flow

1. Build the Docker image.
2. Push the image to ECR.
3. Update the Kubernetes manifest with the correct ECR image path.
4. Create or update the cluster kubeconfig.
5. Apply the deployment and service manifest.
6. Wait for the LoadBalancer to expose an external endpoint.
7. Open the app in the browser and test predictions.

The Kubernetes manifest includes:

- 2 Flask replicas.
- Container port 5000.
- A LoadBalancer service on port 5000.
- An injected CAPSTONE_TEST secret.

## Monitoring

Prometheus scrapes the Flask metrics endpoint and Grafana visualizes the traffic.

The application exposes custom metrics for:

- request count
- request latency
- prediction count by class

Metrics and dashboard evidence:

![Grafana dashboard panel](EKS%20RUNS_img/Screenshot%20%281051%29.png)

![Grafana dashboard overview](EKS%20RUNS_img/Screenshot%20%281052%29.png)

## Proof Of Work

The following screenshots show the main pieces of the deployment and runtime validation.

### Deployed app: positive prediction

![Positive sentiment result](EKS%20RUNS_img/Screenshot%20%281029%29.png)

### Deployed app: negative prediction

![Negative sentiment result](EKS%20RUNS_img/Screenshot%20%281031%29.png)

### Kubernetes state

![kubectl pods and service](EKS%20RUNS_img/Screenshot%20%281030%29.png)

### Grafana monitoring

![Grafana edit view](EKS%20RUNS_img/Screenshot%20%281051%29.png)

![Grafana dashboard](EKS%20RUNS_img/Screenshot%20%281052%29.png)

### S3 bucket proof

![S3 bucket objects](EKS%20RUNS_img/Screenshot%20%281054%29.png)

## Cleanup

When you are done, clean up the AWS resources to avoid charges:

	kubectl delete deployment flask-app
	kubectl delete service flask-app-service
	kubectl delete secret capstone-secret
	eksctl delete cluster --name flask-app-cluster --region us-east-1

Also remove any unused ECR images, S3 objects, and CloudFormation stacks tied to the cluster.

## Notes

- Replace the hardcoded account details in the code and workflow with your own values before using this in a different AWS or DagsHub account.
- Keep all real secrets in GitHub Secrets, AWS Secrets Manager, or Kubernetes Secrets.
- The project flow in this repository matches the implementation shown in the screenshot evidence folder.