# Connecting Applications with Services
[The Kubernetes model for connecting containers](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/)
[Namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/)

## 1. Create namespace.
```bash
$ kubectl get namespaces
$ ls -lah admin/
-rwxrwxrwx 1 root root  147 Feb 23 14:29 namespace-dev.json
-rwxrwxrwx 1 root root  145 Feb 23 14:29 namespace-prod.json

$ kubectl create -f admin/
namespace/development created
namespace/production created

$ kubectl get namespaces --show-labels
NAME          STATUS    AGE       LABELS
default       Active    75d       <none>
development   Active    9s        name=development
kube-public   Active    75d       <none>
kube-system   Active    75d       <none>
production    Active    9s        name=production

$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: http://localhost:30000
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/rwibawa/.minikube/client.crt
    client-key: /home/rwibawa/.minikube/client.key

$ kubectl config current-context
minikube

$ kubectl config set-context dev --namespace=development \
> --cluster=minikube \
> --user=minikube
Context "dev" created.

$ kubectl config set-context prod --namespace=production \
> --cluster=minikube \
> --user=minikube
Context "prod" created.

$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: http://localhost:30000
  name: minikube
contexts:
- context:
    cluster: minikube
    namespace: development
    user: minikube
  name: dev
- context:
    cluster: minikube
    namespace: production
    user: minikube
  name: prod
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube

$ kubectl config use-context dev
Switched to context "dev".

```

## 2. Exposing pods to the cluster.
`run-my-nginx.yaml`
```bash
$ kubectl create -f dev/run-my-nginx.yaml
deployment.apps/my-nginx created

$ kubectl get pods -l run=my-nginx -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP            NODE
my-nginx-77f56b88c8-ldw7r   1/1       Running   0          27s       172.17.0.23   minikube
my-nginx-77f56b88c8-m5s96   1/1       Running   0          27s       172.17.0.22   minikube

$ kubectl get pods -l run=my-nginx -o yaml | grep podIP
    podIP: 172.17.0.23
    podIP: 172.17.0.22
```

Links:
* [http://127.0.0.1:30000/api/v1/namespaces/development/pods/my-nginx-77f56b88c8-ldw7r/proxy/](http://127.0.0.1:30000/api/v1/namespaces/development/pods/my-nginx-77f56b88c8-ldw7r/proxy/)
* [http://127.0.0.1:30000/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/#!/overview?namespace=development](http://127.0.0.1:30000/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/#!/overview?namespace=development)


## 3. Creating a Service.
```bash
$ kubectl expose deployment/my-nginx
service/my-nginx exposed

# Equivalent to:
# kubectl create -f dev/nginx-svc.yaml

$ kubectl get svc my-nginx
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.96.116.99   <none>        80/TCP    3m

$ kubectl describe svc my-nginx
Name:              my-nginx
Namespace:         development
Labels:            run=my-nginx
Annotations:       <none>
Selector:          run=my-nginx
Type:              ClusterIP
IP:                10.96.116.99
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         172.17.0.22:80,172.17.0.23:80
Session Affinity:  None
Events:            <none>

$ kubectl get ep my-nginx
NAME       ENDPOINTS                       AGE
my-nginx   172.17.0.22:80,172.17.0.23:80   7m

# Environment Variables
$ kubectl exec my-nginx-77f56b88c8-m5s96 -- printenv | grep SERVICE
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
ryan@hela:~$ curl http://10.96.0.1
```
Links:
* [http://127.0.0.1:30000/api/v1/namespaces/development/services/http:my-nginx:/proxy/](http://127.0.0.1:30000/api/v1/namespaces/development/services/http:my-nginx:/proxy/)

The replicas were created before the Service, so they must be killed and restarted.
```bash
$ kubectl scale deployment my-nginx --replicas=0; kubectl scale deployment my-nginx --replicas=2;
deployment.extensions/my-nginx scaled
deployment.extensions/my-nginx scaled

$ kubectl get pods -l run=my-nginx -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP            NODE
my-nginx-77f56b88c8-9kfcd   1/1       Running   0          4m        172.17.0.22   minikube
my-nginx-77f56b88c8-k6rl5   1/1       Running   0          4m        172.17.0.23   minikube

$ kubectl exec my-nginx-77f56b88c8-9kfcd -- printenv | grep SERVICE
MY_NGINX_SERVICE_HOST=10.96.116.99
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
MY_NGINX_SERVICE_PORT=80
```


## 4. DNS.
```bash
$ kubectl get services kube-dns --namespace=kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   75d

$ kubectl run curl --image=radial/busyboxplus:curl -i --tty
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
[ root@curl-775f9567b5-k6d8b:/ ]$ nslookup my-nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-nginx
Address 1: 10.96.116.99 my-nginx.development.svc.cluster.local
[ root@curl-775f9567b5-k6d8b:/ ]$ curl http://10.96.116.99
<!DOCTYPE html>
<html>
</html>
[ root@curl-775f9567b5-k6d8b:/ ]$ curl http://my-nginx
<!DOCTYPE html>
<html>
</html>
[ root@curl-775f9567b5-k6d8b:/ ]$ exit
Session ended, resume using 'kubectl attach curl-775f9567b5-k6d8b -c curl -i -t' command when the pod is running

$ kubectl attach curl-775f9567b5-k6d8b -c curl -i -t
If you don't see a command prompt, try pressing enter.
[ root@curl-775f9567b5-k6d8b:/ ]$ exit
Session ended, resume using 'kubectl attach curl-775f9567b5-k6d8b -c curl -i -t' command when the pod is running

$ kubectl delete deployment curl
deployment.extensions "curl" deleted
```

## 5. Securing the Service.
```bash
#create a public private key pair
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout cert/nginx.key -out cert/nginx.crt -subj "/CN=my-nginx/O=my-nginx"
Generating a 2048 bit RSA private key
...+++
...................................................+++
writing new private key to 'cert/nginx.key'
-----

$ cat cert/nginx.crt | base64 > dev/nginxsecrets.yaml
$ cat cert/nginx.key | base64 >> dev/nginxsecrets.yaml
$ vi dev/nginxsecrets.yaml

$ kubectl create -f dev/nginxsecrets.yaml
secret/nginxsecret created

$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
default-token-dnfqt   kubernetes.io/service-account-token   3         2h
dpr-secret            kubernetes.io/dockerconfigjson        1         2h
nginxsecret           Opaque                                2         3s

$ kubectl delete deployments,svc my-nginx; kubectl create -f dev/nginx-secure-app.yaml
deployment.extensions "my-nginx" deleted
service "my-nginx" deleted
service/my-nginx created
deployment.apps/my-nginx created

$ kubectl get svc
NAME       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
my-nginx   NodePort   10.110.163.194   <none>        8080:30692/TCP,443:30481/TCP   1m

$ curl -k https://$(minikube ip):30481
```
The parameter `-k` is to tell curl to ignore the CName mismatch.


### Testing the CName
```bash
$ kubectl create -f dev/curlpod.yaml
deployment.apps/curl-deployment created

$ kubectl get pods -l app=curlpod
NAME                               READY   STATUS    RESTARTS   AGE
curl-deployment-765cdd8b76-mg2nc   1/1     Running   0          3m

$ kubectl exec curl-deployment-765cdd8b76-mg2nc -- curl https://my-nginx --cacert /etc/nginx/ssl/nginx.crt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
<html>
```
### Get the external IP
```bash
$ kubectl get svc my-nginx -o yaml | grep nodePort -C 5
spec:
  clusterIP: 10.110.163.194
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 30692
    port: 8080
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 30481
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    run: my-nginx

$ kubectl get nodes -o yaml | grep ExternalIP -C 1

$ kubectl edit svc my-nginx
# Change the NodePort to LoadBalancer
```


## 6. Exposing the Service with traefik and ingress.
```bash
$ curl -O https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-deployment.yaml
$ mv traefik-deployment.yaml dev/

$ kubectl create -f dev/traefik-deployment.yaml
serviceaccount/traefik-ingress-controller created
deployment.extensions/traefik-ingress-controller created

$ kubectl create -f dev/ingress.yaml
ingress.extensions/my-nginx-ingress created
```

* [StackOverflow](https://stackoverflow.com/questions/39850819/kubernetes-minikube-external-ip-does-not-work)
* [Ingress Controller](https://medium.com/@rothgar/exposing-services-using-ingress-with-on-prem-kubernetes-clusters-f413d87b6d34#.bnawbxr29)
