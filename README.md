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

kubectl create deployment my-deployment-name --image=docker-image-name

kubectl exec deploy/my-deployment-name-1 -- sh -c 'wget -O - http://localhost > /dev/null'
kubectl logs --tail=1 -l app=my-deployment-name-1

kubectl delete deploy --all
kubectl get all
```

### [Service](https://kubernetes.io/docs/concepts/services-networking/)

```
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

// LKLM - 7
// KA - 1
// ACKAE - 1

### Scaling

[ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/):

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

[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/):

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