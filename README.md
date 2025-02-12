# Installation for Jenkins in k8s local cluster
<sub>This is an example for running Jenkins locally within k8s & creating
and agent with the Kubernetes plugin within that same cluster, each agent
will be a pod. Different rules apply for connecting from Docker to
k8s.</sub>

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
![jenkis-first-login](https://github.com/tlgevers/local-jenkins-k8s/blob/main/images/jenkins-first-login.png?raw=true)

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
#### Plugins
Select: Install suggested plugins

Enter required fields: Username, Password, Confirm Password, Full name, E-mail address

Leave default Jenkins URL: as http://localhost:{{local-port}}/ (same as kubectl port-forward address specified earlier via browser)

Save & Finish
Select Start Using Jenkins

#### More Plugins:
install another:
https://plugins.jenkins.io/kubernetes/
1. Navigate to Manage Jenkins
2. Plugins
3. Available Plugins
4. Search for Kubernetes
5. Select the first one, or one named Kubernetes ONLY(Installs several plugins)
6. Toggle the checkbox & click Install

#### Add a new agent
1. Select Manage Jenkins
2. Select Clouds
3. Select New cloud
4. enter Cloud name as k8s
5. Toggle Kubernetes
6. Click Create

#### Create a secret/token for access to be used by Jenkins connected to **jenkins-admin** service account
```bash
kubectl apply -f token.yaml
kubectl get secrets --namespace jenkins
kubectl describe secret jenkins-secret -n jenkins
```
![jenkis-secret](https://github.com/tlgevers/local-jenkins-k8s/blob/main/images/k8s-secret-token.png?raw=true)
NB: COPY THE **TOKEN** FOR NEXT STEP

#### Obtain internal IP of Jenkins pod, save the **IP**
```bash
kubectl describe pod {{pod-name}} -n jenkins
```

#### Complete the k8s(or selected name) Configuration
1. Toggle Disable https certificate key
2. Set Kubernetes Namespace to jenkins
3. Create Credentials by clicking Add, then Jenkins
4. Select Secret Text
5. Leave Domain
6. Set secret to **TOKEN** from previous step
7. Set ID to jenkins-secret
8. Click Add
9. Set Jenkins URL to Internal IP Address to the Master Jenkins pod **IP** obtained in previous step(communication between pods is permitted within the same namespace)
**http://internal-ip:8080**
NB! Make sure to set the port as this is the private http port
11. Click Save

#### Create a test Job:
1. Click + New Item
2. Enter an item name
3. Select Pipeline
4. Set the pipeline script as follows:

```
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        metadata:
          namespace: jenkins
        spec:
          serviceAccount: jenkins-admin
          serviceAccountName: jenkins-admin
          containers:
          - name: maven
            image: maven:alpine
            command:
            - cat
            tty: true
        '''
    }
  }
  stages {
    stage('Run maven') {
      steps {
        container('maven') {
          sh 'mvn -version'
        }
      }
    }
  }
}
```

5. Click Save
6. Build Now
7. Watch Build History
8. Select the #number of the build fo results
9. Green is successful!

