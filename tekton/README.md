# BlueCompute - Tekton Pipelines

## Set Up

1. Install [Tekton Pipelines](https://github.com/tektoncd/pipeline/blob/master/docs/install.md):
```
  # Create a new project called `tekton-pipelines`
  oc new-project tekton-pipelines

  # The `tekton-pipelines-controller` service account needs the `anyuid` security context constraint in order to run the webhook pod.  
  oc adm policy add-scc-to-user anyuid -z tekton-pipelines-controller

  # Install latest version of Tekton Pipelines
  oc apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.notags.yaml
```

2. To access the docker registry, create the required secret as follows.

```
# Create the below environment variables

export DOCKER_USERNAME='<DOCKER_USERNAME>'
export DOCKER_PASSWORD='<DOCKER_PASSWORD>'
export DOCKER_EMAIL='<DOCKER_EMAIL>'

# Create the docker secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=${DOCKER_USERNAME} \
  --docker-password=${DOCKER_PASSWORD} \
  --docker-email=${DOCKER_EMAIL} \
  -n ${NAMESPACE}
```

Before creating, replace the values `<DOCKER_USERNAME>`, `<DOCKER_PASSWORD>`, and `<DOCKER_EMAIL>` with your credentials.

**Note**: If your docker password contains special characters in it, please enclose the password in double quotes or place an escape character before each special character.

(Optional) Only if you have problems with the credentials you can recreate it, but you have to delete it first before recreating it. In order to do that, use the below command.

```
oc delete secret regcred -n $NAMESPACE
```

3. Create a `pipeline-account` service account, role and rolebinding:

```
# Create service account
oc apply -f pipeline-account.yaml
```

4. Create a `clusteradmin` rolebinding for the pipeline-account.

NOTE: This should not be done in production and the service account should only be set with permissions required to update deployment resource.

```
# Create clusteradmin rolebinding
oc apply -f clusteradmin-rolebinding.yaml
```

## Configuring Tekton Pipelines

### Inventory Microservice

1. Create the pipeline resources.

```
# Create git and docker pipeline resources
oc apply -f PipelineResources/bluecompute-inventory-pipeline-resources.yaml
```

Verify it by running the below command.

```
$ tkn res ls
NAME                     TYPE    DETAILS
git-source-inventory     git     url: https://github.com/ibm-cloud-architecture/refarch-cloudnative-micro-inventory
docker-image-inventory   image   url: index.docker.io/ibmcase/bluecompute-inventory
```

2. Create the Tasks.

```
# Create build task
oc apply -f Tasks/source-to-image-inventory.yaml

# Create deploy task
oc apply -f Tasks/deploy-using-kubectl-inventory.yaml
```

Verify it by running the below command.

```
tkn task ls
NAME                                AGE
bluecompute-inventory-build-task    4 days ago
bluecompute-inventory-deploy-task   3 days ago
```

3. Create the pipeline.

```
# Create the pipeline
oc apply -f Pipelines/build-and-deploy-pipeline-inventory.yaml
```

Verify it by running the below command.

```
tkn pipeline ls
NAME                                              AGE              LAST RUN                   STARTED          DURATION    STATUS
bluecompute-build-and-deploy-pipeline-inventory   4 days ago       bluecompute-inventory-pr   3 days ago       2 minutes   Succeeded
```

4. Create the pipeline run to trigger the pipeline.

```
# Create the pipeline run
oc apply -f PipelineRuns/bluecompute-inventory-pipeline-run.yaml
```

Verify it by running the below command.

```
tkn pipelinerun ls
NAME                       STARTED          DURATION    STATUS               
bluecompute-inventory-pr   3 days ago       2 minutes   Succeeded
```

If you want to view the logs, you can get them by using the below command.

```
tkn tr logs -f -a
```
