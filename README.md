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

```
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  labels:
    purpose: demonstrate-command
spec:
  containers:
  - name: command-demo-container
    image: debian
    command: ["printenv"]
    args: ["HOSTNAME", "KUBERNETES_PORT"]
  restartPolicy: OnFailure
```

```
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/node-hello:1.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
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
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
  # You set volumes at the Pod level, then mount them into containers inside that Pod
  - name: config
    configMap:
      # Provide the name of the ConfigMap you want to mount.
      name: game-demo
      # An array of keys from the ConfigMap to create as files
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "user-interface.properties"  
```

```
echo -n 'plaintext' | base64
echo -n 'cGxhaW50ZXh0' | base64 --decode

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

```
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_Host: sql01
  DB_User: root
  DB_Password: password123
```

```
apiVersion: v1
kind: Pod
metadata:
  name: my-app-name
spec:
  containers:
  - image: my-image-name
    name: my-app-name
    envFrom:
    - secretRef:
        name: db-secret
........
```

```
apiVersion: v1
kind: Pod
metadata:
  name: my-app-name
spec:
  containers:
  - image: my-image-name
    name: my-app-name
    env:
    - name: DB_Host
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_Host
    - name: DB_User
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_User
    - name: DB_Password
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_Password
........
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

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

### [Communicate Between Containers in Pod](https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:

  restartPolicy: Never

  volumes:
  - name: shared-data
    emptyDir: {}

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
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
kubectl rollout pause deploy/my-deployment-name
kubectl rollout resume deploy/my-deployment-name
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

### [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

```
kubectl api-versions
kubectl get clusterroles
kubectl get clusterrolebinding
kubectl describe clusterrole cluster-admin
kubectl get serviceaccounts
```

```
kubectl auth can-i --list
kubectl auth can-i "*" "*"
kubectl auth can-i "*" "*" --as system:serviceaccount:my-service-account-name:default
kubectl auth can-i get pods -n my-namespace-name --as system:serviceaccount:my-service-account-name:default

kubectl exec user1 -- kubectl auth can-i delete pods
kubectl exec user2 -- kubectl get pods
```

```
kubectl create role my-reader-role-cli --verb=get,list,watch --resource=pods
kubectl get role my-reader-role-cli -o yaml
kubectl create role my-reader-role-cli --verb=get,list,watch --resource=pods --dry-run=client -o yaml > role.yaml

kubectl create rolebinding my-reader-role-cli-binding --role=my-reader-role-cli --user=seba

kubectl auth can-i get pods -n default --as seba
kubectl auth can-i delete pods -n default --as seba
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-reader-role
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["pods"]
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kube-explorer-system
  namespace: kube-system
subjects:
- kind: ServiceAccount
  name: kube-explorer
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-reader-role
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-reader-role-binding
  namespace: default
subjects:
- kind: User
  name: sebaczech@mail
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-cluste-role-create-approve-csr
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests"]
  verbs: ["create", "get", "list", "watch"]
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/approval"]
  verbs: ["update"]
- apiGroups:  ["certificates.k8s.io"]
  resources:  ["signers"]
  resourceNames:  ["kubernetes.io/kube-apiserver-client"]
  verbs: ["approve"]
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-log-reader
rules:
- apiGroups: [""]
  resources: ["pods ", "pods/log"]
  verbs: ["get"]
```

### Users

```
openssl genrsa -out seba.key 2048
openssl req -new -key seba.key -subj "/CN=seba/O=company" -out seba.csr
export REQUEST=$(cat seba.csr | base64 -w 0)

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
name: seba
spec:
groups:
- company
request: $REQUEST
signerName: kubernetes.io/kube-apiserver-client
usages:
- client auth
EOF

kubectl get csr
kubectl certificate approve seba
kubectl get csr seba -o jsonpath='{.status.certificate}' | base64 -d > seba.crt
kubectl config set-credentials seba --client-key=seba.key --client-certificate=seba.crt --embed-certs
kubectl config view
kubectl config set-context seba --user=seba --cluster=kind
kubectl config get-contexts
kubectl config use-context seba
```

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

### [Certificates](https://kubernetes.io/docs/tasks/administer-cluster/certificates/)

```
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config csr.conf
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out server.crt -days 10000 \
    -extensions v3_ext -extfile csr.conf -sha256
openssl req  -noout -text -in ./server.csr
openssl x509  -noout -text -in ./server.crt
```

### [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

```
kubectl label node kind-worker disktype=ssd
kubectl get nodes --show-labels
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{.spec.taints[*].key}{end}'

kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoSchedule-

kubectl taint nodes --all node-role.kubernetes.io/master-
```

```
spec:
 containers:
   - name: my-pod-name
     image: my-image-name
     imagePullPolicy: IfNotPresent
 tolerations:
     - key: "my-key-name"
       operator: "Equal"
       value: "my-key-value"
       effect: "NoSchedule"
 nodeSelector:
   disktype: ssd
```

```
kubectl run affinity --image nginx --dry-run=client -o yaml > affinity.yaml
```

```
affinity:
 nodeAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
     nodeSelectorTerms:
       - matchExpressions:
         - key: kubernetes.io/arch
           operator: In
           values:
             - amd64
         - key: kubernetes.io/os
           operator: In
           values:
             - linux
             - windows
   preferredDuringSchedulingIgnoredDuringExecution:
     - weight: 1
       preference:
       matchExpressions:
       - key: kubernetes.io/os
         operator: In
          values:
           - linux
```

```
affinity:
 podAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
     - labelSelector:
         matchExpressions:
           - key: app
             operator: In
             values:
               - my-app-name
       topologyKey: "kubernetes.io/hostname"
```

```
kubectl cordon node kind-worker # disable scheduling
kubectl drain kind-worker –-ignore-daemonsets
kubectl uncordon kind-worker # enable scheduling
```

```
apiVersion: v1
kind: Pod
metadata:
  name: annotation-default-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulerName: default-scheduler
  containers:
  - name: pod-with-default-annotation-container
    image: registry.k8s.io/pause:2.0

apiVersion: v1
kind: Pod
metadata:
  name: annotation-second-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulerName: my-scheduler
  containers:
  - name: pod-with-second-annotation-container
    image: registry.k8s.io/pause:2.0
```

### [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
 name: my-hpa
spec:
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: my-deployment
 minReplicas: 1
 maxReplicas: 5
 targetCPUUtilizationPercentage: 75
```

```
kubectl get hpa my-hpa
kubectl top pods -l app=my-app-name
```

### [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

[CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/):

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

```
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

```
kubectl get crd

kubectl get crontab
```

### Linux services

```
sudo systemctl list-unit-files --type service --all | grep kubelet -a6
sudo systemctl list-unit-files --type service --all | grep journald.service
```

### Docker

```
docker exec -it kind-control-plane bash

# apt update; apt install -y kubeadm=1.24.3-00
# kubeadm upgrade plan
# kubeadm upgrade apply v1.24.3

# apt update; apt install -y kubelet
# crictl ps
# systemctl status kubelet
# systemctl stop kubelet
# systemctl restart kubelet
# systemctl daemon-reload
```

### Upgrade control plane and node

#### Control plane

```
kubectl drain controlplane --ignore-daemonsets

apt update; apt install -y kubeadm=1.26.0-00
kubeadm upgrade plan
kubeadm upgrade apply v1.26.0

apt update; apt install -y kubelet=1.26.0-00 
systemctl restart kubelet
systemctl daemon-reload

kubectl uncordon  controlplane
```

#### Node

```
kubectl drain node01 --ignore-daemonsets

apt update; apt install -y kubeadm=1.26.0-00
kubeadm upgrade node

apt update; apt install -y kubelet=1.26.0-00 
systemctl restart kubelet
systemctl daemon-reload

kubectl uncordon node01
```

### kubeadm

```
kubeadm token create --print-join-command
kubeadm upgrade node phase kubelet-config
```

### [etcd](https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md) - [backup and recovery](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)

```
docker exec -it kind-control-plane bash
# apt update; apt install -y etcd-client
# export ETCDCTL_API=3
# etcdctl snapshot save snapshotdb --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key
# etcdctl snapshot status snapshotdb --write-out=table
# etcdctl snapshot restore snapshotdb --data-dir /var/lib/etcd-restore

# apt install update; apt install vim
# vim /etc/kubernetes/manifests/etcd.yaml ### change /var/lib/etcd to /var/lib/etcd-restore

# ls /etc/kubernetes
# ls /etc/kubernetes/pki
# cat /etc/kubernetes/kubelet.conf
```

### [Metrics](https://github.com/kubernetes-sigs/metrics-server)

```
kubectl top node
kubectl top pod
```

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