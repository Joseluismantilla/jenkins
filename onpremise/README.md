
## Create Jenkins with deployment file (just for a replica)

The **onpremise/jenkins-deployment.yaml** file in the “jenkins” namespace is all you need and my recommendation.

- This deployment file is defining a Deployment as indicated by the kind field using a single replica and the service expose.

- The container image name is jenkins and version is 2.32.2

- The list of ports specified within the spec are a list of ports to expose from the container on the Pods IP address.

        Jenkins running on (http) port 8080.
        The Pod exposes the port 8080 of the jenkins container.

- The volumeMounts section of the file creates a Persistent Volume. This volume is mounted within the container at the path /var/jenkins_home and so modifications to data within /var/jenkins_home are written to the volume. The role of a persistent volume is to store basic Jenkins data and preserve it beyond the lifetime of a pod.

- The Service is of type **NodePort**. Other options are ClusterIP (only accessible within the cluster) and LoadBalancer (IP address assigned by a cloud provider e.g. AWS Elastic IP) but this is not our case.



Let's go to apply the manifest file in our kubernetes cluster.

```
$ kubectl create ns jenkins
$ kubectl create -f onpremise/jenkins-deployment -n jenkins
```
Now, you will see:
```
$ kubectl get services -n jenkins
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP    PORT(S)           AGE
jenkins    NodePort    10.103.31.217    <none>         8080:32664/TCP    59s
```

if you want to update the prefix without helm command you can edit the deployment adding.

```
    env:
    - name: JENKINS_OPTS
      value: --prefix=/jenkins
    volumeMounts: ....
```

## Accessing to the Jenkins dashboard

If you are in minukube you can use:

``` $ minikube ip
  192.168.99.100 
```
> If you aren't using minikube, you can replace the ip by localhost.

Now, let's go to the url **ip:32664** by browser.

The initial admin password is required, you should following the next steps:

```
$ kubectl get pods -n jenkins
$ kubectl logs <pod_name> -n jenkins
```

The password is at the end of the log, just copy and paste it.
