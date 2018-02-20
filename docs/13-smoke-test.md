# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
EXTERNAL_IP=$(az network public-ip show -n k8s-ctrl-0PublicIP --query ipAddress -o tsv) && ssh -t $EXTERNAL_IP \
"ETCDCTL_API=3 etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a fe a0 2e a5 ae 80 19  |:v1:key1:.......|
00000050  87 bf 26 e5 80 2d 72 75  e5 d8 b2 53 31 ea bc 48  |..&..-ru...S1..H|
00000060  15 eb f0 84 0f 42 57 56  40 b4 b1 92 11 a8 08 1c  |.....BWV@.......|
00000070  ab bf d0 b9 b3 f7 dc 98  ce 27 4f 24 dd 3f a0 4a  |.........'O$.?.J|
00000080  74 57 fb 67 fc b9 6f 3a  bf f3 6d 31 04 86 0a 40  |tW.g..o:..m1...@|
00000090  34 c9 63 b9 17 8a ab 4e  e0 a2 36 69 c0 20 7f 9e  |4.c....N..6i. ..|
000000a0  56 c1 bf 58 50 a0 c8 a8  f5 2a b1 4d 88 a6 bf 01  |V..XP....*.M....|
000000b0  b0 07 3a bd 12 4a 2a 16  4c 94 b6 60 b9 d0 cb 90  |..:..J*.L..`....|
000000c0  87 31 49 b9 f4 ab d8 3e  ec a0 76 77 74 54 f5 d1  |.1I....>..vwtT..|
000000d0  55 12 6b 96 ae 26 49 17  5b 85 55 e2 47 bf e8 f1  |U.k..&I.[.U.G...|
000000e0  60 20 fa 8c 21 5b 55 2a  2d 0a                    |` ..![U*-.|
000000ea
Connection to 52.233.193.13 closed.
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl run nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l run=nginx
```

> output

```
NAME                     READY     STATUS    RESTARTS   AGE
nginx-4217019353-b5gzn   1/1       Running   0          15s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.13.7
Date: Mon, 18 Dec 2017 14:50:36 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 Nov 2017 14:28:04 GMT
Connection: keep-alive
ETag: "5a1437f4-264"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
127.0.0.1 - - [20/Feb/2018:13:16:52 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.167 Safari/537.36" "-"
127.0.0.1 - - [20/Feb/2018:13:16:52 +0000] "GET /robots.txt HTTP/1.1" 404 571 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.167 Safari/537.36" "-"
2018/02/20 13:16:52 [error] 5#5: *3 open() "/usr/share/nginx/html/robots.txt" failed (2: No such file or directory), client: 127.0.0.1, server: localhost, request: "GET /robots.txt HTTP/1.1", host: "localhost:8080"
2018/02/20 13:16:52 [error] 5#5: *2 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 127.0.0.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "localhost:8080", referrer: "http://localhost:8080/"
127.0.0.1 - - [20/Feb/2018:13:16:52 +0000] "GET /favicon.ico HTTP/1.1" 404 571 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.167 Safari/537.36" "-"```
```
### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.13.8
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Create a firewall rule that allows remote access to the `nginx` node port:

```
az network nsg rule create --nsg-name k8s-nsg -n k8s-nginx --priority 1050 \
--source-address-prefixes 0.0.0.0/0 --source-port-ranges  0-65535 \
--destination-address-prefixes VirtualNetwork --destination-port-ranges ${NODE_PORT} --protocol Tcp --access Allow
```

Retrieve the external IP address of  worker instance:

```
EXTERNAL_IP=$(az network public-ip show -n k8s-worker-1PublicIP --query ipAddress -o tsv)
```

Make an HTTP request using the external IP address and the `nginx` node port:
> It can take about 1 to 2 minutes for the NSG to become open. So don't panic if the curl outputs 'Connection refused'.
```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.13.7
Date: Mon, 18 Dec 2017 14:52:09 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 Nov 2017 14:28:04 GMT
Connection: keep-alive
ETag: "5a1437f4-264"
Accept-Ranges: bytes
```

Next: [Cleaning Up](14-cleanup.md)
