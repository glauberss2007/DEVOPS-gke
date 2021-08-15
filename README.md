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

The frontend and backend modes support two additional URLs:
- version prints the running version.
- healthz reports the application's health. In frontend mode, the health displays as OK if the backend is reachable.

## Deploying the sample app to Kubernetes

Deploy the gceme frontend and backend to Kubernetes using manifest files that describe the deployment environment. The files use a default image that is updated later in this tutorial.

Deploy the applications into two environments.
- Production. The live site that your users access.
- Canary. A smaller-capacity site that receives a percentage of your user traffic. Use this environment to sanity check your software with live traffic before it's released to the live environment.

First, deploy your application into the production environment to seed the pipeline with working code.

1. Create the Kubernetes namespace to logically isolate the production deployment:
```
kubectl create ns production
```
2. Create the canary and production deployments and services:
```
kubectl --namespace=production apply -f k8s/production
kubectl --namespace=production apply -f k8s/canary
kubectl --namespace=production apply -f k8s/services
```
3. Scale up the production environment frontends:
```
kubectl --namespace=production scale deployment gceme-frontend-production --replicas=4
```
4. Retrieve the external IP for the production services. It can take several minutes before you see the load balancer IP address.
```
kubectl --namespace=production get service gceme-frontend
```

When the process completes, an IP address is displayed in the EXTERNAL-IP column.

5. Store the frontend service load balancer IP in an environment variable:
```
export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}"  --namespace=production services gceme-frontend)
```

6. Confirm that both services are working by opening the frontend external IP address in your browser.

7. Open a separate terminal and poll the production endpoint's /version URL so you can observe rolling updates in the next section:
```
while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1;  done
```



