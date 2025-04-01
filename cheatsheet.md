**OpenShift Command Cheat Sheet**

### General Commands



**2. Log in to the OpenShift cluster as admin**
```
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
```
*Logs in to the OpenShift cluster using admin credentials.*

**3. Get the current OpenShift cluster version**
```
oc version
```
*Displays the OpenShift client and server version.*

### User & Group Management

**4. Create a new group**
```
oc adm groups new <group-name>
```
*Creates a new user group in OpenShift.*

**5. Add a user to a group**
```
oc adm groups add-users <group-name> <user-name>
```
*Adds a specified user to an existing group.*

**6. View all groups**
```
oc get groups
```
*Lists all existing groups and their users.*

**7. Grant a cluster role to a group**
```
oc adm policy add-cluster-role-to-group <role> <group>
```
*Assigns a cluster role to a specific group.*

**8. Assign a role to a user group**
```
oc adm policy add-role-to-group <role> <group>
```
*Grants a role to a specified user group within a project.*

### Project & Namespace Management

**9. Create a new OpenShift project**
```
oc new-project <project-name>
```
*Creates a new project (namespace) in OpenShift.*

**10. Label a namespace**
```
oc label ns <namespace> <label-key>=<label-value>
```
*Adds a label to a namespace.*

**11. View a project's configuration**
```
oc get project <project-name> -o yaml
```
*Displays detailed configuration for a project in YAML format.*

**12. Apply changes to the cluster project configuration**
```
oc edit projects.config.openshift.io cluster
```
*Modifies cluster-wide project settings.*

### Resource Management

**13. Create resources from a YAML file**
```
oc create -f <filename>.yaml
```
*Creates resources defined in a YAML file.*

**14. Set resource quotas**
```
oc create -f quota.yaml
```
*Defines resource limits for a project using a YAML file.*

**15. Define limit ranges**
```
oc create -f limitrange.yaml
```
*Sets default resource requests and limits for containers in a namespace.*

**16. Create a network policy**
```
oc create -f networkpolicy.yaml
```
*Defines network restrictions between namespaces.*

### Pod & Deployment Management

**17. View pods and their details**
```
oc get pod -o wide
```
*Lists all pods along with additional details such as IP addresses.*

**18. Debug a pod and execute a command**
```
oc debug --to-namespace=<namespace> -- <command>
```
*Runs a debug session within a namespace and executes a command.*

**19. Watch OpenShift API server pods**
```
watch oc get pod -n openshift-apiserver
```
*Continuously monitors API server pods.*

**20. Edit an existing cluster role binding**
```
oc edit clusterrolebinding <binding-name>
```
*Modifies an existing role binding configuration.*

### Advanced Configuration

**21. Generate a project template**
```
oc adm create-bootstrap-project-template -o yaml > project-template.yaml
```
*Creates a project template YAML file.*




**22. Change to the `~/DO280/labs/compreview-apps` directory and log in to the cluster as the admin user.**

```sh
cd ~/DO280/labs/compreview-apps
oc login -u admin -p redhatocp \ 
  https://api.ocp4.example.com:6443
```

**23. Create and prepare the `workshop-support` namespace.**

```sh
oc create namespace workshop-support
oc label namespace workshop-support category=support
oc project workshop-support
oc adm policy add-cluster-role-to-group admin workshop-support
```

**24. Set resource quotas for `workshop-support`.**

```sh
oc create quota workshop-support \ 
 --hard=limits.cpu=4,limits.memory=4Gi,requests.cpu=3500m,requests.memory=3Gi
```

**25. Set limit ranges for `workshop-support`.**

Edit `limitrange.yaml` to match:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: workshop-support
  namespace: workshop-support
spec:
  limits:
    - default:
        cpu: 300m
        memory: 400Mi
      defaultRequest:
        cpu: 100m
        memory: 250Mi
      type: Container
```

Apply the limit range:

```sh
oc apply -f limitrange.yaml
```

**26. Create and configure the `project-cleaner` service account.**

```sh
oc create sa project-cleaner-sa
cd ~/DO280/labs/compreview-apps/project-cleaner
oc apply -f cluster-role.yaml
oc adm policy add-cluster-role-to-user project-cleaner -z project-cleaner-sa
```

**27. Configure the `project-cleaner` cron job.**

```sh
oc login -u do280-support -p redhat
oc apply -f cron-job.yaml
```

Verify cleanup by creating a test project:

```sh
oc new-project clean-test
oc project workshop-support
oc get jobs,pods
oc logs pod/<latest-project-cleaner-pod>
oc get project clean-test
```

**28. Deploy `beeper-db`.**

```sh
cd ~/DO280/labs/compreview-apps/beeper-api
oc apply -f beeper-db.yaml
oc get pod -l app=beeper-db
```

**29. Configure TLS for `beeper-api`.**

```sh
oc create secret tls beeper-api-cert \
  --cert certs/beeper-api.pem --key certs/beeper-api.key
```

Edit `deployment.yaml` to mount the certificate and enable TLS:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beeper-api
  namespace: workshop-support
spec:
  template:
    spec:
      containers:
        - name: beeper-api
          env:
            - name: TLS_ENABLED
              value: "true"
          volumeMounts:
            - name: beeper-api-cert
              mountPath: /etc/pki/beeper-api/
      volumes:
        - name: beeper-api-cert
          secret:
            secretName: beeper-api-cert
```

Apply deployment and service changes:

```sh
oc apply -f deployment.yaml
oc apply -f service.yaml
```

**30. Create a passthrough route for external access.**

```sh
oc create route passthrough beeper-api-https \ 
  --service beeper-api \ 
  --hostname beeper-api.apps.ocp4.example.com
```

Test API access:

```sh
curl -s --cacert certs/ca.pem https://beeper-api.apps.ocp4.example.com/api/beeps; echo
```

**31. Restrict database access using network policies.**

Verify access before applying policies:

```sh
oc debug --to-namespace="workshop-support" -- \
  nc -z -v beeper-db.workshop-support.svc.cluster.local 5432
```

Edit `db-networkpolicy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: workshop-support
spec:
  podSelector:
    matchLabels:
      app: beeper-db
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              category: support
          podSelector:
            matchLabels:
              app: beeper-api
      ports:
        - protocol: TCP
          port: 5432
```

Apply the network policy:

```sh
oc apply -f db-networkpolicy.yaml
```

Verify the database is inaccessible from other pods:

```sh
oc debug --to-namespace="workshop-support" -- \
  nc -z -v beeper-db.workshop-support.svc.cluster.local 5432
```

Check API access:

```sh
curl -s --cacert certs/ca.pem https://beeper-api.apps.ocp4.example.com/api/beeps; echo
```

**32. Restrict `beeper-api` ingress to OpenShift router pods.**

Edit `beeper-api-ingresspolicy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: beeper-api-ingresspolicy
  namespace: workshop-support
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              policy-group.network.openshift.io/ingress: ""
      ports:
        - protocol: TCP
          port: 8080
```

Apply the policy:

```sh
oc apply -f beeper-api-ingresspolicy.yaml
```

Verify access restriction:

```sh
oc debug --to-namespace="workshop-support" -- \
  nc -z -v beeper-api.workshop-support.svc.cluster.local 443
```

---

# OpenShift Cheat Sheet

## 1. Creating and Managing Namespaces
...

## 33. Add the Classroom Helm Repository

1. Add the Helm repository and examine its contents:
   ```sh
   helm repo add do280-repo http://helm.ocp4.example.com/charts
   ```
   
2. Verify the repository list:
   ```sh
   helm repo list
   ```
   
3. Search for available Helm charts:
   ```sh
   helm search repo
   ```

## 34. Deploy a Helm Chart

1. Log in as the developer user:
   ```sh
   oc login -u developer -p developer https://api.ocp4.example.com:6443
   ```
   
2. Create a new project:
   ```sh
   oc new-project compreview-package
   ```
   
3. Deploy the MySQL chart from the repository:
   ```sh
   helm install roster-database do280-repo/mysql-persistent
   ```
   
4. Monitor the pod status until `mysql-1-deploy` shows `Completed`:
   ```sh
   watch oc get pods
   ```
   
## 35. Deploy Kustomize Configurations

1. Navigate to the Kustomize directory:
   ```sh
   cd DO280/labs/compreview-package/
   ```
   
2. Examine the directory structure:
   ```sh
   tree
   ```
   
3. Verify that the production overlay generates the expected resources:
   ```sh
   oc kustomize roster/overlays/production/
   ```
   
4. Deploy the Kustomize configuration:
   ```sh
   oc apply -k roster/overlays/production/
   ```
   
5. Monitor the pod status:
   ```sh
   watch oc get pods
   ```
   
6. Get the application route:
   ```sh
   oc get route
   ```
   
7. Access the application via the displayed URL over HTTPS.
   
8. Return to the home directory:
   ```sh
   cd ~
   ```





