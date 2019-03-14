# Tilt

![img](https://github.com/windmilleng/tilt/raw/master/assets/logo-wordmark.png)



### Introduction 

I am exploring a series of open source tools to simplify the inner loop of the container native development workflow. This describes the period of time during which you are writing code, but have not yet pushed it to a version control system. These tools, [**Draft**](draft.md), [**Skaffold**](skaffold.md), and [**Tilt**](tilt.md) each take a different approach to the task at hand.  Each tool can be used to build an image of your project, push the image to a registry service of your choice, and deploy the image onto a Kubernetes cluster. Adopting these tools will free up your time and allow you to focus on writing code. You can learn more about the motivation behind this series in my [first post](intro.md). 



### Definition 

[Tilt](https://github.com/windmilleng/tilt) is a command line tool used for local continuous development of microservice applications. Tilt watches your files for edits with `tilt up`, and then automatically builds, pushes, and deploys any changes to bring your environment up-to-date in real-time. Tilt provides visibility into your microservices with a command line UI. In addition to monitoring deployment success, the UI also shows logs and other helpful information about your deployments. 



### Differentiators

Tilt makes it easy to deploy multiple microservices simultaneously. You can reference more than one `Dockerfile` and more than one Kubernetes manifest in your `Tiltfile`. While Tilt and Skaffold are both useful for the development of multiple interconnected microservices, Tilt differentiates itself with two key features: **heads up display** and [**fast build**](https://docs.tilt.dev/fast_build.html).

The heads up display indicates whether or not a pod has been successfully deployed to your cluster with a simple green (successful) and red (unsuccessful) indicator next to the pod name. It also includes information about the name of each pod deployed in the cluster using Tilt, the deployment status, deployment history, and pod logs: both build logs and running container logs. This is especially helpful when you find yourself developing a growing number of interrelated microservices. Some might push back on the UI as superfluous - of course all of this information is available using basic `kubectl` commands - but it certainly makes it harder to miss pod issues when they are as clear as a green dot flipping to red. The UI is justified because it helps you focus on what matters. 

Fast build addresses one of the major bottlenecks of local development: the build and update process. Tilt created fast build to perform incremental image builds. Rather than re-download all of the dependencies and recompile from scratch each time code changes, fast build uses a build cache to skip steps done previously and inject the build cache from the previous run. Tilt also speeds up the deployment process by means of a sidecar Synclet running on the same node as your existing pod. The Synclet adds updated files to an existing pod and restarts the container instead of deploying a new pod from scratch when your code changes. The Synclet further speeds up the build and deploy process by allowing you to bypass the registry to send code updates directly to the Synclet and also by allowing you to run build commands directly in your cluster. Fast build take over much of the build responsibility from your `Dockerfile`. As a result, your `Dockerfile` cannot contain any ADD or COPY lines, so you may need to make small changes. 

Tilt emphasizes deployment to local Kubernetes clusters rather than managed ones. For instance, currently there is no command to pass in a registry to your configuration. It not hard to work around this, but it does require you to manually replace the image repositories in the Tiltfile and manifest file, which is more cumbersome than a single command that updates a config file. 


### Requirements

- [Docker For Desktop](https://www.docker.com/products/docker-desktop)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [Oracle Container Engine for Kubernetes (OKE)](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengoverview.htm)
- [Oracle Cloud Infrastructure Registry (OCIR)](https://docs.cloud.oracle.com/iaas/Content/Registry/Concepts/registryoverview.htm)



### Installation 

Download and install Tilt:

`brew tap windmilleng/tap && brew install windmilleng/tap/tilt`

Verify your installation: 

`tilt version`



### Local Configuration 

Clone the [Tilt repository](https://github.com/windmilleng/tilt) on GitHub and change to the `/integration/oneup` directory. The directory contains: 

```
$ ls
Dockerfile	Tiltfile	main.go		oneup.yaml
```

- Tilt is configured by means of a `Tiltfile`. In the repository you will see an example `Tiltfile` that will be used to deploy `oneup.yaml` 
- The `Dockerfile` contains information about how to build your `main.go` application 
- `oneup.yaml` is the Kubernetes manifest used to build the image created by the Dockerfile

When deploying to a local cluster, if there is no registry specified in your `tiltfile`, Tilt will bypass the step to push your image to a registry. Tilt uses your current Kubernetes context to determine the cluster on which your application will be deployed. To find your context run: `kubectl config current-context`. 

#### Deploy the Application 

Run `tilt up` to build and deploy the application. The Tilt UI will indicate a successful deployment with a green indicator and a deploy status of `running`. 



#### Verify Your Deployment

Tilt deployed the application in the `tilt-integration` namespace. Verify your application was properly deployed by switching to that namespace and running `kubectl get pods`

```
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
oneup-7f7d69bd66-ckhct   1/1     Running   0          5s
```

#### Connect to the Application 

Connect to your application by typing `B`. This will open your browser and connect to the "OneUp" application based on the port forward indicated in the `Tiltfile`. In this example the browser will open to `http://localhost:8100`.  

#### Modify the Application 

While `tilt up` is running, the build, push, and deploy application will reoccur every time a change is made to your code. 

#### Remove the Application 

When you are finished testing the application, you can exit the UI by typing `Q` and remove the application from your cluster with `tilt down`:

```
$ tilt down
Deleting via kubectl: Namespace/tilt-integration
Deleting via kubectl: Deployment/oneup
```



### Docker Registry and Hosted Kubernetes Configuration 

#### Registry Configuration 

To use a cloud registry service you will need to add the URL of the registry to two files:  `app.yaml` and `tiltfile`. To set OCIR as your registry you will need to provide the server URL of the registry. Here is the URL format:  `<region code>.ocir.io/<tenancy name>/<repo name>/<image name>:<tag>` 

- `<region-code>` is one of `fra`, `iad`, `lhr`, or `phx`.
- `ocir.io` is the Oracle Cloud Infrastructure Registry name.
- `<tenancy-name>` is the name of the tenancy that owns the repository to which you want to push the image, for example `acme-dev`. Note that your user must have access to the tenancy.
- `<repo-name>`, if specified, is the name of a repository to which you want to push the image. For example, `project01`. Note that specifying a repository is optional. If you don't specify a repository name, the name of the image is used as the repository name in Oracle Cloud Infrastructure Registry.
- `<image-name>` is the name you want to give the image in Oracle Cloud Infrastructure Registry, for example, `helloworld`.
- `<tag>` is an image tag you want to give the image in Oracle Cloud Infrastructure Registry, for example, `latest`.

Update the name and image under `spec` in your  `app.yaml` file: 

```
    spec:
      containers:
      - name: [name of the image built by Docker]
        image: <region code>.ocir.io/<tenancy name>/<repo name>/<image name>
```

Update docker_build in your `tiltfile` with the registry URL:

```
docker_build('image: <region code>.ocir.io/<tenancy name>/<repo name>/<image name>', '.')
```

You will also need to log into the registry locally with:

`docker login <region code>.ocir.io`

- Username: `<tenancy-name>/<oci-username>`
- Password: `<oci-auth-token>`

You may also be required to establish trust with the registry. For example, the OCIR registry will be set to private by default. If you would like to continue with a private repository, you will have to add an image pull secret which allows Kubernetes to authenticate with a container registry to pull a private image. For more information about using image secrets, refer to [this guide](https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-and-registry/index.html). A simpler option for testing purposes is to set the registry to be **public**. 

#### Kubernetes Cluster Configuration 

Because Tilt uses your current Kubernetes context to determine the cluster on which your application will be deployed you will need remember to switch to your context to OKE. Verify your context with: `kubectl config current-context`. 

Once the configuration is complete, run `tilt up` to build your application using the Dockerfile, push the application to OCIR, and deploy the application to OKE. As you did before locally, you can verify the successful deployment of the application by changing to the proper namespace and running `kubectl get pods`. Connect to your application with `B`. 



### References

[Tilt User Docs](https://docs.tilt.dev/)

[Tilt GitHub](https://github.com/windmilleng/tilt)
