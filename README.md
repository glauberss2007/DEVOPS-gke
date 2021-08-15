# gke-jenkins
Continuous deployment to Google Kubernetes Engine using Jenkins

This tutorial shows you how to set up a continuous delivery pipeline using Jenkins and Google Kubernetes Engine (GKE), as described in the following diagram.

![image](https://user-images.githubusercontent.com/22028539/129487666-a4f29547-ef82-43d9-bf9e-c7f24111b876.png)

## Objectives
1. Understand a sample application.
2. Deploy an application to GKE.
3. Upload code to Cloud Source Repositories.
4. Create deployment pipelines in Jenkins.
5. Deploy development environments.
5. Deploy a canary release.
6. Deploy production environments.

## Costs
This tutorial uses billable components of Google Cloud, including:
- Compute Engine
- Google Kubernetes Engine
- Cloud Build

[Pricing Calculator](https://cloud.google.com/products/calculator#id=7ecbf600-bb93-4d6d-8381-d986f33dc9a0)

## Create the project
1. In the Google Cloud Console, on the project selector page, select or create a Google Cloud project.
2. Make sure that billing is enabled for your Cloud project.
3. Enable the Compute Engine, GKE, and Cloud Build APIs.

Preparing your environment
1. Complete the [Setting up Jenkins on GKE](https://cloud.google.com/architecture/jenkins-on-kubernetes-engine-tutorial) tutorial. Ensure that you have a working Jenkins install running in GKE.
2. In Cloud Shell, clone the sample code:
```
git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes
cd continuous-deployment-on-kubernetes/sample-app
```
3. Apply the cluster-admin role to the Jenkins service account:
```
kubectl create clusterrolebinding jenkins-deploy \
    --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins
```
PS: In this tutorial, the Jenkins service account needs cluster-admin permissions so that it can create Kubernetes namespaces and any other resources that the app requires. For production use, you should catalog the individual permissions necessary and apply them to the service account individually.

## Understanding the application

You'll deploy the sample application, gceme, in your continuous deployment pipeline. The application is written in the Go language, and is located in the repository's sample-app directory. When you run the gceme binary on a Compute Engine instance, the app displays the instance's metadata in an info card.

The application mimics a microservice by supporting two operation modes:
- In backend mode, gceme listens on port 8080 and returns Compute Engine instance metadata in JSON format.
- In frontend mode, gceme queries the backend gceme service, and renders the resulting JSON in the user interface.

![image](https://user-images.githubusercontent.com/22028539/129488515-2751867a-fa14-4bf9-a166-80fcc464e959.png)


