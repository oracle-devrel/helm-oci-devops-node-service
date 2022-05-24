# Getting Started Helm-based Deployments with OCI DevOps

This is an example project using Node.js with the Express [getting started generator](https://expressjs.com/en/starter/generator.html). With the [OCI DevOps service](https://www.oracle.com/devops/devops-service/) and this project, you'll be able to build, test and deploy this application to Oracle Container Engine for Kubernetes (OKE) via Helm Charts.

In this example, you'll build a container image of this Node Express Getting Started App, and deploy your built container to the OCI Container Registry, then deploy the getting started app to Oracle Container Engine for Kubernetes (OKE) all using the OCI DevOps service!

Let's go!

## Download the repo

The first step to get started is to download the repository to your local workspace

```shell
git clone git@github.com:oracle-quickstart/oci-devops-node
cd oci-devops-node
```

## Install and run the Express example

Open a terminal and test out the simple Express example - a web app that returns a "Welcome to Express" page

1. Install Node 12 and NPM: https://docs.npmjs.com/downloading-and-installing-node-js-and-npm 
1. Build the app: `npm install`
1. Run tests: `npm test`
1. Run the app: `npm start`
1. Verify the app locally, open your browser to [http://localhost:3000/](http://localhost:3000/) or whatever port you set, if you've changed the local port

## Build a container image for the app

You can locally build a container image using docker (or your favorite container image builder), to verify that you can run the app within a container

```
docker build --pull --rm -t node-express-getting-started -f DOCKERFILE .
```

Verify that your image was built, with `docker images` 

Next run your local container and confirm you can access the app running in the container
```
docker run --rm -d -p 3000:3000 --name node-express-getting-started node-express-getting-started:latest
```

And open your browser to [http://localhost:3000/](http://localhost:3000/)

# Build and test the app in OCI DevOps

Now that you've seen you can locally build and test this app, let's build our CI/CD pipeline in OCI DevOps

## Create your Git repo

1. [Create a DevOps project](https://docs.oracle.com/en-us/iaas/Content/devops/using/devops_projects.htm), or use an existing project
1. Create a Code Repository in your DevOps project
1. Add the new Code Repository as a remote to your local git repo
```
git remote add devops ssh://devops.scmservice.us-ashburn-1.oci.oraclecloud.com/namespaces/MY-TENANCY/projects/MY-PROJECT/repositories/MY-REPO
```
1. View the Getting Started Guide to connect to your Code Repository via https or ssh

## Setup your Build Pipeline

Create a new Build Pipeline to build, test and deliver artifacts from a recent commit

## Managed Build stage

In your Build Pipeline, first add a Managed Build stage
1. The Build Spec File Path is the relative location in your repo of the build_spec.yaml . Leave the default, for this example
1. For the Primary Code Repository choose your Code Repository you created above
   - The Name of your Primary Code Repository is used in the build_spec.yaml. In this example, you will need to use the name `node_express` for the build_spec.yaml instructions to acess this source code
   - Select the `main` branch

## Create a Container Registry repository

Create a [Container Registry repository](https://docs.oracle.com/en-us/iaas/Content/Registry/Tasks/registrycreatingarepository.htm) for the `node-express-getting-started` container image built in the Managed Build stage. 
1. You can name the repo: `node-service`. So if you create the repository in the Ashburn region, the path is iad.ocir.io/TENANCY-NAMESPACE/node-express-getting-started
1. Set the repostiory access to public so that you can pull the container image without authorization, from OKE. Under "Actions", choose `Change to public`.

## Create a Container Registry Helm Repository

Create a [Container Registry repository](https://docs.oracle.com/en-us/iaas/Content/Registry/Tasks/registrycreatingarepository.htm) for the `helm-repo` container image built in the Managed Build stage. 
1. You can name the repo: `helm-repo`. So if you create the repository in the Ashburn region, the path is iad.ocir.io/TENANCY-NAMESPACE/node-service-helm-repo
1. Set the repostiory access to public so that you can pull the container image without authorization, from OKE. Under "Actions", choose `Change to public`.

## Create a DevOps Artifact for your container image repository

The version of the container image that will be delivered to the OCI repository is defined by a [parameter](https://docs.oracle.com/en-us/iaas/Content/devops/using/configuring_parameters.htm) in the Artifact URI that matches a Build Spec exported variable or Build Pipeline parameter name.

Create a DevOps Artifact to point to the Container Registry repository location you just created above. Enter the information for the Artifact location:
1. Name: node-express-getting-started container
1. Type: Container image repository
1. Path: `iad.ocir.io/TENANCY-NAMESPACE/node-service`
1. Replace parameters: Yes

Next, you'll set the container image tag to use the the Managed Build stage `exportedVariables:` name for the version of the container image to deliver in a run of a build pipeline. In the build_spec.yaml for this project, the variable name is: `BUILDRUN_HASH`
```
  exportedVariables:
    - BUILDRUN_HASH
```

Edit the DevOps Artifact path to add the tag value as a parameter name.
1. Path: `iad.ocir.io/TENANCY-NAMESPACE/node-service:${BUILDRUN_HASH}`

Now create DevOps Artifact for `helm-repo` too.
1. Name: `node-service-helm`
2. Type: `Helm Chart`
3. Helm Chart URL: `oci://iad.ocir.io/TENANCY-NAMESPACE/helm-repo/node-service`
4. Version: `0.1.0-${BUILDRUN_HASH}`

### Override values.yaml

Default `values.yaml` is found in `/helm` folder. But `values.yaml` can be overridden by creating Artifact in DevOps Project. 
1. Goto `Artifacts` in your DevOps Project.
2. Click on `Add Artifact` and enter Name as `values.yaml`, Type as `General artifact`, Artifact Source as `Inline`, value as content give below. 

```
replicaCount: 3

service:
  type: LoadBalancer
  port: 80

image:
  repository: iad.ocir.io/TENANCY-NAMESPACE/node-service
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: ${BUILDRUN_HASH}
```
3. Save the artifact.

**Note:**
* This values.yaml changes replicaCount to `3` from `1` and makes the service as `LoadBalancer` for public IP Address to get assigned. 
* Replace `TENANCY-NAMESPACE` with your valid tenancy name.

## Add a Deliver Artifacts stage

Let's add a **Deliver Artifacts** stage to your Build Pipeline to deliver the `node-service` container to an OCI repository. 

The Deliver Artifacts stage **maps** the ouput Artifacts from the Managed Build stage with the version to deliver to a DevOps Artifact resource, and then to the OCI repository.

Add a **Deliver Artifacts** stage to your Build Pipeline after the **Managed Build** stage. To configure this stage:
1. In your Deliver Artifacts stage, choose `Select Artifact` 
1. From the list of artifacts select the `node-express-getting-started container` artifact that you created above
1. In the next section, you'll assign the  container image outputArtifact from the `build_spec.yaml` to the DevOps project artifact. For the "Build config/result Artifact name" enter: `output01`

## Configure Build Parameters

`build_spec.yaml` takes care of running build and pushing helm charts to the OCIR repository. For publishing helm charts to OCIR, the credentials and OCIR path are sent as parameters. Under `Parameters` tab create below parameters with appropriate valid values.

| Parameter      | Description| Sample  |
| -----------    | ---------- | --------|
| USER_AUTH_TOKEN      | Auth token of the user who has access to OCIR. Refer [documentation](https://docs.oracle.com/en-us/iaas/Content/Registry/Tasks/registrygettingauthtoken.htm) to create token.   | |
| HELM_REPO_USER   | User name to publish helm package to OCIR        | <TENANCY_NAME>/<USER_NAME> |
| HELM_REPO_URL | OCIR helm repository URL | oci://iad.ocir.io/<TENANCY_NAME>/node-service-helm-repo |
| HELM_REPO | OCIR domain name | iad.ocir.io |

# Run your Build in OCI DevOps

## From your Build Pipeline, choose `Manual Run`

Use the Manual Run button to start a Build Run

Manual Run will use the latest commit to your Primary Code Repository, if you want to specify a specific commit, you can optionally make that choice for the Primary Code Repository in the dropdown and selection below the Parameters section.


## Connect your Code Repository to your Build Pipeline

To automatically start your Build Pipeline from a commit to your Code Repository, navigate to your Project and create a Trigger. 

A Trigger is the resource to 
filter the events from your Code Repository and on a matching event will start the run of a Build Pipeline.

## Push a commit to your DevOps Code Repository

Test out your Trigger by editing a file in this repo and pushing a change to your DevOps code repository.

# Connect your Build Pipeline with a Deployment Pipeline

For CI + CD: continous integration with a Build Pipeline and continuous deployment with a Deployment Pipeline, first create the Deployment Pipeline to deploy this example web application service to your OKE cluster. To review Deployment Pipelines, see the [example Reference Architecture](https://docs.oracle.com/en/solutions/build-pipeline-using-devops/index.html), and [docs](https://docs.oracle.com/en-us/iaas/Content/devops/using/deploy_oke.htm#deploy_to_oke). You'll need to [setup the policies](https://docs.oracle.com/en-us/iaas/Content/devops/using/devops_policy_examples.htm) to enable deployments, as well.

Because the K8s manifest doesn't change each build, we're just going to create a single version of the K8s manifest by hand (or via API/CLI), in the Artifact Registry.

## Create a DevOps Environment, Artifact Registry file, and DevOps Artifact

1. Create an [Enivornment](https://docs.oracle.com/en-us/iaas/Content/devops/using/create_oke_environment.htm) to point to your OKE cluster destination for this example. You will already need to have an OKE cluster created, or go through the [Reference Architecture automated setup](https://docs.oracle.com/en/solutions/build-pipeline-using-devops/index.html).

## Create your Deployment Pipeline

You've created the references to your OKE cluster and manifest to deploy, now create your [Deployment Pipeline](https://docs.oracle.com/en-us/iaas/Content/devops/using/deployment_pipelines.htm)
1. Create a new Deployment Pipeline
1. Add your first stage - choose the type to deploy helm to OKE: `Install helm chart to Kubernetes Cluster`
    1. Add stage name
    1. Choose the Environment that you created above
    1. For Select Artifact, select the Helm Chart DevOps artifact that points to `node-service-helm`
    1. For Select values artifacts, select values.yaml DevOps artifact that points to `values.yaml`
    1. Add

To run this pipeline on its own, you can add a parameter for `BUILDRUN_HASH` or, trigger it from the Build Pipeline which will forward the `build_spec.yaml` exported variables to the Deployment Pipeline.

## Add a Trigger Deployment stage to your Build Pipeline

Once you've created your Deployment Pipeline, you can add a **Trigger Deployment** stage as the last step of your Build Pipeline.

After the latest version of the container image is delivered to the Container Registry via the **Deliver Artifacts** stage, we can start a deployment to an OKE cluster

1. Add stage to your Build Pipeline
1. Choose a **Trigger Deployment** stage type
1. Choose `Select Deployment Pipeline` to choose the Deployment Pipeline that you created above.

From the Deployment Pipeline you selected, you can confirm the parameters of that pipeline in the Deployment Pipeline details.

# Make this your own

Fork this repo from Github and make changes if you want to play around with the sample app, the OCI DevOps build configuration, and the k8s manifest.