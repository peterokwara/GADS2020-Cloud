# Activate Google Cloud Shell

Google Cloud Shell is a virtual machine that is loaded with development tools. It offers a persistent 5GB home directory and runs on the Google Cloud. Google Cloud Shell provides command-line access to your GCP resources.

1. In GCP console, on the top right toolbar, click the Open Cloud Shell button.

Cloud Shell icon

2. Click Continue. cloudshell_continue.png

It takes a few moments to provision and connect to the environment. When you are connected, you are already authenticated, and the project is set to your PROJECT_ID. For example:

Cloud Shell Terminal

gcloud is the command-line tool for Google Cloud Platform. It comes pre-installed on Cloud Shell and supports tab-completion.

You can list the active account name with this command:

```
gcloud auth list
```

Output:

Credentialed accounts:

```
- <myaccount>@<mydomain>.com (active)
  Example output:
```

Credentialed accounts:

```
- google1623327_student@qwiklabs.net
  You can list the project ID with this command:
```

```
gcloud config list project
```

Output:

```
[core]
project = <project_ID>
```

Example output:

```
[core]
project = qwiklabs-gcp-44776a13dea667a6
```

Full documentation of gcloud is available on Google Cloud gcloud Overview .

# Clone Repository

Let's get set up. You'll first set your zone and then clone the lab's sample code into your Cloud Shell:

```
gcloud config set compute/zone us-east1-d

git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git

cd continuous-deployment-on-kubernetes
```

The Git repository contains Kubernetes manifests that you'll use to deploy an application. The manifests and their settings are described in Configuring Jenkins for Kubernetes Engine.

# Create a Service Account with permissions

Using a service account is the recommended way to avoid granting extra permissions in Jenkins and the cluster.

1. Create a Google Cloud Platform (GCP) service account.

```
gcloud iam service-accounts create jenkins-sa \
    --display-name "jenkins-sa"

```

Output (do not copy):

```
Created service account [jenkins-sa].
```

2. Add required permissions, to the service account, using predefined roles.

Most of these permissions are related to Jenkins use of Cloud Build, and storing/retrieving build artifacts in Cloud Storage. Also, the service account needs to enable the Jenkins agent to read from a repo you will create in Cloud Source Repositories (CSR).

```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member "serviceAccount:jenkins-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
    --role "roles/viewer"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member "serviceAccount:jenkins-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
    --role "roles/source.reader"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member "serviceAccount:jenkins-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
    --role "roles/storage.admin"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member "serviceAccount:jenkins-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
    --role "roles/storage.objectAdmin"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member "serviceAccount:jenkins-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
    --role "roles/cloudbuild.builds.editor"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member "serviceAccount:jenkins-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
    --role "roles/container.developer"
```

You can check the permissions added using IAM & admin in Cloud Console.

jenkins_sa_iam

3. Export the service account credentials to a JSON key file in Cloud Shell:

```
gcloud iam service-accounts keys create ~/jenkins-sa-key.json \
    --iam-account "jenkins-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com"
```

Output (do not copy):

```
created key [...] of type [json] as [/home/.../jenkins-sa-key.json] for [jenkins-sa@myproject.aiam.gserviceaccount.com]
```

4. Download the JSON key file to your local machine.

Click Download File from More on the Cloud Shell toolbar:

download_file

5. Enter the File path as jenkins-sa-key.json and click Download.

The file will be downloaded to your local machine, for use later.

# Create a Kubernetes cluster for Jenkins and your application

1. Provision a Kubernetes cluster, on Google Kubernetes Engine (GKE):

```
gcloud container clusters create jenkins-cd \
 --num-nodes 2 \
 --machine-type n1-standard-2 \
 --cluster-version latest \
 --service-account "jenkins-sa@\$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com"
```

This step can take up to several minutes to complete.

Click Check my progress to verify the objective.
Create a Kubernetes cluster (zone: us-east1-d)

2. Confirm that your cluster is running by running the following command:

```
gcloud container clusters list
```

3. Now, get the credentials for your cluster:

```
gcloud container clusters get-credentials jenkins-cd
```

Kubernetes Engine uses these credentials to access your newly provisioned cluster.

4. Confirm that you can connect to the cluster:

```
kubectl cluster-info
```

Add yourself as a cluster administrator, in the cluster's RBAC:

This is needed so that you can give Jenkins permissions in the cluster.

```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=\$(gcloud config get-value account)
```

Output (do not copy):

```
Your active configuration is: [cloudshell-...]
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-binding created
```

# Install Helm

In this lab, you will use Helm to install Jenkins from its Charts repository. Helm is a package manager that makes it easy to configure and deploy Kubernetes applications. Once you have Jenkins installed, you'll be able to set up your CI/CD pipeline.

1. Download and install the helm binary:

```
wget https://get.helm.sh/helm-v3.2.1-linux-amd64.tar.gz
```

2. Unzip the file in Cloud Shell:

```
tar zxfv helm-v3.2.1-linux-amd64.tar.gz
cp linux-amd64/helm .
```

3. Add the official stable repository:

```
./helm repo add stable https://kubernetes-charts.storage.googleapis.com
```

4. Ensure Helm is properly installed:

```
./helm version
```

You should see versions appear for the client of v3.2.1.

# Configure and Install Jenkins

You will use a custom values file to add the GCP specific plugins necessary to use service account credentials to reach your Cloud Source Repository.

1. Use the Helm CLI to deploy the chart with your configuration settings:

```
./helm install cd-jenkins -f jenkins/values.yaml stable/jenkins --version 1.7.3 --wait
```

This may take a minute.

2. The Jenkins pod STATUS should change to Running when it's ready:

```
kubectl get pods
```

Output (do not copy):

```
NAME READY STATUS RESTARTS AGE
cd-jenkins-7c786475dd-vbhg4 1/1 Running 0 1m
```

Click Check my progress to verify the objective.
Configure and Install Jenkins

3. Apply the cluster-admin role to the Jenkins service account:

```
kubectl create clusterrolebinding jenkins-deploy \
 --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins
```

In this tutorial, the Jenkins service account needs cluster-admin permissions so that it can create Kubernetes namespaces and any other resources that the app requires. For production use, you should catalog the individual permissions necessary and apply them to the service account individually.

4. Set up port forwarding to the Jenkins UI, from Cloud Shell:

```
export JENKINS_POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd-jenkins" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $JENKINS_POD_NAME 8080:8080 >> /dev/null &
```

5. Now, check that the Jenkins Service was created properly:

```
kubectl get svc
```

Output (do not copy):

```
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
cd-jenkins 10.35.249.67 <none> 8080/TCP 3h
cd-jenkins-agent 10.35.248.1 <none> 50000/TCP 3h
kubernetes 10.35.240.1 <none> 443/TCP 9h
```

This Jenkins configuration is using the Kubernetes Plugin, so that builder nodes will be automatically launched as necessary when the Jenkins master requests them. Upon completion of the work, the builder nodes will be automatically be turned down, and their resources added back to the clusters resource pool.

Notice that this service exposes ports 8080 and 50000 for any pods that match the selector. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster. Additionally, the jenkins-ui services are exposed using a ClusterIP so that it is not accessible from outside the cluster.

# Connect to Jenkins

1. The Jenkins chart will automatically create an admin password for you. To retrieve it, run:

```
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

2. To get to the Jenkins user interface, click on the Web Preview button in cloud shell, then click Preview on port 8080.

web_prev.png

3. You should now be able to log in with username admin and your auto-generated password.

jenkins_login.png

You now have Jenkins set up in your Kubernetes cluster! Jenkins will drive your automated CI/CD pipelines in the next sections.

# Understand the Application

You'll deploy the sample application, gceme, in your continuous deployment pipeline. The application is written in the Go language and is located in the repo's sample-app directory. When you run the gceme binary on GKE, the app displays the instance's metadata in an info card.

90fb779a65809b9e.png

The application mimics a microservice by supporting two operating modes.

- In backend mode: gceme listens on port 8080 and returns host node instance metadata in JSON format.
- In frontend mode: gceme queries the backend gceme service and renders the resulting JSON in the user interface.
  fc22b68ab20dfe0e.png

Both the frontend and backend modes of the application support two additional URLs:

- /version prints the version of the binary (declared as a const in main.go)
- /healthz reports the health of the application. In frontend mode, health will be OK if the backend is reachable.

# Deploy the Application to Kubernetes

You will deploy the application into two different environments:

- Production: The live site that your users access.

- Canary: A smaller-capacity site that receives only a percentage of your user traffic. Use this environment to validate your software with live traffic before it's released to all of your users.

1. In Google Cloud Shell, navigate to the sample application directory:

```
cd sample-app
```

2. Create the Kubernetes namespace to logically isolate the deployment:

```
kubectl create ns production
```

Output (do not copy):

```
namespace/production created
```

3. Create the production Deployments for frontend and backend:

```
kubectl --namespace=production apply -f k8s/production
```

Output (do not copy):

```
deployment.extensions/gceme-backend-production created
deployment.extensions/gceme-frontend-production created
```

4. Create the canary Deployments for frontend and backend:

```
kubectl --namespace=production apply -f k8s/canary
```

Output (do not copy):

```
deployment.extensions/gceme-backend-canary created
deployment.extensions/gceme-frontend-canary created
```

5. Create the Services for frontend and backend:

```
kubectl --namespace=production apply -f k8s/services
```

Output (do not copy):

```
service/gceme-backend created
service/gceme-frontend created
```

Click Check my progress to verify the objective.
Create the production and canary deployments in production namespace

6. Scale the production, frontend service:

By default, only one replica of the frontend is deployed. Use the kubectl scale command to ensure that there are at least 4 replicas running at all times. This will be used to set the ratio of production pods to canary pods.

```
kubectl --namespace=production scale deployment gceme-frontend-production --replicas=4
```

Output (do not copy):

```
deployment.extensions/gceme-frontend-production scaled
```

This may take upto a minute for all the pods to be running.

7. Confirm that you have 5 pods running for the frontend:

4 are for production traffic and 1 is for canary releases. Changes to the canary release will only affect 1 out of 5 (20%) users.

```
kubectl get pods -n production -l app=gceme -l role=frontend
```

8. Also confirm that you have 2 pods for the backend:

1 is for production and 1 is for canary.

```
kubectl get pods -n production -l app=gceme -l role=backend
```

9. Retrieve the External IP for the production services:

```
kubectl --namespace=production get service gceme-frontend
```

It can take several minutes before you see the load balancer external IP address.

Output (do not copy):

```
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
gceme-frontend LoadBalancer 10.79.241.131 104.196.110.46 80/TCP 5h
```

10. Confirm that both services are working by opening the frontend EXTERNAL-IP in your browser

backend_card.png

11. Check the version output of the service:

```
export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
curl http://$FRONTEND_SERVICE_IP/version
```

Output (do not copy):

```
1.0.0
```

You have successfully deployed the sample application!

# Create a repository to host the sample app source code

1. Create your own copy of the gceme sample app in Cloud Source Repository.

```
gcloud source repos create gceme
```

You can ignore the warning, you will not be billed for this repository.

Click Check my progress to verify the objective.
Create a repository

2. Initialize the sample-app directory as its own local Git repository:

```
git init

git config credential.helper gcloud.sh
```

3. Add a git remote for the new repo in CSR.

```
git remote add origin https://source.developers.google.com/p/$GOOGLE_CLOUD_PROJECT/r/gceme
```

4. Review the remotes created:

```
git remote -v
```

```
origin https://source.developers.google.com/p/qwiklabs-gcp-.../r/gceme (fetch)
origin https://source.developers.google.com/p/qwiklabs-gcp-.../r/gceme (push)
```

5. Set the username and email address for your Git commits.

```
git config --global user.email "\$USER@qwiklabs.net"

git config --global user.name "\$USER"
```

6. Review the username and email address.

```
git config --global -l
```

7. Add, commit, and push the files to CSR:

```
git add .

git commit -m "Initial commit"

git push origin master
```

You have successfully pushed the sample application source to a new repo. Next, you will set up a pipeline for deploying your changes continuously and reliably.

# Create the Jenkins job

Navigate to your Jenkins user interface and follow these steps to configure a Pipeline job.

1. Click Jenkins > New Item in the left navigation:

86e92964a4599231.png

2. Use sample-app as the item name, then choose the Multibranch Pipeline option and click OK.

sample-app

3. On the next page, in the Branch Sources section, click Add Source and select git.

4. Paste the HTTPS clone URL of your gceme repo in Cloud Source Repositories into the Project Repository field.

Use the output of this command:

```
echo https://source.developers.google.com/p/$GOOGLE_CLOUD_PROJECT/r/gceme
```

5. From the Credentials drop-down, select the name of the credentials you created when adding your service account in the previous steps. It should have the format PROJECT_ID service account.

If you did not paste your project repository correctly, you will only see 1 choice: - none -. Try fixing your repo name until you see another credential name.

6. Under Scan Multibranch Pipeline Triggers section, check the Periodically if not otherwise run box and set the Interval value to 1 minute.

git-credentials

7. Click Save leaving all other options with their defaults.

After you complete these steps, a job named Branch indexing runs. This meta-job identifies the branches in your repository and ensures changes haven't occurred in existing branches. If you click sample-app in the top left, the master job should be seen.

The first run of the master job will fail until you make a few code changes in the next step.

You have successfully created a Jenkins pipeline! Next, you'll create the development environment for continuous integration.

# Creating the Development Environment

Development branches are a set of environments your developers use to test their code changes before submitting them for integration into the live site. These environments are scaled-down versions of your application, but need to be deployed using the same mechanisms as the live environment.

Creating a development branch
To create a development environment from a feature branch, you can push the branch to the Git server and let Jenkins deploy your environment.

Create a development branch and push it to the Git server:

```
git checkout -b new-feature
```

# Modifying the pipeline definition

The Jenkinsfile that defines that pipeline is written using the Jenkins Pipeline Groovy syntax. Using a Jenkinsfile allows an entire build pipeline to be expressed in a single file that lives alongside your source code. Pipelines support powerful features like parallelization and require manual user approval.

In order for the pipeline to work as expected, you need to modify the Jenkinsfile to set your project ID.

1. Open the Jenkinsfile in your terminal editor, for example vi:

```
vi Jenkinsfile
```

2. Start insert-mode in vi:

```
i
```

3. Update the correct PROJECT environment variable.

Look for the line that begins with PROJECT =. Replace the string with your PROJECT_ID, found in the CONNECTION DETAILS section of the lab. You can also use the results of gcloud config get-value project.

Be sure to replace REPLACE_WITH_YOUR_PROJECT_ID with your project name.

4. Save the Jenkinsfile file:

For vi, hit Esc then, :wq, for write-quit.

```
:wq
```

# Modify the application in your new-feature branch

To demonstrate changing the application, change the gceme cards from blue to orange.

1. Open html.go:

```
vi html.go
```

2. Start insert-mode in vi:

```
i
```

3. Change the two instances of <div class="card blue"> to the following:

```
<div class="card orange">
```

4. Save the html.go file:

For vi, hit Esc then, :wq, for write-quit.

```
:wq
```

5. Open main.go:

```
vi main.go
```

6. Start insert-mode in vi:

```
i
```

7. Change the version string:

Update this:

```
const version string = "1.0.0"
```

To this:

```
const version string = "2.0.0"
```

8. Save the main.go:

For vi, hit Esc then, :wq, for write-quit.

```
:wq
```

# Build and deploy the changes

1. Commit and push your changes:

```
git add -A

git commit -m "Version 2.0.0"

git push origin new-feature
```

This will kick off a build of your development environment.

2. Navigate back to the Jenkins UI and review the new-feature job:

After the change is pushed to the Git repository, you can see that your build started automatically for the new-feature branch. It can take up to a minute for the changes to be picked up by Jenkins.

757c7d394c91b309.png

3. After the build is running, follow the build output:

Click the down arrow next to the build# in the Build Executor Status area. Select Console output.

6ea3b2ed776e3b29.png

Track the output of the build for a few minutes. When you see Finished: SUCCESS, your new-feature branch has been deployed to your cluster.

# Verify the changes

In a development scenario, you wouldn't use a public-facing load balancer to service your application. To help secure your application, you can use port-forwarding check your deployed changes, without exposing your service to the public internet.

If you missed the build in Build Executor, not to worry. Just go to the Jenkins UI > sample-app. Verify that the new-feature pipeline has been created.

1. Once the build is complete, start port-forwarding on 8001 in the background:

```
export DEV_POD_NAME=$(kubectl get pods -n new-feature -l "app=gceme,env=dev,role=frontend" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward -n new-feature $DEV_POD_NAME 8001:80 >> /dev/null &
```

2. Verify that your application is accessible:

Send a request to localhost; kubectl will forward it to your service.

```
curl http://localhost:8001/api/v1/namespaces/new-feature/services/gceme-frontend:80/
```

You should see it respond with 2.0.0, which is the version that is now running.

You have set up the development environment! Next, you will build on what you learned in the previous module by deploying a canary release to test out a new feature.

# Deploying a Canary Release

You've verified that your app is running the latest code in the development environment, so let's deploy that code to the canary environment.

1. Create a canary branch and push it to the Git server:

```
git checkout -b canary

git push origin canary
```

In the Jenkins UI, you should notice the canary pipeline has kicked off.

2. Once complete, check the service URL for the new version:

Ensure that some of the traffic is being served by your new version. You should see about 1 in 5 requests (in no particular order) returning version 2.0.0.

```
export FRONTEND_SERVICE_IP=\$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
```

```
while true; do curl http://\$FRONTEND_SERVICE_IP/version; sleep 1; done
```

If you keep seeing 1.0.0, try checking and re-running the above commands again. Eventually, you should verify 2.0.0 is returned at times.

3. End the while loop, with Ctrl+C.

That's it! You have deployed a canary release. Next, you will deploy the new version to production.

# Deploying to production

Now that the canary release was successful, and you haven't heard any customer complaints, deploy to the rest of your production fleet.

1. Change to the master branch, merge canary, and push to Git:

```
git checkout master

git merge canary

git push origin master
```

2. Return to the Jenkins UI, and look for the master pipeline to kick off.

3. Once complete, check the external service URL:

4. Ensure that all of the traffic is being served by your new version, 2.0.0.

```
while true; do curl http://\$FRONTEND_SERVICE_IP/version; sleep 1; done
```

Once again, if you see responses of 1.0.0, try running the above commands again.

5. Stop the command by pressing Ctrl+C.

Output (do not copy):

```
gcpstaging9854_student@qwiklabs-gcp-df93aba9e6ea114a:~/continuous-deployment-on-kubernetes/sample-app$ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
2.0.0
2.0.0
2.0.0
2.0.0
2.0.0
2.0.0
^C
```

5. Navigate to the External-IP, where gceme displays the info cards.

The card color changed from blue to orange. Here's the command again to get the External-IP address so you can check it out:

```
kubectl get service gceme-frontend -n production
```

Output:

orange_card.png

You're done!

Awesome job, you have successfully deployed your application to production!
