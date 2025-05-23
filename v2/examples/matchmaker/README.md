# Development

This example matchmaker is a golang application designed to be built using [CNCF Buildpacks](https://www.cncf.io/projects/buildpacks/), and our primary build process specifically uses [Google Cloud's buildpacks](https://cloud.google.com/docs/buildpacks/overview). It requires the use of Open Match 2, and won't startup if it can't contact an OM2 instance.

## Build

This repo includes the files required to build [remotely using Cloud Build on Google Cloud](https://cloud.google.com/docs/buildpacks/build-application#remote_builds).  A typical build goes something like this, with the repo cloned,  `gcloud` initialized, and an existing [Docker Artifact Registry](https://cloud.google.com/artifact-registry/docs/docker/store-docker-container-images) called `myregistry`:
```
# Update the dependencies to the latest versions and kick off a Cloud Build
go get && go mod tidy && gcloud builds submit --async --substitutions=COMMIT_SHA=abc123
```
There is no `cloudbuild.yaml` or `Dockerfile` required for this. [Google Cloud's buildpacks](https://cloud.google.com/docs/buildpacks/overview) takes care of everything.

## Deploy
The `cloudbuild.yaml` file contains steps to deploy a copy of `matchmaker-example` service to Cloud Run in Google Cloud. 
This file should be populated with the following:
* Your 
* Your Service Account created for `matchmaker-example`. The section below has steps on how to create it with the proper IAM roles.
* Your `om-core` instance IP address that can be reached from Cloud Run. We test against `om-core` deployed in Cloud Run.
  
With the `service.yaml` file populated with everything you've configured, you can deploy the service with a command like this:
```
gcloud run services replace service.yaml
```
You may need to adjust the scaling configuration and amount of memory `om-core` is allowed to use depending on your matchmaker.

`core` is just a gRPC/HTTP golang application in a container image and can be deployed in most environments (local Docker/Minikube/Kubernetes/kNative) with a little effort. 

## Service Account
Best practices dictate that you deploy applications to Cloud Run with minimal IAM permissions. This creates a service account named `matchmaker-example-identity` that only has Cloud Run Invoker permissions (so it can talk to `om-core` running in Cloud Run). This service account is usable only by you and by the default Cloud Build service account.

```
export PROJECT_ID=$(gcloud config get project)
export PROJECT_NUMBER=$(gcloud projects list --filter=$(gcloud config get project) --format="value(PROJECT_NUMBER)")
export GCLOUD_ACCOUNT=$(gcloud config get account)

gcloud iam service-accounts create matchmaker-example-identity \
  --display-name="Example Matchmaker Service Account"

gcloud iam service-accounts add-iam-policy-binding \
  matchmaker-example-identity@${PROJECT_ID}.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountUser \
  --member=user:${GCLOUD_ACCOUNT}

gcloud iam service-accounts add-iam-policy-binding \
  matchmaker-example-identity@${PROJECT_ID}.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountUser \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member=serviceAccount:matchmaker-example-identity@${PROJECT_ID}.iam.gserviceaccount.com \
--role=roles/run.invoker
```

In addition, if you are trying to deploy using the step in the `cloudbuild.yaml`, your Cloud Build service account (`<PROJECT_NUMBER>-compute@developer.gserviceaccount.com`) will need permission to deploy to Cloud Run. This is not necessary if you are just manually deploying the built container image to Cloud Run using the Cloud Console UI or the `gcloud` CLI.

## Using the example matchmaker
The behavior of the example matchmaker can be controlled with environment variables. Here are some variables you may want to set in the `cloudbuild.yaml` deploy step or by using a [manual gcloud command](https://cloud.google.com/run/docs/configuring/services/environment-variables#setting):

