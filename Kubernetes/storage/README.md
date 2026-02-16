# Exercise - Storage

- Try adding a PersistentVolumeClaim to your GitOps configuration with the name linkding-data-pvc.

- Don't specify a storageclass, and use size 1Gi.

- Next, add the persistentVolumeClaim to your Deployment manifest.

**Bonus:**
- Describe the pod and exec into the container to verify that the storage has been mounted.

## Solution

**(base) endie@Endiesworld:~/Projects/Homelab/sochi/hml-project-1$ tree**
.
├── README.md
├── apps
│   ├── base
│   │   └── linkding
│   │       ├── deployment.yaml
│   │       ├── kustomization.yaml
│   │       ├── namespace.yaml
│   │       └── storage.yaml
│   └── staging
│       └── linkding
│           └── kustomization.yaml
└── clusters
    └── staging
        ├── apps.yaml
        └── flux-system
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml

8 directories, 10 files
**(base) endie@Endiesworld:~/Projects/Homelab/sochi/hml-project-1$ cat  apps/base/linkding/storage.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim  # <--- Changed from PersistentVolume
metadata:
  name: linkding-data-pvc      # typically ends in -pvc
spec:
  accessModes:
    - ReadWriteOnce          # "I need a drive that locks to one node"
  resources:                 # <--- VALID now! (PVs use 'capacity')
    requests:
      storage: 1Gi         # "I need at least 500Mi of space"
  # storageClassName: local-path # This is the default for Rancher Desktop/k3s.
  storageClassName: local-path # <--- CHANGED: Minikube's default class name Use 'standard' for most managed k8s services
```
**(base) endie@Endiesworld:~/Projects/Homelab/sochi/hml-project-1$ cat apps/base/linkding/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      containers:
        - name: linkding
          image: sissbruecker/linkding:1.45.0
          ports:
            - containerPort: 9090
          volumeMounts:
          - mountPath: "/app/data"
            name: linkding-storage
      volumes:
      - name: linkding-storage
        persistentVolumeClaim:
          claimName: linkding-data-pvc
```

**(base) endie@Endiesworld:~/Projects/Homelab/sochi/hml-project-1$ cat apps/base/linkding/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - namespace.yaml
  - storage.yaml
```
**(base) endie@Endiesworld:~$ kubectl get pods -n linkding**

NAME                        READY   STATUS    RESTARTS   AGE
linkding-77f996bcc6-tbgz9   1/1     Running   0          3m27s
(base) endie@Endiesworld:~$ kubectl describe pod/linkding-77f996bcc6-tbgz9
Error from server (NotFound): pods "linkding-77f996bcc6-tbgz9" not found
(base) endie@Endiesworld:~$ kubectl describe pod/linkding-77f996bcc6-tbgz9 -n linkding
Name:             linkding-77f996bcc6-tbgz9
Namespace:        linkding
Priority:         0
Service Account:  default
Node:             sochi/192.168.1.101
Start Time:       Sat, 31 Jan 2026 06:26:15 -0500
Labels:           app=linkding
                  pod-template-hash=77f996bcc6
Annotations:      <none>
Status:           Running
IP:               10.42.0.43
IPs:
  IP:           10.42.0.43
Controlled By:  ReplicaSet/linkding-77f996bcc6
Containers:
  linkding:
    Container ID:   containerd://1059fb20cc0f1ee07fbf3b688c9cf312a439f63831926d3a7e37710f1dcdc7fa
    Image:          sissbruecker/linkding:1.45.0
    Image ID:       docker.io/sissbruecker/linkding@sha256:61b2eb9eed8e5772a473fb7f1f8923e046cb8cbbeb50e88150afd5ff287d4060
    Port:           9090/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 31 Jan 2026 06:26:15 -0500
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /app/data from linkding-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-mbhqp (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  linkding-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  linkding-data-pvc
    ReadOnly:   false
  kube-api-access-mbhqp:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  4m10s  default-scheduler  Successfully assigned linkding/linkding-77f996bcc6-tbgz9 to sochi
  Normal  Pulled     4m10s  kubelet            Container image "sissbruecker/linkding:1.45.0" already present on machine
  Normal  Created    4m10s  kubelet            Created container: linkding
  Normal  Started    4m10s  kubelet            Started container linkding

**(base) endie@Endiesworld:~$ kubectl exec -n linkding linkding-77f996bcc6-tbgz9 -c linkding -i -t -- bash**
root@linkding-77f996bcc6-tbgz9:/etc/linkding# ls
LICENSE.txt           bootstrap.sh  manage.py          postcss.config.js  static                  supervisord.log  uwsgi.ini
background_tasks.log  data          package-lock.json  pyproject.toml     supervisord-all.conf    supervisord.pid  version.txt
bookmarks             libicu.so     package.json       rollup.config.mjs  supervisord-tasks.conf  uv.lock
root@linkding-77f996bcc6-tbgz9:/etc/linkding# cd /
root@linkding-77f996bcc6-tbgz9:/# ls
app  bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@linkding-77f996bcc6-tbgz9:/# ls  mnt/
root@linkding-77f996bcc6-tbgz9:/# ls  app/
data
root@linkding-77f996bcc6-tbgz9:/#
