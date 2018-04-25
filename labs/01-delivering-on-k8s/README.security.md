# Delivering APIs into Kubernetes

## 4) Identifing security drawbacks

### 4.1) Introducing a "malicious" Container

Add and run a "malicious" Container:
```bash
$ kubectl create namespace malicious
$ kubectl run malicious-curl --image=radial/busyboxplus:curl -it -n malicious --replicas=1
If you don't see a command prompt, try pressing enter.
[ root@malicious-curl-6b6f76f5f9-bsl9x:/ ]$ exit
Session ended, resume using 'kubectl attach malicious-curl-6b6f76f5f9-bsl9x -c malicious-curl -i -t' command when the pod is running
```

Check status, delete and re-attach:
```bash
$ kubectl get deploy,pods -n malicious
$ kubectl delete deploy/malicious-curl -n malicious
$ kubectl attach malicious-curl-6b6f76f5f9-bsl9x -c malicious-curl -it -n malicious
```

### 4.2) Gathering information about services living into Kubernetes

NSLookup:
```bash
$ kubectl exec malicious-curl-6b6f76f5f9-bsl9x -n malicious -- nslookup hello-svc-np.hello.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hello-svc-np.hello.svc.cluster.local
Address 1: 10.105.255.213 hello-svc-np.hello.svc.cluster.local
```

Making one HTTP request:
```bash
$ kubectl get svc hello-svc-np -n hello
NAME           TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
hello-svc-np   NodePort   10.105.255.213   <none>        5030:30705/TCP   8h

$ kubectl exec malicious-curl-6b6f76f5f9-bsl9x -n malicious -- curl -sv hello-svc-np.hello.svc.cluster.local:5030/hello
> GET /hello HTTP/1.1
> User-Agent: curl/7.35.0
> Host: hello-svc-np.hello.svc.cluster.local:5030
> Accept: */*
>
< HTTP/1.0 200 OK
< Content-Type: text/html; charset=utf-8
< Content-Length: 54
< Server: Werkzeug/0.12.2 Python/2.7.13
< Date: Wed, 14 Mar 2018 17:42:22 GMT
<
{ [data not shown]
Hello version: v1, instance: hello-v1-69c9685b5-97vqs
```

DoS (flooding of HTTP requests):
```bash
[ root@malicious-curl-6b6f76f5f9-bsl9x:/ ]$ while true; do curl -s -o /dev/null http://hello-svc-np.hello.svc.cluster.local:5030/hello; done
```
We can change the `malicious-curl` Pod from `--replicas=1` to `--replicas=100` and repeat the DoS.


Get System Environment Variables:
```bash
$ kubectl exec hello-v1-69c9685b5-97vqs -n hello -- printenv
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=hello-v1-69c9685b5-97vqs
HELLO_SVC_LB_SERVICE_PORT_HTTP=5020
HELLO_SVC_NP_SERVICE_PORT=5030
HELLO_SVC_NP_PORT=tcp://10.105.255.213:5030
KUBERNETES_SERVICE_PORT=443
HELLO_SVC_CIP_PORT=tcp://10.98.52.87:5010
HELLO_SVC_LB_SERVICE_HOST=10.96.254.211
HELLO_SVC_LB_SERVICE_PORT=5020
HELLO_SVC_NP_SERVICE_PORT_HTTP=5030
HELLO_SVC_NP_PORT_5030_TCP_ADDR=10.105.255.213
HELLO_SVC_CIP_SERVICE_PORT=5010
HELLO_SVC_LB_PORT_5020_TCP_PROTO=tcp
HELLO_SVC_LB_PORT_5020_TCP_PORT=5020
HELLO_SVC_NP_SERVICE_HOST=10.105.255.213
KUBERNETES_PORT_443_TCP_PROTO=tcp
HELLO_SVC_CIP_PORT_5010_TCP=tcp://10.98.52.87:5010
HELLO_SVC_LB_PORT_5020_TCP=tcp://10.96.254.211:5020
HELLO_SVC_NP_PORT_5030_TCP=tcp://10.105.255.213:5030
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
HELLO_SVC_CIP_SERVICE_HOST=10.98.52.87
HELLO_SVC_CIP_PORT_5010_TCP_PORT=5010
HELLO_SVC_CIP_PORT_5010_TCP_ADDR=10.98.52.87
HELLO_SVC_NP_PORT_5030_TCP_PROTO=tcp
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP_PORT=443
HELLO_SVC_NP_PORT_5030_TCP_PORT=5030
HELLO_SVC_CIP_SERVICE_PORT_HTTP=5010
HELLO_SVC_LB_PORT=tcp://10.96.254.211:5020
HELLO_SVC_LB_PORT_5020_TCP_ADDR=10.96.254.211
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
HELLO_SVC_CIP_PORT_5010_TCP_PROTO=tcp
LANG=C.UTF-8
GPG_KEY=C01E1CAD5EA2C4F0B8E3571504C367C218ADD4FF
PYTHON_VERSION=2.7.13
PYTHON_PIP_VERSION=9.0.1
SERVICE_VERSION=v1
HOME=/root
```