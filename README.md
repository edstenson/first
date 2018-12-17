# Deeptest Private Cloud Deployment

There are two parts to deploying Deeptest inside a cloud environment:

1. Setting up the cluster.
2. Deploying Deeptest to the cluster.

The exact instructions for the first part will depend on the cloud infrastructure you are using (Google, Azure, AWS or other).
The second part will be the same irrespective of the cloud infrastructure you are using.

## Set up the cluster

### Google Cloud

Set your project name, cluster name, and image repository name. For example:
```bash
export GCLOUD_PROJECT="deeptest-demo-project"
export CLUSTER_NAME="deeptest-demo"
export REPO_NAME="eu.gcr.io/${GCLOUD_PROJECT}"
```

Create GKE cluster.
```bash
gcloud container --project "${GCLOUD_PROJECT}" clusters create "${CLUSTER_NAME}" --machine-type "n1-standard-4"
```

Enable authentication to access your cluster using **kubectl**.
```bash
gcloud container --project "${GCLOUD_PROJECT}" clusters get-credentials "${CLUSTER_NAME}"
```

Enable authentication to push images to your container repository.
```bash
gcloud auth configure-docker
```

### Microsoft Azure

Set your resource group, cluster name, and image repository name. For example:
```bash
export RESOURCE_GROUP="deeptest-group"
export CLUSTER_NAME="deeptest-demo"
export REPO_NAME="deeptest-repo"
```

Create a service principal. This is required to allow the application to download the containers from the private repo.
NOTE: You will need the id for the service principal from the output later in the process.
```bash
az ad sp create-for-rbac --skip-assignment
```

Create a resource group to deploy the cluster into.
```bash
az group create --name ${RESOURCE_GROUP} --location uksouth
```

Create a private container repository for the images.
```bash
az acr create --resource-group ${RESOURCE_GROUP} --name ${REPO_NAME} --sku Basic
```

Login (locally) to the container repository.
```bash
az acr login --name ${REPO_NAME}
```

Get the name of the repository (to change the tags).
```bash
az acr list --resource-group ${RESOURCE_GROUP} --query "[].{acrLoginServer:loginServer}" --output table
```

Create the cluster.
```bash
az aks create \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER_NAME} \
    --node-count 10 \
    --service-principal 2be2fb0b-cdf4-4d5a-9d4a-3f671d2b33a6 \
    --client-secret 519a1242-9bfd-462b-8dd7-6b1410db9554 \
    --generate-ssh-keys
```

Allow the cluster to access repository to get the images.
```bash
az role assignment create --assignee 2be2fb0b-cdf4-4d5a-9d4a-3f671d2b33a6 --role Reader
```

### AWS

Follow the AWS documentation to create a cluster:

https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html

This should include the following:

* Amazon EKS Prerequisites
* Create Your Amazon EKS Cluster
* Configure kubectl for Amazon EKS
* Launch and Configure Amazon EKS Worker Nodes

You will also need to create a default **Storage Class** for your cluster, or the Persistent Volume Claims will fail:

https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html


Set up your account ID and image repository name. For example:
```bash
export ACCOUNT_ID="xxxxxxxxxxx"
export REPO_NAME="${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com"
```

Create an **ECR** repository for each image type:
```bash
for image in front-end application-server deeptest-worker keymaker; do aws ecr create-repository --repository-name ${image}; done
```

Set up local authenticaton to **ECR**:

```bash
$(aws ecr get-login --no-include-email --region us-east-1)
```

### IBM Cloud

Create a Kubernetes cluster using the wizard on the IBM Cloud platform, once running follow the instructions for operating
with the cluster on your local CLI:

```bash
curl -sL https://ibm.biz/idt-installer | bash
ibmcloud login -a https://api.eu-gb.bluemix.net
ibmcloud cs region-set uk-south
ibmcloud cs cluster-config diffblue
export KUBECONFIG=~/.bluemix/plugins/container-service/clusters/diffblue/kube-config-lon02-diffblue.yml
```

Then create a registry to push the Deeptest images to:

```bash
ibmcloud plugin install container-registry -r Bluemix
ibmcloud login -a https://api.eu-gb.bluemix.net
ibmcloud cr namespace-add <my_namespace>
ibmcloud cr login
```

Finally set the `repoName` environment variable to point to your registry and namespace:

```bash
export REPO_NAME=registry.eu-gb.bluemix.net/diffblue-cr
```

## Deploying Deeptest

Extract the files from the distribution package:

```bash
tar -xvf diffblue.tar.gz
```

Copy the deeptest images locally:

``` bash
docker load -i docker.tar.gz
```

Tag images:

```bash
for image in front-end application-server deeptest-worker keymaker; do docker tag "eu.gcr.io/diffblue-cr/${image}:latest" "${REPO_NAME}/${image}:latest"; done
```

Push images to your repository:

```bash
for image in front-end application-server deeptest-worker keymaker; do docker push "${REPO_NAME}/${image}:latest"; done
```

Modify the Kubernetes manifests with details of your repository:

```bash
sed -i "s;type: NodePort;type: LoadBalancer;g" kubernetes/manifests/20-service-front-end.yaml
for manifest in kubernetes/manifests/*; do sed -i "s;image: eu.gcr.io/diffblue-cr/;image: ${REPO_NAME}/;g" $manifest; done
```

Apply the Kubernetes manifests to your cluster:

```bash
for manifest in kubernetes/manifests/*; do kubectl apply -f $manifest; done
```

See all pods and nodes:

```bash
kubectl get pods -o wide -n diffblue
```

Get `front-end` external/public IP address:

```bash
kubectl get services front-end -n diffblue
```

## Troubleshooting

### IBM Cloud

#### ImagePullError

If you see errors pulling images from the repository this tends to mean the registry is in a different zone.
By default registries are open to nodes inside the same zone, but if they are different you can add a token can be added
to the deployment:

```bash
ibmcloud cr token-add --description "Registry token" --non-expiring --readwrite
kubectl --namespace diffblue create secret docker-registry registry-token --docker-server=registry.eu-de.bluemix.net --docker-username=token --docker-password=<token from first step> --docker-email=a@b.com
```

Ensuring the token is replaced as well as the correct registry server. Once this is done the following needs
to be added to the `spec` sections of the `deeptest-worker`, `keymaker`, and `front-end` deployment yamls and
the `application-server` stateful set yaml:

```yaml
    spec:
      imagePullSecrets:
      - name: registry-token
```

These can then be re-applied.

#### Mongo CrashLoopBackoff

In some cases Mongo can fail to start in the initial interval timeframe set in the manifest yaml files. This
seems to be specific to IBM Cloud, and how volumes take some time to start. This can be rectified by increasing
the `initialDelaySeconds: 30` value to around 100 seconds in both the liveness and readiness checks.
