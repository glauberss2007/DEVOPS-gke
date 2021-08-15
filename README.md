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

## Creating a repository to host the sample app source code

Next, create a copy of the gceme sample app and push it to Cloud Source Repositories.

1. Create the repository in Cloud Source Repositories:
```
gcloud source repos create gceme
```
2. Initialize the local Git repository:
```
git init
git config credential.helper gcloud.sh
export PROJECT_ID=$(gcloud config get-value project)
git remote add origin https://source.developers.google.com/p/$PROJECT_ID/r/gceme
```
3. Set the username and email address for your Git commits in this repository. Replace [EMAIL_ADDRESS] with your Git email address, and [USERNAME] with your Git username.
```
git config --global user.email "[EMAIL_ADDRESS]"
git config --global user.name "[USERNAME]"
```
4. Add, commit, and push the files:
```
git add .
git commit -m "Initial commit"
git push origin master
```

## Creating a pipeline

Use Jenkins to define and run a pipeline for testing, building, and deploying your copy of gceme to your Kubernetes cluster.

### Add your service account credentials

Configure your credentials to allow Jenkins to access the code repository.

1. In the Jenkins user interface, Click Credentials in the left navigation.

2. Click the Jenkins link in the Credentials table.

3. Click Global Credentials.

4. Click Add Credentials in the left navigation.

5. Select Google Service Account from metadata from the Kind drop-down list.

6. Click OK.

There are now two global credentials. Make a note of second credential's name for use later on in this tutorial.

### Create a Jenkins job
Next, use the Jenkins Pipeline feature to configure the build pipeline. Jenkins Pipeline files are written using a Groovy-like syntax.

Navigate to your Jenkins user interface and follow these steps to configure a Pipeline job.

1. Click the Jenkins link in the top left of the interface.
2. Click the New Item link in the left navigation.
3. Name the project sample-app, choose the Multibranch Pipeline option, and then click OK.
4. On the next page, click Add Source and select git.
5. Paste the HTTPS clone URL of your sample-app repository in Cloud Source Repositories into the Project Repository field. Replace [PROJECT_ID] with your project ID.
```
https://source.developers.google.com/p/[PROJECT_ID]/r/gceme
```
6. From the Credentials drop-down list, select the name of the credentials you created when adding your service account.
7. In the Scan Multibranch Pipeline section, select the Periodically if not otherwise run box. Set the Interval value to '1 minute'.
8. Click Save.

After you complete these steps, a job named "Branch indexing" runs. This meta-job identifies the branches in your repository and ensures changes haven't occurred in existing branches. If you refresh Jenkins, the master branch displays this job.

The first run of the job fails until you make a few code changes in the next step.

### Modify the pipeline definition

Create a branch for the canary environment, called canary.
```
git checkout -b canary
```

The Jenkinsfile container that defines that pipeline is written using the Jenkins Pipeline Groovy syntax. Using a Jenkinsfile allows an entire build pipeline to be expressed in a single file that lives alongside your source code. Pipelines support powerful features like parallelization and requiring manual user approval.

Modify the Jenkinsfile so that it contains your project ID on line 1.

### Deploying a canary release

Now that your pipeline is configured properly, you can make a change to the gceme application and let your pipeline test, package, and deploy it.

The canary environment is set up as a canary release. As such, your change is released to a small percentage of the pods behind the production load balancer. You accomplish this in Kubernetes by maintaining multiple deployments that share the same labels. For this application, the gceme-frontend services load balance across all pods that have the labels app: gceme and role: frontend. The k8s/frontend-canary.yaml canary manifest file sets the replicas to 1 and includes labels required for the gceme-frontend service.

Currently, you have 1 out of 5 of the frontend pods running the canary code while the other 4 are running the production code. This helps ensure that the canary code doesn't negatively affect users before rolling out to your full fleet of pods.

1. Open html.go and replace the two instances of blue with orange.
2. Open main.go and change the version number from 1.0.0 to 2.0.0:
```
const version string = "2.0.0"
```
3. Next, add and commit those files to your local repository:

4. Finally, push your changes to the remote Git server:

5. After the change is pushed to the Git repository, navigate to the Jenkins user interface where you can see that your build started.

6. After the build is running, click the down arrow next to the build in the left navigation and select Console Output.

7. Track the output of the build for a few minutes and watch for the kubectl --namespace=production apply... messages to begin. When they start, check back on the terminal that was polling the production /version URL and observe begin to change in some of the requests. You have now rolled out that change to a subset of users.

8. After the change is deployed to the canary environment, you can continue to roll it out to the rest of your users by merging the code with the master branch and pushing that to the Git server:
```
git checkout master
git merge canary
git push origin master
```
9. In approximately 1 minute, the master job in the sample-app folder kicks off.
10. Click the master link to show the stages of your pipeline, as well as pass/fail information and timing characteristics.
11. Poll the production URL to verify that the new version 2.0.0 has been rolled out and is serving requests from all users.
````
export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1;  done
````
You can look at the Jenkinsfile in the project to see the workflow.

### Deploying a development branch
Sometimes you need to work with nontrivial changes that can't be pushed directly to the canary environment. A development branch is a set of environments your developers use to test their code changes before submitting them for integration into the live site. These environments are a scaled-down version of your application, but are deployed using the same mechanisms as the live environment.

To create a development environment from a feature branch, you can push the branch to the Git server and let Jenkins deploy your environment. In a development scenario, you wouldn't use a public-facing load balancer. To help secure your application you can use kubectl proxy. The proxy authenticates itself with the Kubernetes API, and proxies requests from your local machine to the service in the cluster without exposing your service to the Internet.

1. Create another branch and push it to the Git server:
````
git checkout -b new-feature
git push origin new-feature
````
A new job is created and your development environment is in the process of being created. At the bottom of the console output of the job are instructions for accessing your environment.

2. Start the proxy in the background:
````
kubectl proxy &
````
3. Verify that your application is accessible by using localhost:
````
curl http://localhost:8001/api/v1/namespaces/new-feature/services/gceme-frontend:80/proxy/
````
4. You can now push code to this branch to update your development environment. When you are done, merge your branch back into canary to deploy that code to the canary environment.
````
git checkout canary
git merge new-feature
git push origin canary
````
5. When you are confident that your code won't cause problems in the production environment, merge from the canary branch to the master branch to kick off the deployment:
````
git checkout master
git merge canary
git push origin master
````
6. When you are done with the development branch, delete it from the server and delete the environment from your Kubernetes cluster:
````
git push origin :new-feature
kubectl delete ns new-feature
````
## Clean up
To avoid incurring charges to your Google Cloud account for the resources used in this tutorial, either delete the project that contains the resources, or keep the project and delete the individual resources.

### Deleting the project
The easiest way to eliminate billing is to delete the project that you created for the tutorial.

### To delete the project:

1. In the Cloud Console, go to the Manage resources page.
2. In the project list, select the project that you want to delete, and then click Delete.
3. In the dialog, type the project ID, and then click Shut down to delete the project.
