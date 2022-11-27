# OCP4 Tekton Easy Image Build Pipeline

OCP4 tekton based easy image build pipeline

This simple pipeline has steps to:

* Pull build artifacts from git repository. This includes the Dockerfile and any other code/config artifacts to be added to the image at build time.
* Build application image ( secure base image + build artifacts) using `Buildah` image. Store the built app image in pipeline image cache.
* Push app image to internal ocp registry on hub
* Push app image to hub quay repository

# Pre-reqs

* Hub Cluster installed with Openshift Pipelines Operator
* Developer Org created in Hub Quay and access to r/w robot credential
* Git Repo and SSH credential configured to read contents from code repo
* Able to log into prod hub cluster from cli (https://api.hub.prod.ocp.example.com:6443)

# Setup Steps

* Fork this project to your own repository
* Update `quay-secret.yaml` with quay org robot docker config credential.
* Update `gitlab-secret.yaml` with git ssh private key credential. Update `tekton.dev/git-0` annotation to correct git backend (e.g. gitlab.myorg.example.com or bitbucket or github url). This tells the pipeline which git credential to use during git clones.
* Update `image-tomcat-internal-registry-resource.yaml` url value to point to your OCP4 project.
* Update `image-tomcat-quay-resource.yaml` url to point to your quay org/repo.
* Update `namespace` values in `kustomization.yaml` under each overlay dir. E.g. `gitops/manifests/pipelines/overlays/*/kustomization.yaml`
```
# Log into prod hub cluster

oc login -u patrick https://api.hub.prod.ocp.example.com:6443

# Create a project that will home our image build pipeline
oc new-project <project name>

# Example
oc new-project pwr-demo
```

# Create Pipeline from Manifests

Create pipeline using oc and kustomize.

This example uses the cc1 prod hub cluster overlay (which has correct cc1 hub quay image urls):

```
oc apply -k gitops/manifests/pipelines/overlays/sample-hub
```

Sample output of created pipeline objects:

```
serviceaccount/pipeline configured
secret/quay-secret created
secret/ssh-key created
persistentvolumeclaim/pipeline-task-cache-pvc created
pipelineresource.tekton.dev/dockerfiles created
pipelineresource.tekton.dev/image-tomcat-cache created
pipelineresource.tekton.dev/image-tomcat-internal-registry created
pipelineresource.tekton.dev/image-tomcat-quay created
pipeline.tekton.dev/easy-image-build-pipeline created
task.tekton.dev/build created
task.tekton.dev/clear-buildah-repo created
task.tekton.dev/quay-push created
```

# Create a PipelineRun (pipeline instance)

Create an instance of the pipeline with given input parameters.

Here we are binding our pipeline resource inputs to the genericized pipeline.

```
oc create -f gitops/manifests/pipelines/overlays/sample-hub/pipelinerun.yaml
```

Sample output

```
pipelinerun.tekton.dev/easy-image-build-pr-cq67z created
```

Check status

```
oc get pr
NAME                        SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
easy-image-build-pr-cq67z   Unknown     Running   23s
```

Can also view pipeline status in web console ui

https://console-openshift-console.apps.hub.prod.ocp.example.com/

# Build Code Artifact (war) from Src and Build Application Image with s2i

The following example uses a Dockerfile that copies java source code in place and calls the tomcat base image s2i tooling to pull dependencies and build the war file.

See `app/docker/Dockerfile-person-src` for more details.

Create the pipeline:

```
oc apply -k gitops/manifests/pipelines/overlays/person-hub
```

Start the pipeline:

```
oc create -f gitops/manifests/pipelines/overlays/person-hub/pipelinerun.yaml
```

# TODO

* Update example to pull code artifact(s) from nexus or other artifact repo.

# Appendex

Sample docker config.json for prod hubs

```
{
  "auths": {
    "hub-registry-quay-quay-enterprise.apps.hub.prod.ocp.example.com": {
      "auth": "FIXME-BASE-64-1",
      "email": ""
    },
    "hub-registry-quay-quay-enterprise.apps.hub.prod.ocp.example.com": {
      "auth": "FIXME-BASE-64-2",
      "email": ""
    }
  }
}
```

Where FIXME-BASE-64-* should be replaced with value respectively:

```
echo -n "<quay robot user>:<quay robot password>" | base64 -w0
```