# Jenkins X Builders for Nuxeo Web UI CI/CD

A Jenkins X builder allows to build a Docker image that can be used as a container in a Jenkins X [Pod Template](https://jenkins-x.io/architecture/pod-templates/).

Such a pod template defines the Kubernetes pod used to run a CI/CD pipeline in the Jenkins X platform.

See the default [Jenkins X builders](https://github.com/jenkins-x/jenkins-x-builders).

## Image Builds

The pipeline defined by the [Jenkinsfile](Jenkinsfile) behaves differently depending if it runs from a pull request or on the `master` branch.

### Pull Request

The deployed versions are `$NEXT_VERSION-$BRANCH_NAME`.

For instance, to test the `builder-maven-nodejs-chrome` builder in your Jenkins X instance, you can use the following as a container image (PR-1):

```bash
<DOCKER_REGISTRY_IP>:<DOCKER_REGISTRY_PORT>/nuxeo/builder-maven-nodejs-chrome:0.0.2-PR-1
```

Replace the Docker registry IP and port according to the output of the following command:

```bash
kubectl get svc
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
...
jenkins-x-docker-registry       ClusterIP   10.63.253.90    <none>        5000/TCP    21d
```

### master

The deployed versions are `x.y.z` (depending on the Git tag) or `latest`.

## Manual Testing

In order to have [skaffold](https://github.com/GoogleContainerTools/skaffold) installed, the easiest way is to use a Jenkins X DevPod:

```bash
jx create devpod -l nodejs # Do not hesitate to use a different label
```

Then, from the devpod:

```bash
git checkout <BRANCH>
ORG=test VERSION=latest skaffold build -f skaffold.yaml
```

Finally, to test a Docker image:

```bash
docker pull <DOCKER_REGISTRY_IP>:<DOCKER_REGISTRY_PORT>/test/builder-maven-nodejs-chrome

docker run -it <DOCKER_REGISTRY_IP>:<DOCKER_REGISTRY_PORT>/test/builder-maven-nodejs-chrome:latest /bin/bash
```
