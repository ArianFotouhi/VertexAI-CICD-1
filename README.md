# CI/CD Pipeline for Vertex AI Model - Bikeshare Prediction

This repository contains the code and CI/CD pipeline configuration for training and deploying a Bikeshare prediction model on Google Cloud Vertex AI.

## Overview

The CI/CD pipeline automates the process of building, testing, and deploying a machine learning model using Google Cloud services. The key steps include:

1. **Build Docker Image**: Creates a Docker image with the model training code.
2. **Push Docker Image to GCR**: Uploads the Docker image to Google Container Registry (GCR).
3. **Execute Tests**: Runs tests to ensure the correctness of the training code.
4. **Submit Training Job**: Submits a custom training job to Google Cloud Vertex AI.
5. **Upload Model**: Uploads the trained model to Vertex AI.
6. **Fetch Model ID**: Retrieves the ID of the uploaded model.
7. **Create Endpoint**: Creates an endpoint for serving the model.
8. **Deploy Model to Endpoint**: Deploys the model to the created endpoint.

## Pipeline Configuration

The pipeline configuration is defined in a YAML file used by Google Cloud Build to execute the steps.

### Steps

```yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  id: 'Build Docker Image'
  args: ['build', '-t', 'gcr.io/storied-polymer-427814-e8/cicd-vertex-bikeshare-model', '.']

- name: 'gcr.io/cloud-builders/docker'
  id: 'Push Docker Image To GCR'
  args: ['push', 'gcr.io/storied-polymer-427814-e8/cicd-vertex-bikeshare-model']

- name: 'gcr.io/storied-polymer-427814-e8/cicd-vertex-bikeshare-model'
  id: 'Execute Tests'
  entrypoint: 'bash'
  args:
   - '-c'
   - |
      pytest test-training.py
      
- name: 'gcr.io/cloud-builders/gcloud'
  id: 'Submit Training Job'
  args: ['ai', 'custom-jobs', 'create', '--region=us-central1', '--project=storied-polymer-427814-e8', '--worker-pool-spec=replica-count=1,machine-type=n1-standard-4,container-image-uri=gcr.io/storied-polymer-427814-e8/cicd-vertex-bikeshare-model', '--display-name=bike-sharing-model-training']
  
- name: 'gcr.io/cloud-builders/gcloud'
  id: 'Upload Model'
  args: ['ai', 'models', 'upload',
         '--container-image-uri=us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-0:latest',
         '--description=bikeshare-new-model',
         '--display-name=bikeshare-new-model',
         '--artifact-uri=gs://clfunctions002/bike-share-rf-regression-artifact/',
         '--project=storied-polymer-427814-e8',
         '--region=us-central1']

- name: 'gcr.io/cloud-builders/gcloud'
  id: 'Fetch Model ID'
  entrypoint: 'bash'
  args: ['-c', 'gcloud ai models list --region=us-central1 --project=storied-polymer-427814-e8 --format="get(MODEL_ID)" --sort-by="createTime" --limit=1 > /workspace/model_id.txt']

- name: 'gcr.io/cloud-builders/gcloud'
  id: 'Create Endpoint'
  entrypoint: 'bash'
  args: ['-c', 'gcloud beta ai endpoints create --display-name=bikeshare-model-endpoint-1 --format="get(name)" --region=us-central1 --project=storied-polymer-427814-e8 > /workspace/endpoint_id.txt']

- name: 'gcr.io/cloud-builders/gcloud'
  id: 'Deploy Model Endpoint'
  entrypoint: 'bash'
  args: 
  - '-c'
  - |
    gcloud beta ai endpoints deploy-model $(cat /workspace/endpoint_id.txt) --region=us-central1 --model=$(cat /workspace/model_id.txt) --display-name=bikeshare-model-endpoint --traffic-split=0=100 --machine-type=n1-standard-4