# Installation for Jenkins in k8s local cluster

#### [Create a local cluster](https://docs.rancherdesktop.io/getting-started/installation/)

#### [Install kubectl](https://kubernetes.io/docs/tasks/tools/)

```bash
kubectl create namespace jenkins
```

#### Obtain name of node from new cluster
```bash
kubectl get nodes | awk '{print $1}'
> NAME
lima-rancher-desktop(example)
```

#### Update manifest volume.yaml, change {{lima-rancher-desktop}} to your node retrieved above:
```yaml
spec:
  storageClassName: local-storage
  claimRef:
    name: jenkins-pv-claim
    namespace: jenkins
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - {{lima-rancher-desktop}}
```

#### Create a service-account to be used by Jenkins in the jenkins namespace
```bash
kubectl apply -f serviceAccount.yaml
```

#### Create the volume to be used by Jenkins, adjust storage sizes accordingly as needed
```bash
kubectl apply -f volume.yaml
```

#### Deploy jenkins to jenkins namespace
```bash
kubectl apply -f deployment.yaml
```

#### Verify jenkins is deployed & pod are operational
```bash
kubectl get deployments -n jenkins
kubectl get pods -n jenkins
```

#### Access Jenkins locally via kubectl port-forward, first obtain pod name:
1. Obtain main pod name
2. Update {{pod-name}} with actual pod name retrieved via first command
3. Update {{local-port}} with actual local target port available
```bash
kubectl get pods -n jenkins | awk '{print $1}'
kubectl port forward {{pod-name}} -n jenkins {{local-port}}:8080
```

#### Naviage to browser and access Jenkins via
http://localhost:{{local-port}}

#### Logging in first time you will be prompted to:
![jenkis-first-login](https://github.com/tlgevers/local-jenkins-k8s/blob/main/images/jenkins-first-login.png?raw=true

Obtain the password via:
(Use the pod name as found above)
```bash
kubectl logs {{pod-name}} -n jenkins

*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required.
An admin user has been created and a password generated.
Please use the following password to proceed to installation:

94b73ef6578c4b4692a157f768b2cfef

This may also be found at:
/var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

