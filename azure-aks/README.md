## Jenkins on Azure Kubernetes

First of all, you’ll need an account at portal.azure.com. Microsoft provides credit for new registration. If you have MSDN subscription you’ll get some credit which is available at portal my.visualstudio.com.

Software prerequisites:

- Install Docker Desktop (not Desktop Enterprise). 
- The installation also contains kubectl command from Kubernetes.
- Install [Helm](https://helm.sh/docs/intro/install/)
- Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

> If you want to create the resources from your cli, you should log in by az cli.

These are the steps from azure cloud shell.

```
$ az group create --resource-group jenkins-group --location eastus
$ az aks create -n kube-jenkins --resource-group jenkins-group --node-count 1 --node-vm-size Standard_B2ms --generate-ssh-keys
```

Now, we have to retrieve credentials for working with Kubernetes cluster on Azure.

` $ az aks get-credentials -g jenkins-group -n kube-jenkins`

Switch to newly created Kubernetes.

``` 
kubectl config use-context kube-jenkins 
kubectl create ns jenkins
```

Install Jenkins on Kubernetes by Helm.

``` 
$ helm repo add jenkinsci https://charts.jenkins.io 
$ helm repo update
$ helm install jenkinsci jenkinsci/jenkins --namespace jenkins
```

Output
```
WARNING: "kubernetes-charts.storage.googleapis.com" is deprecated for "stable" and will be deleted Nov. 13, 2020.
WARNING: You should switch to "https://charts.helm.sh/stable"
NAME: jenkinsci
LAST DEPLOYED: Thu May  6 21:20:19 2021
NAMESPACE: jenkins
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace jenkins -it svc/jenkinsci -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  echo http://127.0.0.1:8080
  kubectl --namespace jenkins port-forward svc/jenkinsci 8080:8080

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```

Validating installation

```
$ kubectl get pod -n jenkins
NAME          READY   STATUS     RESTARTS   AGE
jenkinsci-0   0/2     Init:0/1   0          2m3s

$ kubectl get pod -n jenkins
NAME          READY   STATUS    RESTARTS   AGE
jenkinsci-0   2/2     Running   0          3m18s

$ kubectl get svc -n jenkins
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
jenkinsci         ClusterIP   10.0.175.83    <none>        8080/TCP    23m
jenkinsci-agent   ClusterIP   10.0.221.114   <none>        50000/TCP   23m
```
Checking

Now, we should expose the service by external-ip, by default this service is ClusterIP where just can be accesed by other cluster nodes.

$ kubectl edit svc -n jenkins jenkinsci
#replace LoadBalancer rather than ClusterIP.

Now, we can access via external ip.

```
$ kubectl get svc -n jenkins -w
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
jenkinsci         LoadBalancer   10.0.175.83    52.224.23.88   8080:32238/TCP   24m
jenkinsci-agent   ClusterIP      10.0.221.114   <none>         50000/TCP        24m
```

## Jenkins behind Ingress Controller

You can add the prefix "/jenkins" since the installation by helm.

> $ helm install jenkins jenkinsci/jenkins --set controller.jenkinsUriPrefix='/jenkins'

Or, if you've already installed Jenkins by helm

> helm upgrade jenkins jenkinsci/jenkins --set controller.jenkinsUriPrefix='/jenkins'

Finally, if you want to update the prefix without helm command, you should update the statefulset adding.

```
    env:
    - name: JENKINS_OPTS
      value: --prefix=/jenkins
```

Then, in the manifest ingress controller:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  namespace: jenkins
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /jenkins
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
```


Now, we can acces via browser by the external ip with the port 8080.

Getting admin password
```
$ kubectl exec --namespace jenkins -it svc/jenkinsci -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
```
Output
> u1BJzCjFlkXErrlswPSVzY

We must copy and paste it in the Jenkins login.