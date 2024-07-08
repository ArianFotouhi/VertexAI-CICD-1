# CI/CD Pipeline for Vertex AI Model - Bikeshare Prediction

The repository comprises the code and CI/CD pipeline configuration for training and deploying a Bikeshare prediction model on Google Cloud Vertex AI.

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

## Steps:

1. **Create a Vertex AI instance and use Jupyter Notebook**:
   - Upload your files into the Jupyter Notebook as shown in the image.
![CC Image](https://github.com/ArianFotouhi/VertexAI-CICD-1/blob/main/assets/vertexai-1.png?raw=true)

2. **Create a bucket and upload dataset files**:
   - Create a Google Cloud Storage bucket.
   - Upload your dataset files into the bucket.
![CC Image](https://github.com/ArianFotouhi/VertexAI-CICD-1/blob/main/assets/buckets.png?raw=true)

3. **Submit a build job**:
   - Go to Vertex AI.
   - Open a terminal in your instance.
   - Submit a build job (CICD job) from the terminal using
```
gcloud builds submit --region us-central1
```
![CC Image](https://github.com/ArianFotouhi/VertexAI-CICD-1/blob/main/assets/vertexai-2.png?raw=true)
4. **Monitor job progress**:
   - Use Google Cloud Build to observe the job progress.
![CC Image](https://github.com/ArianFotouhi/VertexAI-CICD-1/blob/main/assets/builds.png?raw=true)


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
```

## Test Script
The test script test-training.py contains test cases for validating the training and preprocessing logic.

### Tests Include:
**Model Name Validation: Ensures invalid model names raise an error.
**Model Training Validation: Verifies model training and checks RMSE.
**Model Artifact Validation: Confirms the model artifact is saved in the Cloud Storage bucket.
**Data Preprocessing Validation: Checks the correctness of preprocessing output.

## Model Training Script
The model-training-code.py script contains functions to load data, preprocess data, train the model, and save the model artifact.

### Key Functions:
**Data Loading: Reads data from a CSV file in a Google Cloud Storage bucket.
**Data Preprocessing: Prepares the feature matrix and target variable.
**Model Training: Trains a RandomForestRegressor model.
**Model Artifact Saving: Saves the trained model and uploads it to Cloud Storage.



