# Example Application

## TL;DR

```bash
$ kubectl create -f https://raw.githubusercontent.com/bitnami/bitnami-docker-java/master/example/kubernetes.yml
```

## Introduction

This example demostrates the use of the `bitnami/java` image to create a production build of your java application.

For demonstration purposes we will use the [Jenkins](https://jenkins.io/) application, build a image with the tag `bitnami/java-example` and deploy it on a [Kubernetes](https://kubernetes.io) cluster.

## Build and Test

To build a production Docker image of our application we'll use the `bitnami/java:1.8-prod` image, which is a production build of the Bitnami JavaB Image optimized for size.

```Dockerfile
FROM bitnami/java:1.8 as builder
WORKDIR /app
RUN wget http://mirrors.jenkins.io/war-stable/latest/jenkins.war

FROM bitnami/java:1.8-prod
COPY --from=builder /app /app
WORKDIR /app
EXPOSE 8080
CMD ["java", "-jar", "jenkins.war"]
```

The `Dockerfile` consists of two build stages.

* The first stage uses the development image, `bitnami/java:1.8`, to download the application.
* The second stage uses the production image, `bitnami/java:1.8-prod`, and copies over the application. This creates a minimal Docker image that only consists of the application source and the java runtime.

To build the Docker image, execute the command:

```bash
$ docker build -t bitnami/java-example:0.0.1 example/
```

Since the `bitnami/java:1.8-prod` image is optimized for production deployments it does not include any packages that would bloat the image.

```console
$ docker image ls
REPOSITORY                          TAG                    IMAGE ID            CREATED             SIZE
bitnami/java-example                0.0.1                  b6ad4097d596        About an hour ago   339MB
```

You can now launch and test the image locally.

```console
$ docker run -it --rm -p 8080:8080 bitnami/java-example:0.0.1

...
INFO: Jenkins is fully up and running
```

Finally, push the image to the Docker registry

```bash
$ docker push bitnami/java-example:0.0.1
```

## Deployment

The `kubernetes.yml` file from the `example/` folder can be used to deploy our `bitnami/java-example:0.0.1` image to a Kubernetes cluster.

Simply download the Kubernetes manifest and create the Kubernetes resources described in the manifest using the command:

```console
$ kubectl create -f kubernetes.yml
ingress "example-ingress" created
service "example-svc" created
configmap "example-configmap" created
persistentvolumeclaim "example-data-pvc" created
deployment "example-deployment" created
```

From the output of the above command you will notice that we create the following resources:

 - [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
 - [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
 - [Volume](https://kubernetes.io/docs/concepts/storage/volumes/)
    + [ConfigMap](https://kubernetes.io/docs/concepts/storage/volumes/#projected)
    + [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim)
 - [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

> **Note**
>
> Our example application is stateless and does not store any data or does not require any user configurations. As such we do not need to create the ConfigMap or PersistentVolumeClaim resources. Our kubernetes.yml creates these resources strictly to demostrate how they are defined in the manifest.

## Accessing the application

Typically in production you would access the application via a Ingress controller. Our `kubernetes.yml` already defines a `Ingress` resource. Please refer to the [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) documentation to learn how to deploy an ingress controller in your cluster.

> **Hint**
>
> https://kubeapps.com/charts/stable/nginx-ingress

The following are alternate ways of accessing the application, typically used during application development and testing.

Since the service `example-svc` is defined to be of type `NodePort`, we can set up port forwarding to access our web application like so:

```bash
$ kubectl port-forward $(kubectl get pods -l app=example -o jsonpath="{ .items[0].metadata.name }") 8080:8080
```

The command forwards the local port `8080` to port `8080` of the Pod container. You can access the application by visiting the http://localhost:8080.

> **Note:**
>
> If your using minikube, you can access the application by simply executing the following command:
>
> ```bash
> $ minikube service example-svc
> ```

## Health Checks

The `kubernetes.yml` manifest defines default probes to check the health of the application. For our application we are simply probing if the application is responsive to queries on the root resource.

You application can define a route, such as the commonly used `/healthz`, that reports the application status and use that route in the health probes.
