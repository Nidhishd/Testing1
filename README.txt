Environment
+++++++++++

Installed and configured Minikube in Oracle VirtualBox Manager.
Downloaded Kubectl in my local Windows10 Machine and set the environment variable.
Deployed Wordpress-MariaDB Application using Helm Charts within the Minikube created Kubernetes environment..




Deployment- Wordpress with Helm


## Downloaded and initialized helm.

root@kube-master:#  helm init

## Given tiller admin access to the entire Kubernetes cluster.

kubectl create serviceaccount --namespace kube-system tiller ()

kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller  


## Created the tiller pod, same can be deployed in Helm init command(Helm init --serviceaccount tiller --tiller-namespace tiller-world).

 kubectl -n kube-system patch deployment tiller-deploy -p "{\"spec\":{\"template\":{\"spec\":{\"serviceAccount\":\"tiller\"}}}}"


Once the titler pod is up and running, deploy wordpress using bitnami docker images(Will share the Dockerfile in git repo). For this we need to go and create PersistentVolume and PersistentVolumeClaim.

Define the PersistentVolume for mariadb-pv where the mariadb data to be stored. The hostPath tells the mysql directory is in /bitnami/mariadb location.

# cat mariadb-hostpath.yaml 

apiVersion: v1
kind: PersistentVolume
metadata:
  name: mariadb-pv
spec:
  capacity: 
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /bitnami/mariadb

# kubectl create -f  mariadb-hostpath.yaml 


# cat wordpress-hostpath.yaml 

apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity: 
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data

# kubectl create -f  wordpress-hostpath.yaml


# cat wordpress-mariadb-pvc.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: data-wordpress-mariadb-0
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi


# kubectl create -f wordpress-mariadb-pvc.yaml


# cat wordpress-pvc.yaml 

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: wordpress-wordpress
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi


# kubectl create -f wordpress-pvc.yaml


# kubectl get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                               STORAGECLASS   REASON   AGE
mariadb-pv                                 1Gi        RWO,ROX        Retain           Bound    default/data-wordpress-mariadb-0                            3h36m
wordpress-pv                               1Gi        RWO,ROX        Retain           Bound    default/wordpress-wordpress                                 3h30m

#kubectl get pvc

NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-wordpress-mariadb-0    Bound    mariadb-pv                                 1Gi        RWO,ROX                       3h30m
wordpress-wordpress         Bound    wordpress-pv                               1Gi        RWO,ROX                       3h28m


##Created Wordpress and MariaDB services, pods and configmap and wordpress secrets.


#helm install --name wordpress  --set wordpressUsername=admin,wordpressPassword=adminpassword,mariadb.mariadbRootPassword=secretpassword,persistence.existingClaim=wordpress-wordpress,allowEmptyPassword=false stable/wordpress  --set securityContext.runAsUser=0 --set securityContext.fsGroup=0



#kubectl get pods

NAME                         READY   STATUS    RESTARTS   AGE
wordpress-7676f89ccb-78v4l   1/1     Running   0          74m
wordpress-mariadb-0          1/1     Running   0          74m


## Converted the Wordpress password using base64 decoder => adminpassword.


#kubectl get secret  wordpress  -o yaml


#kubectl get svc -o wide

NAME                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
hello-minikube      NodePort       10.104.55.175   <none>        8080:30432/TCP               2d4h    app=hello-minikube
kubernetes          ClusterIP      10.96.0.1       <none>        443/TCP                      3d22h   <none>
wordpress           LoadBalancer   10.109.8.165    <pending>     80:30392/TCP,443:32200/TCP   77m     app.kubernetes.io/instance=wordpress,app.kubernetes.io/name=wordpress
wordpress-mariadb   ClusterIP      10.111.14.114   <none>        3306/TCP                     77m     app=mariadb,component=master,release=wordpress


#kubectl get secrets

NAME                  TYPE                                  DATA   AGE
default-token-lx7s7   kubernetes.io/service-account-token   3      3d22h
wordpress             Opaque                                1      79m
wordpress-mariadb     Opaque                                2      79m

#minikube service  wordpress --url

http://192.168.99.100:30392
http://192.168.99.100:32200



Deployment yaml.


Wordpress


# kubectl edit pod wordpress-7676f89ccb-78v4l

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2020-11-15T08:22:41Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: wordpress
    app.kubernetes.io/managed-by: Tiller
    app.kubernetes.io/name: wordpress
    helm.sh/chart: wordpress-9.0.3
  name: wordpress
  namespace: default
  resourceVersion: "150331"
  selfLink: /apis/apps/v1/namespaces/default/deployments/wordpress
  uid: 7b793e27-a4bd-48b9-8fbd-79b5a367cea3
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/instance: wordpress
      app.kubernetes.io/name: wordpress
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/instance: wordpress
        app.kubernetes.io/managed-by: Tiller
        app.kubernetes.io/name: wordpress
        helm.sh/chart: wordpress-9.0.3
    spec:
      containers:
      - env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "no"
        - name: MARIADB_HOST
          value: wordpress-mariadb
        - name: MARIADB_PORT_NUMBER
          value: "3306"
        - name: WORDPRESS_DATABASE_NAME
          value: bitnami_wordpress
        - name: WORDPRESS_DATABASE_USER
          value: bn_wordpress
        - name: WORDPRESS_DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              key: mariadb-password
              name: wordpress-mariadb
        - name: WORDPRESS_USERNAME
          value: admin
        - name: WORDPRESS_PASSWORD
          valueFrom:
            secretKeyRef:
              key: wordpress-password
              name: wordpress
        - name: WORDPRESS_EMAIL
          value: user@example.com
        - name: WORDPRESS_FIRST_NAME
          value: FirstName
        - name: WORDPRESS_LAST_NAME
          value: LastName
        - name: WORDPRESS_HTACCESS_OVERRIDE_NONE
          value: "no"
        - name: WORDPRESS_BLOG_NAME
          value: User's Blog!
        - name: WORDPRESS_SKIP_INSTALL
          value: "no"
        - name: WORDPRESS_TABLE_PREFIX
          value: wp_
        - name: WORDPRESS_SCHEME
          value: http
        image: docker.io/bitnami/wordpress:5.3.2-debian-10-r32
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 6
          httpGet:
            path: /wp-login.php
            port: http
            scheme: HTTP
          initialDelaySeconds: 120
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: wordpress
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 8443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 6
          httpGet:
            path: /wp-login.php
            port: http
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        resources:
          requests:
            cpu: 300m
            memory: 512Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /bitnami/wordpress
          name: wordpress-data
          subPath: wordpress
      dnsPolicy: ClusterFirst
      hostAliases:
      - hostnames:
        - status.localhost
        ip: 127.0.0.1
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 0
        runAsUser: 0
      terminationGracePeriodSeconds: 30
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-wordpress
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2020-11-15T08:23:43Z"
    lastUpdateTime: "2020-11-15T08:23:43Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2020-11-15T08:22:41Z"
    lastUpdateTime: "2020-11-15T08:23:43Z"
    message: ReplicaSet "wordpress-7676f89ccb" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1


MariaDB

#kubectl edit pod wordpress-mariadb-0

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-11-15T08:22:41Z"
  generateName: wordpress-mariadb-
  labels:
    app: mariadb
    chart: mariadb-7.3.12
    component: master
    controller-revision-hash: wordpress-mariadb-fc99f458f
    release: wordpress
    statefulset.kubernetes.io/pod-name: wordpress-mariadb-0
  name: wordpress-mariadb-0
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: StatefulSet
    name: wordpress-mariadb
    uid: 2f9cd825-4f9f-4d47-a889-b349ce8e7836
  resourceVersion: "150270"
  selfLink: /api/v1/namespaces/default/pods/wordpress-mariadb-0
  uid: 0e2122ca-915d-46aa-9c6a-bc92a8c8ea8a
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - podAffinityTerm:
          labelSelector:
            matchLabels:
              app: mariadb
              release: wordpress
          topologyKey: kubernetes.io/hostname
        weight: 1
  containers:
  - env:
    - name: MARIADB_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          key: mariadb-root-password
          name: wordpress-mariadb
    - name: MARIADB_USER
      value: bn_wordpress
    - name: MARIADB_PASSWORD
      valueFrom:
        secretKeyRef:
          key: mariadb-password
          name: wordpress-mariadb
    - name: MARIADB_DATABASE
      value: bitnami_wordpress
    image: docker.io/bitnami/mariadb:10.3.22-debian-10-r27
    imagePullPolicy: IfNotPresent
    livenessProbe:
      exec:
        command:
        - sh
        - -c
        - |
          password_aux="${MARIADB_ROOT_PASSWORD:-}"
          if [ -f "${MARIADB_ROOT_PASSWORD_FILE:-}" ]; then
              password_aux=$(cat $MARIADB_ROOT_PASSWORD_FILE)
          fi
          mysqladmin status -uroot -p$password_aux
      failureThreshold: 3
      initialDelaySeconds: 120
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 1
    name: mariadb
    ports:
    - containerPort: 3306
      name: mysql
      protocol: TCP
    readinessProbe:
      exec:
        command:
        - sh
        - -c
        - |
          password_aux="${MARIADB_ROOT_PASSWORD:-}"
          if [ -f "${MARIADB_ROOT_PASSWORD_FILE:-}" ]; then
              password_aux=$(cat $MARIADB_ROOT_PASSWORD_FILE)
          fi
          mysqladmin status -uroot -p$password_aux
      failureThreshold: 3
      initialDelaySeconds: 30
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 1
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /bitnami/mariadb
      name: data
    - mountPath: /opt/bitnami/mariadb/conf/my.cnf
      name: config
      subPath: my.cnf
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-lx7s7
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  hostname: wordpress-mariadb-0
  nodeName: minikube
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext:
    fsGroup: 1001
    runAsUser: 1001
  serviceAccount: default
  serviceAccountName: default
  subdomain: wordpress-mariadb
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-wordpress-mariadb-0
  - configMap:
      defaultMode: 420
      name: wordpress-mariadb
    name: config
  - name: default-token-lx7s7
    secret:
      defaultMode: 420
      secretName: default-token-lx7s7
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-11-15T08:22:41Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-11-15T08:23:20Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-11-15T08:23:20Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-11-15T08:22:41Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://a05a6c8d222b59f3361afb9ba81ddffbc8e59ccc920b10ff6b51d1236cc6385b
    image: bitnami/mariadb:10.3.22-debian-10-r27
    imageID: docker-pullable://bitnami/mariadb@sha256:6f6b95d95e0effb86e0d699bc01e5e53530a0ab5ce8fa1d42bd2280479f57b35
    lastState: {}
    name: mariadb
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-11-15T08:22:43Z"
  hostIP: 192.168.99.100
  phase: Running
  podIP: 172.17.0.7
  podIPs:
  - ip: 172.17.0.7
  qosClass: BestEffort
  startTime: "2020-11-15T08:22:41Z"
