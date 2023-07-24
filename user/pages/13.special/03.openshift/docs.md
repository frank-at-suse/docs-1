---
title: OpenShift (Allinone)
taxonomy:
    category: docs
---


### Deploy Using RedHat OpenShift
NeuVector is compatible with standard ovs SDN plug-ins as well as others such as flannel, weave, or calico. The samples below assume a standard ovs plug-in is used. This also assumes a local docker registry will be used. The default yaml deployment deploys one allinone on a labeled node and enforcers on all others, including the master. 

First, pull the appropriate NeuVector containers from the NeuVector registry into your local registry.

For Docker Hub
```
docker pull docker.io/neuvector/allinone:5.2.0
docker pull docker.io/neuvector/enforcer:5.2.0
docker pull docker.io/neuvector/scanner
docker pull docker.io/neuvector/updater
```

Next, tag/push the containers, set the route and allow privileged NeuVector containers using the instructions below. By default, OpenShift does not allow privileged containers. Also, by default OpenShift does not schedule pods on the Master node. See the instructions at the end to enable/disable this.


NOTE: Please see the Enterprise Integration section for details on integration with OpenShift Role Based Access Controls (RBACs). 

1. Login as a normal user
```
oc login -u <user_name>
```

2. Create a new project
Note: If the --node-selector argument is used when creating a project this will restrict pod placement such as for the NeuVector enforcer to specific nodes.
```
oc new-project neuvector
```

3. Push NeuVector images to OpenShift docker registry. IMPORTANT! The docker login below is for the docker-registry (not to be confused with the oc login above), and requires the user/authentication token in the command.

Note: For OpenShift 4.6+, change docker-registry.default.svc below to image-registry.openshift-image-registry.svc in the commands below.
```
docker login -u <user_name> -p `oc whoami -t` docker-registry.default.svc:5000
docker tag docker.io/neuvector/allinone:<version> docker-registry.default.svc:5000/neuvector/allinone
docker tag docker.io/neuvector/enforcer:<version> docker-registry.default.svc:5000/neuvector/enforcer
docker tag docker.io/neuvector/scanner docker-registry.default.svc:5000/neuvector/scanner
docker tag docker.io/neuvector/updater docker-registry.default.svc:5000/neuvector/updater
docker push docker-registry.default.svc:5000/neuvector/allinone
docker push docker-registry.default.svc:5000/neuvector/enforcer
docker push docker-registry.default.svc:5000/neuvector/scanner
docker push docker-registry.default.svc:5000/neuvector/updater
docker logout docker-registry.default.svc:5000
```

4. Login as system:admin account
```
oc login -u system:admin
```

5. Create Service Accounts and Grant Service Account Access to the Privileged SCC
```
oc create sa controller -n neuvector
oc create sa enforcer -n neuvector
oc create sa basic -n neuvector
oc create sa updater -n neuvector
oc -n neuvector adm policy add-scc-to-user privileged -z controller -z enforcer
```
The following info will be added in the Privileged SCC
users:
```
- system:serviceaccount:neuvector:controller
- system:serviceaccount:neuvector:enforcer
```
In OpenShift 4.6+ use the following to check:
```
# oc get rolebinding system:openshift:scc:privileged -n neuvector -o wide
```
```
NAME                              ROLE                                          AGE     USERS   GROUPS   SERVICEACCOUNTS
system:openshift:scc:privileged   ClusterRole/system:openshift:scc:privileged   9m22s                    neuvector/controller, neuvector/enforcer
```

Create the custom resources (CRD) for NeuVector security rules. For OpenShift 4.6+ (Kubernetes 1.19+):
```
oc apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.2.0/crd-k8s-1.19.yaml
oc apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.2.0/waf-crd-k8s-1.19.yaml
oc apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.2.0/dlp-crd-k8s-1.19.yaml
oc apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.2.0/admission-crd-k8s-1.19.yaml
```

6. Add read permission to access the kubernetes API and OpenShift RBACs. IMPORTANT: The standard NeuVector 5.2+ deployment uses least-privileged service accounts instead of the default. See below if upgrading to 5.2+ from a version prior to 5.2.
```
oc create clusterrole neuvector-binding-app --verb=get,list,watch,update --resource=nodes,pods,services,namespaces
oc create clusterrole neuvector-binding-rbac --verb=get,list,watch --resource=rolebindings.rbac.authorization.k8s.io,roles.rbac.authorization.k8s.io,clusterrolebindings.rbac.authorization.k8s.io,clusterroles.rbac.authorization.k8s.io,imagestreams.image.openshift.io
oc adm policy add-cluster-role-to-user neuvector-binding-app system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-rbac system:serviceaccount:neuvector:controller
oc create clusterrole neuvector-binding-admission --verb=get,list,watch,create,update,delete --resource=validatingwebhookconfigurations,mutatingwebhookconfigurations
oc adm policy add-cluster-role-to-user neuvector-binding-admission system:serviceaccount:neuvector:controller
oc create clusterrole neuvector-binding-customresourcedefinition --verb=watch,create,get,update --resource=customresourcedefinitions
oc adm policy add-cluster-role-to-user neuvector-binding-customresourcedefinition system:serviceaccount:neuvector:controller
oc create clusterrole neuvector-binding-nvsecurityrules --verb=list,delete --resource=nvsecurityrules,nvclustersecurityrules
oc adm policy add-cluster-role-to-user neuvector-binding-nvsecurityrules system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user view system:serviceaccount:neuvector:controller --rolebinding-name=neuvector-binding-view
oc create clusterrole neuvector-binding-nvwafsecurityrules --verb=list,delete --resource=nvwafsecurityrules
oc adm policy add-cluster-role-to-user neuvector-binding-nvwafsecurityrules system:serviceaccount:neuvector:controller
oc create clusterrole neuvector-binding-nvadmissioncontrolsecurityrules --verb=list,delete --resource=nvadmissioncontrolsecurityrules
oc adm policy add-cluster-role-to-user neuvector-binding-nvadmissioncontrolsecurityrules system:serviceaccount:neuvector:controller
oc create clusterrole neuvector-binding-nvdlpsecurityrules --verb=list,delete --resource=nvdlpsecurityrules
oc adm policy add-cluster-role-to-user neuvector-binding-nvdlpsecurityrules system:serviceaccount:neuvector:controller
oc create role neuvector-binding-scanner --verb=get,patch,update,watch --resource=deployments -n neuvector
oc adm policy add-role-to-user neuvector-binding-scanner system:serviceaccount:neuvector:updater system:serviceaccount:neuvector:controller -n neuvector --role-namespace neuvector
oc create clusterrole neuvector-binding-csp-usages --verb=get,create,update,delete --resource=cspadapterusagerecords
oc adm policy add-cluster-role-to-user neuvector-binding-csp-usages system:serviceaccount:neuvector:controller
oc create clusterrole neuvector-binding-co --verb=get,list --resource=clusteroperators
oc adm policy add-cluster-role-to-user neuvector-binding-co system:serviceaccount:neuvector:enforcer system:serviceaccount:neuvector:controller
```
NOTE: If upgrading from a previous NeuVector deployment (prior to 5.2), you will need to delete the old bindings, then create new ones:
```
oc delete clusterrolebinding neuvector-binding-app neuvector-binding-rbac neuvector-binding-admission neuvector-binding-customresourcedefinition neuvector-binding-nvsecurityrules neuvector-binding-view neuvector-binding-nvwafsecurityrules neuvector-binding-nvadmissioncontrolsecurityrules neuvector-binding-nvdlpsecurityrules neuvector-binding-co
oc delete rolebinding neuvector-admin -n neuvector
oc adm policy add-cluster-role-to-user neuvector-binding-app system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-rbac system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-admission system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-customresourcedefinition system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-nvsecurityrules system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user view system:serviceaccount:neuvector:controller --rolebinding-name=neuvector-binding-view
oc adm policy add-cluster-role-to-user neuvector-binding-nvwafsecurityrules system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-nvadmissioncontrolsecurityrules system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-nvdlpsecurityrules system:serviceaccount:neuvector:controller
oc create role neuvector-binding-scanner --verb=get,patch,update,watch --resource=deployments -n neuvector
oc adm policy add-role-to-user neuvector-binding-scanner system:serviceaccount:neuvector:updater system:serviceaccount:neuvector:controller -n neuvector --role-namespace neuvector
oc create clusterrole neuvector-binding-csp-usages --verb=get,create,update,delete --resource=cspadapterusagerecords
oc adm policy add-cluster-role-to-user neuvector-binding-csp-usages system:serviceaccount:neuvector:controller
oc adm policy add-cluster-role-to-user neuvector-binding-co system:serviceaccount:neuvector:enforcer system:serviceaccount:neuvector:controller
```

7. Run the following command to check if the neuvector/controller, neuvector/enforcer and neuvector/updater service accounts are added successfully.
```
oc get ClusterRoleBinding neuvector-binding-app neuvector-binding-rbac neuvector-binding-admission neuvector-binding-customresourcedefinition neuvector-binding-nvsecurityrules neuvector-binding-view neuvector-binding-nvwafsecurityrules neuvector-binding-nvadmissioncontrolsecurityrules neuvector-binding-nvdlpsecurityrules neuvector-binding-csp-usages neuvector-binding-co -o wide
```
Sample output:
```
NAME                                                ROLE                                                            AGE   USERS   GROUPS   SERVICEACCOUNTS
neuvector-binding-app                               ClusterRole/neuvector-binding-app                               56d                    neuvector/controller
neuvector-binding-rbac                              ClusterRole/neuvector-binding-rbac                              34d                    neuvector/controller
neuvector-binding-admission                         ClusterRole/neuvector-binding-admission                         72d                    neuvector/controller
neuvector-binding-customresourcedefinition          ClusterRole/neuvector-binding-customresourcedefinition          72d                    neuvector/controller
neuvector-binding-nvsecurityrules                   ClusterRole/neuvector-binding-nvsecurityrules                   72d                    neuvector/controller
neuvector-binding-view                              ClusterRole/view                                                72d                    neuvector/controller
neuvector-binding-nvwafsecurityrules                ClusterRole/neuvector-binding-nvwafsecurityrules                72d                    neuvector/controller
neuvector-binding-nvadmissioncontrolsecurityrules   ClusterRole/neuvector-binding-nvadmissioncontrolsecurityrules   72d                    neuvector/controller
neuvector-binding-nvdlpsecurityrules                ClusterRole/neuvector-binding-nvdlpsecurityrules                72d                    neuvector/controller
neuvector-binding-csp-usages                        ClusterRole/neuvector-binding-csp-usages                        24d                    neuvector/controller
neuvector-binding-co                                ClusterRole/neuvector-binding-co                                72d                    neuvector/enforcer, neuvector/controller
```

And this command:
```
oc get RoleBinding neuvector-binding-scanner -n neuvector -o wide
```
Sample output:
```
NAME                        ROLE                             AGE   USERS   GROUPS   SERVICEACCOUNTS
neuvector-binding-scanner   Role/neuvector-binding-scanner   70d                    neuvector/updater, neuvector/controller
```
8. Add a nvallinone label on one of the master or worker nodes where the allinone will be deployed
```
oc label nodes <nodename> nvallinone=true
```

9. Create the neuvector services and pods based on the sample yaml files below
```
oc create -f <compose file>
```

If you have created your own namespace instead of using “neuvector”:
1. Replace all instances of “namespace: neuvector” with your namespace.
2. Search for all instances of ”neuvector-svc-allinone.neuvector” in the files below. Then replace the “neuvector” (after the .) with the namespace you use.

Note1: If you are using the Allinone container for testing NeuVector, deploy only one Allinone for your cluster. Multiple Manager instances are not supported on Kubernetes. To test high availability for the Controller refer to the Deploying in Production section.


<strong>Sample Config File for OpenShift 4.6+</strong>
Note: For 4.6+, see the section Deploying NeuVector / OpenShift for yaml file changes required for the CRI-O run-time and for changing the default registry.
```
apiVersion: v1
kind: Service
metadata:
  name: neuvector-svc-crd-webhook
  namespace: neuvector
spec:
  ports:
  - port: 443
    targetPort: 30443
    protocol: TCP
    name: crd-webhook
  type: ClusterIP
  selector:
    app: neuvector-allinone-pod

---

apiVersion: v1
kind: Service
metadata:
  name: neuvector-svc-admission-webhook
  namespace: neuvector
spec:
  ports:
  - port: 443
    targetPort: 20443
    protocol: TCP
    name: admission-webhook
  type: ClusterIP
  selector:
    app: neuvector-allinone-pod

---

apiVersion: v1
kind: Service
metadata:
  name: neuvector-service-webui
  namespace: neuvector
spec:
  ports:
    - port: 8443
      name: manager
      protocol: TCP
  type: ClusterIP
  selector:
    app: neuvector-allinone-pod

---

apiVersion: v1
kind: Service
metadata:
  name: neuvector-svc-allinone
  namespace: neuvector
spec:
  ports:
  - port: 18300
    protocol: "TCP"
    name: "cluster-tcp-18300"
  - port: 18301
    protocol: "TCP"
    name: "cluster-tcp-18301"
  - port: 18301
    protocol: "UDP"
    name: "cluster-udp-18301"
  clusterIP: None
  selector:
    app: neuvector-allinone-pod

---

apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: neuvector-route-webui
  namespace: neuvector
spec:
  to:
    kind: Service
    name: neuvector-service-webui
  port:
    targetPort: manager
  tls:
    termination: passthrough

---

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: neuvector-allinone-pod
  namespace: neuvector
spec:
  selector:
    matchLabels:
      app: neuvector-allinone-pod
  minReadySeconds: 60
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: neuvector-allinone-pod
    spec:
      hostPID: true
      serviceAccountName: controller
      serviceAccount: controller
      containers:
        - name: neuvector-allinone-pod
          image: image-registry.openshift-image-registry.svc:5000/neuvector/allinone:<version>
          securityContext:
            privileged: true
          readinessProbe:
            exec:
              command:
              - cat
              - /tmp/ready
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-allinone.neuvector
            - name: CLUSTER_ADVERTISED_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CLUSTER_BIND_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - mountPath: /lib/modules
              name: modules-vol
              readOnly: true
            - mountPath: /var/neuvector
              name: nv-share
              readOnly: false
            - mountPath: /var/run/crio/crio.sock
              name: runtime-sock
              readOnly: true
            - mountPath: /host/proc
              name: proc-vol
              readOnly: true
            - mountPath: /host/cgroup
              name: cgroup-vol
              readOnly: true
            - mountPath: /etc/config
              name: config-volume
              readOnly: true
      terminationGracePeriodSeconds: 1200
      nodeSelector:
        nvallinone: "true"
      restartPolicy: Always
      volumes:
        - name: modules-vol
          hostPath:
            path: /lib/modules
        - name: nv-share
          hostPath:
            path: /var/neuvector
        - name: runtime-sock
          hostPath:
            path: /var/run/crio/crio.sock
        - name: proc-vol
          hostPath:
            path: /proc
        - name: cgroup-vol
          hostPath:
            path: /sys/fs/cgroup
        - name: config-volume
          projected:
            sources:
              - configMap:
                  name: neuvector-init
                  optional: true
              - secret:
                  name: neuvector-init
                  optional: true

---

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: neuvector-enforcer-pod
  namespace: neuvector
spec:
  selector:
    matchLabels:
      app: neuvector-enforcer-pod
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: neuvector-enforcer-pod
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: "nvallinone"
                  operator: NotIn
                  values: ["true"]
      hostPID: true
      serviceAccountName: enforcer
      serviceAccount: enforcer
      containers:
        - name: neuvector-enforcer-pod
          image: image-registry.openshift-image-registry.svc:5000/neuvector/enforcer:<version>
          securityContext:
            privileged: true
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-allinone.neuvector
            - name: CLUSTER_ADVERTISED_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CLUSTER_BIND_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - mountPath: /lib/modules
              name: modules-vol
              readOnly: true
            - mountPath: /var/run/crio/crio.sock
              name: runtime-sock
              readOnly: true
            - mountPath: /host/proc
              name: proc-vol
              readOnly: true
            - mountPath: /host/cgroup
              name: cgroup-vol
              readOnly: true
      terminationGracePeriodSeconds: 1200
      restartPolicy: Always
      volumes:
        - name: modules-vol
          hostPath:
            path: /lib/modules
        - name: runtime-sock
          hostPath:
            path: /var/run/crio/crio.sock
        - name: proc-vol
          hostPath:
            path: /proc
        - name: cgroup-vol
          hostPath:
            path: /sys/fs/cgroup

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: neuvector-scanner-pod
  namespace: neuvector
spec:
  selector:
    matchLabels:
      app: neuvector-scanner-pod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  replicas: 2
  template:
    metadata:
      labels:
        app: neuvector-scanner-pod
    spec:
      serviceAccountName: basic
      serviceAccount: basic
      containers:
        - name: neuvector-scanner-pod
          image: image-registry.openshift-image-registry.svc:5000/neuvector/scanner:latest
          imagePullPolicy: Always
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-controller.neuvector
      restartPolicy: Always

---

apiVersion: batch/v1
kind: CronJob
metadata:
  name: neuvector-updater-pod
  namespace: neuvector
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: neuvector-updater-pod
        spec:
          serviceAccountName: updater
          serviceAccount: updater
          containers:
          - name: neuvector-updater-pod
            image: image-registry.openshift-image-registry.svc:5000/neuvector/updater:latest
            imagePullPolicy: Always
            command:
            - /bin/sh
            - -c
            - TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`; /usr/bin/curl -kv -X PATCH -H "Authorization:Bearer $TOKEN" -H "Content-Type:application/strategic-merge-patch+json" -d '{"spec":{"template":{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt":"'`date +%Y-%m-%dT%H:%M:%S%z`'"}}}}}' 'https://kubernetes.default/apis/apps/v1/namespaces/neuvector/deployments/neuvector-scanner-pod'
          restartPolicy: Never
```

**Master Node Taints and Tolerations**
All taint info must match to schedule Enforcers on nodes. To check the taint info on a node (e.g. Master):
```
$ oc get node taintnodename -o yaml
```
Sample output:
```
spec:
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
  # there may be an extra info for taint as below
  - effect: NoSchedule
    key: mykey
    value: myvalue
```

If there is additional taints as above, add these to the sample yaml tolerations section:
```
spec:
  template:
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
        # if there is an extra info for taints as above, please add it here. This is required to match all the taint info defined on the taint node. Otherwise, the Enforcer won't deploy on the taint node
        - effect: NoSchedule
          key: mykey
          value: myvalue
```


### Enable/Disable Scheduling on the Master Node

The following commands can be used to enable/disable the scheduling on the master node.

```
oc adm manage-node nodename --schedulable
```

```
oc adm manage-node nodename --schedulable=false
```
