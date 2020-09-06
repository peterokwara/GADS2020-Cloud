## Task 1. Deploy GKE clusters

In this task, you use the GCP Console and Cloud Shell to deploy GKE clusters.

Use the GCP Console to deploy a GKE cluster

1. In the GCP Console, on the Navigation menu, click Kubernetes Engine > Clusters.

2. Click Create cluster to begin creating a GKE cluster.

3. Examine the console UI and the controls to change the cluster name, the cluster location, Kubernetes version.
   Clusters can be created across a region or in a single zone. A single zone is the default. When you deploy across a region the nodes are deployed to three separate zones and the total number of nodes deployed will be three times higher.

4. Name the cluster standard-cluster-1 and select zone us-central1-a. Leave all the values at their defaults and click Create.

```
gcloud beta container --project "qwiklabs-gcp-00-dd007816cb18" clusters create "standard-cluster-1" --zone "us-central1-a" --no-enable-basic-auth --cluster-version "1.15.12-gke.2" --machine-type "e2-medium" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "3" --enable-stackdriver-kubernetes --enable-ip-alias --network "projects/qwiklabs-gcp-00-dd007816cb18/global/networks/default" --subnetwork "projects/qwiklabs-gcp-00-dd007816cb18/regions/us-central1/subnetworks/default" --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0
```

The cluster begins provisioning. In a few minutes, provisioning is complete, and the Kubernetes Engine > Clusters page looks like the screenshot:

5. Click Check my progress to verify the objective.

6. You can scroll down the page to view more details.

7. Click the Storage and Nodes tabs under the cluster name (standard-cluster-1) at the top to view more of the cluster details.

## Task 2. Modify GKE clusters

It is easy to modify many of the parameters of existing clusters using either the GCP Console or Cloud Shell. In this task, you use the GCP Console to modify the size of GKE clusters.

1. In the GCP Console, click Edit at the top of the details page for standard-cluster-1.

2. Scroll down to the Node Pools section, click on default-pool then click Edit and change the number of nodes in the default pool from 3 to 4.

3. Scroll to the bottom and click Save.

4. Click the arrow to the left of Clusters to return to the GKE Clusters page.

When the operation completes, the Kubernetes Engine > Clusters page should show that standard-cluster-1 now has four nodes.

## Task 3. Deploy a sample workload

In this task, using the GCP console you will deploy a Pod running the nginx web server as a sample workload.

1. In the GCP Console, on the Navigation menu( Navigation menu), click Kubernetes Engine > Workloads.

2. Click Deploy to show the Create a deployment wizard

3. Click Continue to accept the default container image, nginx.latest, which deploys a Pod with a single container running the latest version of nginx.

4. Scroll to the bottom of the window and click the Deploy button leaving the Configuration details at the defaults.

5. When the deployment completes your screen will refresh to show the details of your new nginx deployment.

Click Check my progress to verify the objective.

## Task 4. View details about workloads in the GCP Console

In this task, you view details of your GKE workloads directly in the GCP Console.

1. In the GCP Console, on the Navigation menu (Navigation menu), click Kubernetes Engine > Workloads.

2. In the GCP Console, on the Kubernetes Engine > Workloads page, click nginx-1.

This displays the overview information for the workload showing details like resource utilization charts, links to logs, and details of the Pods associated with this workload.

3. In the GCP Console, click the Details tab for the nginx-1 workload. The Details tab shows more details about the workload including the Pod specification, number and status of Pod replicas and details about the horizontal Pod autoscaler.

4. Click the Revision History tab. This displays a list of the revisions that have been made to this workload.

5. Click the Events tab. This tab lists events associated with this workload.

6. And then the YAML tab. This tab provides the complete YAML file that defines this components and full configuration of this sample workload.

7. Still in the GCP Console's Details tab for the nginx-1 workload, click the Overview tab, scroll down to the Managed Pods section and click the name of one of the Pods to view the details page for that Pod.

8. The Pod Details page provides information on the Pod configuration and resource utilization and the node where the Pod is running.

9. In the Pod details page, you can click the Events and Logs tabs to view event details and links to container logs in Cloud Operations.

10. Click the YAML tab to view the detailed YAML file for the Pod configuration.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2020-09-06T17:39:31Z"
  generation: 2
  labels:
    app: nginx-1
  name: nginx-1
  namespace: default
  resourceVersion: "4962"
  selfLink: /apis/apps/v1/namespaces/default/deployments/nginx-1
  uid: d4e20015-b143-459d-b754-d5400f574a5a
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx-1
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-1
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always
        name: nginx-1
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2020-09-06T17:39:39Z"
    lastUpdateTime: "2020-09-06T17:39:39Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2020-09-06T17:39:31Z"
    lastUpdateTime: "2020-09-06T17:39:39Z"
    message: ReplicaSet "nginx-1-76949974bb" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 2
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1

```
