# learn-ckad

Learning materials collected while preparing to [Certified Kubernetes Application Developer (CKAD)](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/).

## Learn Kubernetes Basics

### Create a cluster

![](https://d33wubrfki0l68.cloudfront.net/283cc20bb49089cb2ca54d51b4ac27720c1a7902/34424/docs/tutorials/kubernetes-basics/public/images/module_01_cluster.svg)

*source: https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-intro/*

```
minikube version
minikube start

kubectl version
kubectl cluster-info
kubectl get nodes
```

### Deploy an App

```
kubectl version
kubectl get nodes

kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
kubectl get deployments

echo -e "\n\n\n\e[92mStarting Proxy. After starting it will not output a response. Please click the first Terminal Tab\n";
kubectl proxy

curl http://localhost:8001/version

export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/
```

### Explore Your App

```
kubectl get pods
kubectl describe pods

echo -e "\n\n\n\e[92mStarting Proxy. After starting it will not output a response. Please click the first Terminal Tab\n"; kubectl proxy

export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
kubectl logs $POD_NAME

kubectl exec $POD_NAME -- env
kubectl exec -ti $POD_NAME -- bash

cat server.js
curl localhost:8080
```

### Expose Your App Publicly

```
kubectl get pods
kubectl get services

kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
kubectl get services
kubectl describe services/kubernetes-bootcamp

export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
curl $(minikube ip):$NODE_PORT

kubectl describe deployment
kubectl get pods -l app=kubernetes-bootcamp
kubectl get services -l app=kubernetes-bootcamp
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME
kubectl label pods $POD_NAME version=v1
kubectl describe pods $POD_NAME
kubectl get pods -l version=v1

kubectl delete service -l app=kubernetes-bootcamp
kubectl get services
curl $(minikube ip):$NODE_PORT
kubectl exec -ti $POD_NAME -- curl localhost:8080
```

### Scale Your App

```
kubectl get deployments
kubectl get rs
kubectl scale deployments/kubernetes-bootcamp --replicas=4
kubectl get deployments
kubectl get pods -o wide
kubectl describe deployments/kubernetes-bootcamp

kubectl describe services/kubernetes-bootcamp
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
curl $(minikube ip):$NODE_PORT

kubectl scale deployments/kubernetes-bootcamp --replicas=2
kubectl get deployments
kubectl get pods -o wide
```

### Update Your App

```
kubectl get deployments
kubectl get pods
kubectl describe pods
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
kubectl get pods

kubectl describe services/kubernetes-bootcamp
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
curl $(minikube ip):$NODE_PORT
kubectl rollout status deployments/kubernetes-bootcamp
kubectl describe pods

kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
kubectl get deployments
kubectl get pods
kubectl describe pods
kubectl rollout undo deployments/kubernetes-bootcamp
kubectl get pods
kubectl describe pods
```

## Notes from books

### [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)

```
kubectl run my-pod-name --image=docker-image-name --restart=Never
kubectl wait --for=condition=Ready pod my-pod-name

kubectl get pod my-pod-name --output custom-columns=NAME:metadata.name,NODE_IP:status.hostIP,POD_IP:status.podIP,LABELS:metadata.labels
kubectl get pod my-pod-name -o jsonpath='{.status.containerStatuses[0].containerID}'

kubectl label pods -l app=existing-label-value --overwrite app=new-label-value
kubectl get pods -l app=new-label-value

kubectl port-forward pod/my-pod-name 8080:80

kubectl exec -it my-pod-name -- sh
hostname -i
wget -O - http://localhost | head -n 4

kubectl logs --tail=2 my-pod-name
kubectl logs -l app=my-label-value
kubectl logs -l app=my-label-value -c my-container-name-in-multi-conatiner-pod

kubectl create deployment my-deployment-name --image=docker-image-name

kubectl exec deploy/my-deployment-name-1 -- sh -c 'wget -O - http://localhost > /dev/null'
kubectl logs --tail=1 -l app=my-deployment-name-1

kubectl delete deploy --all
kubectl get all
```

### [Service](https://kubernetes.io/docs/concepts/services-networking/)

```
kubectl expose -f file.yaml --type LoadBalancer --port 8080 --target-port 80
kubectl get svc my-service-name
kubectl get svc my-service-name -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'
kubectl get endpoints my-service-name
kubectl exec my-pod-name -- sh -c 'nslookup my-service-name | grep "^[^*]"'
kubectl exec my-pod-name -- sh -c 'nslookup kube-dns.kube-system.svc.cluster.local | grep "^[^*]"'
kubectl delete my-service-name
```

```
apiVersion: v1
kind: Service
metadata:
 name: my-service-name
spec:
 ports:
   - port: 8080          # on this port service is available to other pods (inside Kubernetes cluster)
     targetPort: 80      # on this port traffic is sent (on this port application in container works)
     nodePort: 30080     # on this port service is available externally (outside Kubernetes cluster)
 selector:
 app: my-web-app
 type: NodePort          # This Service is available on node IP addresses.
```

### [ConfigMap and Secrets](https://kubernetes.io/docs/concepts/configuration/)

```
kubectl create configmap my-config-map --from-literal=setting.name='setting.value'
kubectl get cm my-config-map
kubectl exec deploy/my-deploy-name -- sh -c 'printenv | grep "^SETTING"'
```

```
apiVersion: v1
kind: ConfigMap
metadata:
 name: my-app-config-file
data:
 config.json: |-
   {
     "Configuration": {
       "Enabled" : true
     }
   }
 logging.json: |-
   {
     "Logging": {
       "LogLevel": "Debug"
     }
   }
```

```
spec:
 containers:
   - name: my-app-name
     image: my-image-name
     volumeMounts:
       - name: config
         mountPath: "/app/config"
         readOnly: true

 volumes:
   - name: config
     configMap:
       name: my-app-config-file
```

```
kubectl create secret generic my-secret --from-literal=secret=sensitivevaue
kubectl get secret my-secret -o jsonpath='{.data.secret}'
kubectl get secret my-secret -o jsonpath='{.data.secret}' | base64 -d
```

```
spec:
 containers:
   - name: my-app-name
     image: my-image-name
     env:
     - name: MY_SECRET
       valueFrom:
         secretKeyRef:
           name: my-secret
           key: secret
```

```
apiVersion: v1
kind: Secret
metadata:
 name: my-secret-name
type: Opaque
stringData:
 DB_PASSWORD: "***"
```

```
kubectl get secret my-secret-name -o jsonpath='{.data.DB_PASSWORD}'
```

```
spec:
 containers:
   - name: my-app-name
     image: my-image-name
     env:
     - name: DB_PASSWORD_FILE
       value: /secrets/db_password
     volumeMounts:
       - name: secret
         mountPath: "/secrets"
 volumes:
   - name: secret
     secret:
       secretName: my-secret-name
       defaultMode: 0400
       items:
       - key: DB_PASSWORD
         path: db_password
```

```
kubectl exec deploy/my-app-name -- sh -c 'ls -l $(readlink -f /secrets/db_password)'
```

### [Volume and claims](https://kubernetes.io/docs/concepts/storage/)

[emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir):

```
spec:
 containers:
   - name: my-pod-name
     image: my-image-name
     volumeMounts:
      - name: data
         mountPath: /data
 volumes:
   - name: data
     emptyDir: {}
```

[hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath):

```
spec:
 containers:
  - image: nginx
    name: nginx
    ports:
       - containerPort: 80
    volumeMounts:
       - name: cache-volume
         mountPath: /data/nginx/cache
 volumes:
   - name: cache-volume
     hostPath:
       path: /volumes/nginx/cache
       type: DirectoryOrCreate
```

[NSF volume](https://kubernetes.io/docs/concepts/storage/volumes/#nfs):

```
apiVersion: v1
kind: PersistentVolume
metadata:
 name: my-pv-name

spec:
 capacity:
   storage: 100Mi
 accessModes:
   - ReadWriteOnce

 nfs:
   server: my-nfs-server.example.com
   path: /my-nfs-volume
```

[PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/):

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: my-pvc-name
spec:
 accessModes:
   - ReadWriteOnce
 resources:
   requests:
     storage: 40Mi
 storageClassName: "" # if there is no storageClassName field, then this uses the default class
```

```
spec:
 containers:
   - name: my-pod-name
     image: nginx
     volumeMounts:
       - name: data
         mountPath: /data
 volumes:
   - name: data
     persistentVolumeClaim:
       claimName: my-pvc-name
```

```
kubectl get pvc
kubectl get pv
kubectl get sc
```

### [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/):

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
 name: my-replica-name
spec:
 replicas: 1
 selector:
   matchLabels:
     app: my-app-label
 template:
```

```
kubectl get rs
```

### [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/):

```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: my-deployment-name
spec:
 replicas: 2
selector:
 matchLabels:
   app: my-app-label
 template:
```

```
kubectl get deployment

kubectl scale --replicas=4 deploy/my-deployment-name
kubectl get rs -l app=my-app-label
kubectl get po -l app=my-app-label --show-labels
```

### [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
 name: my-daemon-name
spec:
 selector:
   matchLabels:
     app: my-app-label
template:
 metadata:
   labels:
     app: my-app-label
spec:
```

```
kubectl get ds
kubectl delete ds my-daemon-name --cascade=false
kubectl get po -l app=my-app-label
kubectl delete ds
```

### [Init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/):

```
spec:
 initContainers:
   - name: my-init-container-name
     image: image
     command: ['sh', '-c', "# some command to initalize application in /data"]
     volumeMounts:
    - name: data
      mountPath: /data
```

```
initContainers:
 - name: configure-app
# ...
containers:
 - name: legacy-app
# ...
 - name: logger-app
# ...
 - name: health-app
# ...
   ports:
     - containerPort: 8091
 - name: metrics-app
# ...
   ports:
     - containerPort: 8092
```

### [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/):

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
 name: my-statefulset-name
spec:
 selector:
   matchLabels:
     app: my-app-label
 serviceName: my-service-name
 replicas: 2
 template:
```

```
kubectl get statefulset
```

### [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)

```
apiVersion: batch/v1
kind: Job
metadata:
 name: my-job-name
spec:
 template:
   spec:
     containers:
       - name: my-container
         image: my-image
         command: ["java", "-jar", "app.jar"]
     restartPolicy: Never
```

```
kubectl get job
```

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
 name: my-app-backup
spec:
 schedule: "*/5 * * * *"
 concurrencyPolicy: Forbid
 jobTemplate:
   spec:
```

```
kubectl get cronjob
```

### [Rolling Back a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)

```
kubectl rollout history deploy/my-deployment-name
kubectl rollout status deploy/my-deployment-name

kubectl get rs -l app=my-app-label -o=custom-columns=NAME:.metadata.name,REPLICAS:.status.replicas,REVISION:.metadata.annotations.deployment\.kubernetes\.io/revision

kubectl rollout undo deploy/my-deployment-name --dry-run
kubectl rollout undo deploy/my-deployment-name --to-revision=1
kubectl rollout status deploy/my-deployment-name  --timeout=1s
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: vweb
spec:
 replicas: 3
 strategy:
   type: Recreate # by default for deployment there is strategy RollingUpdate
```

### [Helm](https://helm.sh/docs/)

```
helm repo add myrepo https://myrepo
helm repo update

helm search repo myapp --versions
helm show values myrepo/myapp --version 1.0.0

helm install --set servicePort=8010 --set replicaCount=1 myapp-name myrepo/myapp --version 1.0.0
helm upgrade --set servicePort=8010 --set replicaCount=3 myapp-name myrepo/myapp --version 1.0.0
helm upgrade --reuse-values --atomic myapp-name myrepo/myapp --version 1.0.0
helm uninstall myapp-name
helm uninstall $(helm ls -q)

helm history myapp-name
helm rollback myapp-name 2
helm get values myapp-name
helm ls

helm lint mychart-local-folder
helm install myapp-name mychart-local-folder/
helm package mychart-local-folder
```

### [Context](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

```
kubectl config get-contexts
kubectl config set-context --current --namespace=my-namespace
kubectl config set-context --current --namespace=
kubectl config view
```

## [Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

```
spec:
 containers:
    - image: image-name
      readinessProbe:
        httpGet:
          path: /healthz
          port: 80
        periodSeconds: 3
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        periodSeconds: 10
        initialDelaySeconds: 10
        failureThreshold: 2
```

```
kubectl wait --for=condition=ContainersReady pod -l app=my-app-name
```

```
spec:
 containers:
   - image: postgres
     readinessProbe:
       tcpSocket:
         port: 5432
       periodSeconds: 3
     livenessProbe:
       exec:
         command: ["pg_isready", "-h", "localhost"]
       periodSeconds: 10
       initialDelaySeconds: 10
```

```
spec:
 containers:
   - image: image-name
     resources:
       limits:
         memory: 50Mi
         cpu: 250m

```

### [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
 name: my-ingress
spec:
 rules:
 - http:
     paths:
     - path: /
       backend:
         serviceName: my-service-name1
         servicePort: 80
     - path: /app2
       backend:
         serviceName: my-service-name2
         servicePort: 80
```

```
kubectl get ingress
```

### [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: my-policy-name
spec:
 podSelector:
   matchLabels:
     app: my-app-label
 ingress:
 - from:
   - podSelector:
       matchLabels:
         app: my-db-label
   ports:
   - port: app
```

```
kubectl get networkpolicy
```

### [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

```
spec:
 automountServiceAccountToken: false
 securityContext:
   runAsUser: 65534
   runAsGroup: 3000
 containers:
   - image: my-image-name
     securityContext:
       allowPrivilegeEscalation: false
       capabilities:
         drop:
           - all
```

// LKLM -
// KA - 1
// ACKAE - 1

## Links

* [Kubernetes Documentation](https://kubernetes.io/docs/home/)
* [Concepts](https://kubernetes.io/docs/concepts/)
* [Tutorials](https://kubernetes.io/docs/tutorials/)
* [My notes e.g. commands from course "Kubernetes po polsku"](https://github.com/sebastianczech/DevOps-Engineer)
* [Best Kubernetes Certifications for 2023](https://devopscube.com/best-kubernetes-certifications/)
* [Open Source Curriculum for CNCF Certification Courses](https://github.com/cncf/curriculum)
* [CKAD Exercises](https://github.com/dgkanatsios/CKAD-exercises)
* [Killer Shell CKAD](https://killercoda.com/killer-shell-ckad)
* [Kubernetes Exam Simulator](https://killer.sh/)
* [CKAD Exam Study Guide: A Complete Resource for CKAD Aspirants](https://devopscube.com/ckad-exam-study-guide/)
* [Answers to Five Certified Kubernetes Application Developer CKAD Exam Questions (2021)](https://thospfuller.com/2020/11/09/answers-to-five-kubernetes-ckad-practice-questions-2021/)
* [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)