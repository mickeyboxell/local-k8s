# Skaffold

![skaffold.jpg](https://github.com/GoogleContainerTools/skaffold/blob/master/logo/skaffold.jpg?raw=true)



### Introduction 

I am exploring a series of open source tools to simplify the inner loop of the container native development workflow. This describes the period of time during which you are writing code, but have not yet pushed it to a version control system. These tools, [**Draft**](draft.md), [**Skaffold**](skaffold.md), and [**Tilt**](tilt.md) each take a different approach to the task at hand.  Each tool can be used to build an image of your project, push the image to a registry service of your choice, and deploy the image onto a Kubernetes cluster. Adopting these tools will free up your time and allow you to focus on writing code. You can learn more about the motivation behind this series in my [first post](intro.md). 



### Definition 

[Skaffold](https://github.com/GoogleContainerTools/skaffold) is a command line tool used for local continuous development of Kubernetes applications. Skaffold can be used to automate the workflow for building your application image, pushing it to a repository, and deploying it to a Kubernetes cluster. Skaffold can watch your code for changes with `skaffold dev` and then kick off the build, push, deploy process automatically when you save your code. 



### Differentiators

Skaffold can be used to deploy multiple microservices at once. You can reference multiple images and manifest files in your `skaffold.yaml`. Skaffold is a very flexible option. For example, there are five options for something as straight forward as updating an image repository. You could manually replace the image repository in `skaffold.yaml`, or you can use a flag, environment variable, global Skaffold configuration, or skaffold config for your current `kubectl` context. Skaffold also provides two options for deployment: `skaffold dev` and `skaffold run`. `skaffold dev` continously watches your application for changes and restarts the build, push, deploy pipeline whenever changes occur. Exiting from `skaffold dev` will delete your deployed application. Alternatively, `skaffold run` runs your pipeline once and does not watch for changes. 

Skaffold makes it easy to build, push, and deploy using a variety of tools by means of a **pluggable architecture** and also allows you to easily switch between these configurations with the **profiles** feature. Skaffold gives you options to use the tool you prefer in each stage of the build, push, deploy process. There are many build options (Dockerfile locally, Dockerfile in-cluster with Kaniko, Dockerfile on the cloud, Jib Maven/Gradle locally, etc.), deploy options (`kubectl`, Helm, Kustomize), and many optional image tag policies. This pluggability makes it possible to match the tools used in your production deployment pipeline. 

The profiles feature makes the pluggable architecture of Skaffold even more useful. A profile is a set of settings stored in `skaffold.yaml` that overrides the build, test and deploy sections of your current configuration. Profiles enable you to efficiently switch tools as you see fit depending on your context. You can activate a configuration with `skaffold run -p [PROFILE]`. For example, you can create one profile called `local` development, which uses the local Docker daemon to build images and `kubectl` to deploy them to a local cluster. After you finalize your design, you can switch to the `production` profile using Jib with Maven for your build tool and `helm` to deploy to your cluster. 



### Requirements

- Docker For Desktop
- [Kubectl 1.7.4](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [Oracle Container Engine for Kubernetes (OKE)](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengoverview.htm)
- [Oracle Cloud Infrastructure Registry (OCIR)](https://docs.cloud.oracle.com/iaas/Content/Registry/Concepts/registryoverview.htm)



### Installation 

More detailed installation information can be found on the [Skaffold GitHub page](https://github.com/GoogleContainerTools/skaffold). 

Download and install Skaffold: 

`brew install skaffold`

Verify your installation: 

`skaffold version`

The installation will create a `/.skaffold` folder in your root directory to store configuration information.  



### Local Configuration 

Clone the [Skaffold repository](https://github.com/GoogleContainerTools/skaffold) on GitHub and change to the `/examples/getting-started` directory. The directory includes:

```
$ ls 
Dockerfile	README.adoc	k8s-pod.yaml	main.go		skaffold.yaml
```

- The Dockerfile contains information about how to build your application 
- skaffold.yaml specifies the workflow steps
- K8s-pod.yaml is the Kubernetes manifest used to build the image created by the Dockerfile

When using a local context you can bypass the image push step with the command `skaffold config set --global local-cluster true`. 

Skaffold will use your current Kubernetes context to determine the cluster on which your application will be deployed. To find your context run: `kubectl config current-context`. 

Run `skaffold dev` to build and deploy the application every time a change is made to your code. Exiting from `skaffold dev` will delete your pod. Skaffold also provides the option to build and deploy the application only once with the `skaffold run` command. 

Verify your application was properly deployed with `kubectl get pods`

```
$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
getting-started                          1/1     Running   0          5s
```

#### Connect to the Application 

Skaffold will automatically port forward an application based on the configuration of your pod spec. This will open your browser and connect to the "OneUp" application based on the port forward indicated in the `Tiltfile`. In this example the browser will open to `http://localhost:8100`.  

#### Modify the Application 

While `tilt up` is running, the build, push, and deploy application will reoccur every time a change is made to your code. 

#### Application Logs

`skaffold run --tail`

#### Remove the Application 

When you are finished testing the application, you can exit the UI by typing `Q` and remove the application from your cluster with `tilt down`:

```
$ tilt down
Deleting via kubectl: Namespace/tilt-integration
Deleting via kubectl: Deployment/oneup
```





### OKE/OCIR Configuration 

Skaffold can also be used with hosted Kubernetes solutions. My example will use [Oracle Container Engine for Kubernetes (OKE)](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengoverview.htm) as the Kubernetes cluster and [Oracle Cloud Infrastructure Registry (OCIR)](https://docs.cloud.oracle.com/iaas/Content/Registry/Concepts/registryoverview.htm) as the container image registry. Similar steps can be followed to configure Skaffold with other Kubernetes cluster and registry services. 

#### Registry Configuration 

In order to use a cloud registry service you will need to set the registry in Skaffold with the `skaffold config set default-repo` command. To set OCIR as your registry you will need to provide the server URL of the registry. Run: `skaffold config set default-repo <region code>.ocir.io/<tenancy name>/<repo name>/<image name>:<tag>` 

- `<region-code>` is the code for the OCI region that you’re using. See [Regions and Availability Domains](https://docs.cloud.oracle.com/iaas/Content/General/Concepts/regions.htm).
- `ocir.io` is the OCI registry name.
- `<tenancy-name>` is the name of the tenancy that owns the repository to which you want to push the image.
- `<repo-name>` is optional. It is the name of a repository to which you want to push the image (for example, `project01`). If you don’t specify a repository name, the name of the image is used as the repository name in OCIR.
- `<image-name>` is the name you want to give the image in OCIR (for example, `helloworld`).
- `<tag>` is an image tag you want to assign the image in OCIR (for example, `latest`).

You will need to log into the registry locally with:

`docker login <region code>.ocir.io`

- Username: `<tenancy-name>/<oci-username>`
- Password: `<oci-auth-token>`

You may also be required to establish trust with the registry. For example, the OCIR registry will be set to private by default. If you would like to continue with a private repository, you will have to add an image pull secret which allows Kubernetes to authenticate with a container registry to pull a private image. For more information about using image secrets, refer to [this guide](https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-and-registry/index.html). A simpler option for testing purposes is to set the registry to be **public**. 

#### Kubernetes Cluster Configuration 

Because Skaffold uses your current Kubernetes context to determine the cluster on which your application will be deployed you will need remember to switch to your context to OKE. Verify your context with: `kubectl config current-context`. 

Once the configuration is complete, run `skaffold dev` to build your application using the Dockerfile, push the application to OCIR, and deploy the application to OKE. As you did before locally, you can verify the successful deployment of the application by running `kubectl get pods`. Skaffold will automatically port-forward any ports mentioned in the pod spec.

### References

[Skaffold User Docs](https://skaffold.dev/)

[Skaffold GitHub](https://github.com/GoogleContainerTools/skaffold)

