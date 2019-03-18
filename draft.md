# Draft

![Draft Logo](https://github.com/Azure/draft/raw/master/docs/img/draft-logo.png)



### Introduction 

I am exploring a series of open source tools to simplify the inner loop of the container native development workflow. This describes the period of time during which you are writing code, but have not yet pushed it to a version control system. These tools, [**Draft**](draft.md), [**Skaffold**](skaffold.md), and [**Tilt**](tilt.md) each take a different approach to the task at hand.  Each tool can be used to build an image of your project, push the image to a registry service of your choice, and deploy the image onto a Kubernetes cluster. Adopting these tools will free up your time and allow you to focus on writing code. You can learn more about the motivation behind this series in my [first post](intro.md). 



### Definition 

[Draft](https://github.com/Azure/draft) simplifies the process for developers to get started deploying their application to Kubernetes. It does so by creating boilerplate for a variety of programming languages and enabling a workflow to build, push, and deploy your application to a Kubernetes cluster. 



### Differentiators

Draft differentiates itself by means of its low barrier to entry. The majority of developers are looking to quickly test out how to use Kubernetes with their existing programming knowledge. They may not have Kubernetes expertise or hands on experience with `kubectl`. Using Draft, developers can deploy their application to a Kubernetes cluster with two commands: `draft create` and `draft up`. 

`draft create` is used to detect the language of the application you are developing and generate the artifacts needed to deploy the application to your cluster. This is done via **Draft packs**, which provide a Dockerfile and Helm Chart specific to your chosen language. There are examples for most popular programming languages. This saves you time that would otherwise be spent writing a Dockerfile or Kubernetes manifest from scratch. `draft up` reads your configuration and takes care of the build, push, and deployment steps for you. 

Unlike Skaffold and Tilt, Draft does not continuously watch for application changes. You will need to run `draft up` every time you would like the application to be updated. At this time Draft only supports a single Dockerfile and chart in the root directory, which can make it a challenge to simultaneously deploy multiple microservices using Draft. Draft relies on Helm, a package management tool for Kubernetes. Helm is a common tool used to deploy artifacts to a Kubernetes cluster. This can be beneficial if it happens to be the same tool used to deploy to your production cluster. It would be nice to have the flexibility of Skaffold's pluggable architecture, which provides you with the option to deploy with other tools. Unlike using `kubectl`, Helm has to be installed separately and in many cases requires the configuration of RBAC. 

It is worth noting that as of March 2019 the number of commits and stars on GitHub for Draft is about half that of Skaffold and that there are far fewer recent commits for Draft than there are for Skaffold. 


### Requirements

- [Docker For Desktop](https://www.docker.com/products/docker-desktop) or [Minikube](https://github.com/Azure/draft/blob/master/docs/install-minikube.md)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [Oracle Container Engine for Kubernetes (OKE)](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengoverview.htm)
- [Oracle Cloud Infrastructure Registry (OCIR)](https://docs.cloud.oracle.com/iaas/Content/Registry/Concepts/registryoverview.htm)
- [Helm](https://github.com/kubernetes/helm#install) 



### Installation 

More detailed installation information can be found on the [Draft GitHub page](https://github.com/Azure/draft/blob/master/docs/install-cloud.md). 

Download and install Draft: 

`brew tap azure/draft && brew install azure/draft/draft`

Verify your installation: 

`draft version`

To install default plugins and configure $DRAFT_HOME run `draft init`.

The installation will create a ` /.draft` folder in your home directory to store configuration information.  



### Local Configuration 

Clone the [Draft repository](https://github.com/Azure/draft) on GitHub and change to the `/examples/example-go` directory. In order to deploy the `main.go` application to a Kubernetes cluster, we will need a Dockerfile, Helm chart, and draft.toml. These artifacts can easily be created by running `draft create`. Draft detects the language of the application in the directory, in this case Go:  `--> Draft detected Go (100.000000%)` and creates the scaffolding accordingly:

```
$ ls 
Dockerfile	charts		draft.toml	glide.yaml	main.go
```

- The `Dockerfile` starts with a default Go image. It will install the dependencies in `requirements.txt` and copy the current directory into `/usr/src/app`. 
- The Helm chart includes the manifest file used to deploy your application to a Kubernetes cluster. The `/charts` and `Dockerfile` assets created by Draft default to a basic Go configuration. 
- The `draft.toml` file contains basic configuration details about the application: the name, repository, Kubernetes namespace, and whether to deploy the application automatically when local files change.

After exploring the artifacts created by Draft, you have to the option to configure your image registry. If you are using Minikube, run `eval $(minikube docker-env)` to allow Draft to build images directly using Minikube's Docker daemon which lets you skip having to set up a remote/external container registry. If you choose to use Docker for Desktop you will need to login to a Docker registry with `docker login`. If you do not configure an image registry, Draft will bypass this step: `WARNING: no registry has been set, therefore Draft will not push to a container registry.`

After configuring your registry, it is time to deploy the appliction to your local Kubernetes cluster. Draft will use your current Kubernetes context to determine the cluster on which your application will be deployed. To find your context run: `kubectl config current-context`. 

#### Deploy the Application 

Running the command `draft up` will take care of a number of actions at once. It will read the configuration details in `draft.toml`, build the image using Docker, push the image to your registry, and finally will `helm install` the chart referring to the image that was just built onto your Kubernetes cluster. 

```
$ draft up
Draft Up Started: 'example-go': 01D57NYWJ40ANYC6T2MGR6X8FR
example-go: Building Docker Image: SUCCESS ⚓  (189.0753s)
example-go: Pushing Docker Image: SUCCESS ⚓  (172.3998s)
example-go: Releasing Application: SUCCESS ⚓  (4.0052s)
```

#### Verify Your Deployment

Verify your application was properly deployed with `kubectl get pods`:

```
$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
example-go-example-go-7754dd6665-trl6x   1/1     Running   0          5s
```

#### Connect to the Application 

Connect to your application with `draft connect`:

```
$ draft connect
Connect to example-go:8080 on localhost:63118
```

```
$ curl localhost:63118
Hello World, I'm Golang!
```

#### Modify the Application 

If you modify and save the application, the next time you run `draft up`, Draft will determine there is already an existing Helm release and will perform a `helm upgrade` rather than another `helm install`. 

#### Remove the Application 

When you are finished testing the application, remove it from your Kubernetes cluster by running `draft delete`:

```
$ draft delete
app 'example-go' deleted
```



### Hosted Kubernetes and Docker Registry Configuration 

Draft can also be used with hosted Kubernetes solutions. My example will use [Oracle Container Engine for Kubernetes (OKE)](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengoverview.htm) as the Kubernetes cluster and [Oracle Cloud Infrastructure Registry (OCIR)](https://docs.cloud.oracle.com/iaas/Content/Registry/Concepts/registryoverview.htm) as the container image registry. Similar steps can be followed to configure Draft with other Kubernetes cluster and registry services. 

#### Registry Configuration 

To use a cloud registry service you will need to set the registry in Draft with the `draft config set registry` command. To set OCIR as your registry you will need to provide the server URL of the registry. Run: `draft config set registry <region code>.ocir.io/<tenancy name>/<repo name>/<image name>:<tag>` 

- `<region-code>` is one of `fra`, `iad`, `lhr`, or `phx`.
- `ocir.io` is the Oracle Cloud Infrastructure Registry name.
- `<tenancy-name>` is the name of the tenancy that owns the repository to which you want to push the image, for example `acme-dev`. Note that your user must have access to the tenancy.
- `<repo-name>`, if specified, is the name of a repository to which you want to push the image. For example, `project01`. Note that specifying a repository is optional. If you choose specify a repository name, the name of the image is used as the repository name in Oracle Cloud Infrastructure Registry.
- `<image-name>` is the name you want to give the image in Oracle Cloud Infrastructure Registry, for example, `helloworld`.
- `<tag>` is an image tag you want to give the image in Oracle Cloud Infrastructure Registry, for example, `latest`.

You will need to log into the registry with:

`docker login <region code>.ocir.io`

- Username: `<tenancy-name>/<oci-username>`
- Password: `<oci-auth-token>`

You may also be required to establish trust with the registry. For example, the OCIR registry will be set to private by default. If you would like to continue with a private repository, you will have to add an image pull secret which allows Kubernetes to authenticate with a container registry to pull a private image. For more information about using image secrets, refer to [this guide](https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-and-registry/index.html). A simpler option for testing purposes is to set the registry to be **public**. 

#### Kubernetes Cluster Configuration 

Because Draft uses your current Kubernetes context to determine the cluster on which your application will be deployed you will need remember to switch your context to OKE. Verify your context with: `kubectl config current-context`. 

As mentioned before, Draft deploys to Kubernetes by means of a Helm chart. If your cluster has RBAC enabled, you may need to create a new service account to grant Tiller, the in-cluster component of Helm, additional permissions to deploy to your cluster. For information about doing so, refer to the [Helm documentation](https://helm.sh/docs/using_helm/#role-based-access-control). 

Once the configuration is complete, run `draft up` to build your application using the Dockerfile, push the application to OCIR, and deploy the application to OKE. As you did before locally, you can verify the successful deployment of the application by running `kubectl get pods`. Connect to your application with `draft connect`. 



### References

[Draft User Docs](https://draft.sh/)

[Draft GitHub](https://github.com/Azure/draft) 





