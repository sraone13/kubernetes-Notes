1. What is RBAC (Role-Based Access Control)

RBAC (Role-Based Access Control) determines whether a user is allowed to perform a specific action on a Kubernetes resource, based on the role assigned to that user.

Based on the role, the level of access varies.

Example:
Role	Access
Developer	Create, Read, Update
Monitoring	Read only
Admin	Create, Read, Update, Delete


2. Request Flow in Kubernetes

When a user performs any operation on the cluster, the request goes to the API Server.

The API Server:
Authenticates the user (valid user or not)
Authorizes the request (permission check via RBAC)

👉 Authentication happens first, authorization happens next.


3. kubeconfig File Basics

Cluster information is stored in:
~/.kube/config

This file can contain multiple users, clusters, and contexts.

Context:
A context contains:

Cluster
User
Default namespace


4. Certificate Authority (CA)

Kubernetes uses certificate-based authentication.
If a user presents a certificate signed by the cluster CA, the user is considered authenticated.


5. Creating a New User (Certificate-Based)

Step 1: Generate Private Key
openssl genrsa -out sravan.key 2048


This generates the user private key.

Step 2: Create Certificate Signing Request (CSR)
openssl req -new -key sravan.key -out sravan.csr -subj "/CN=sravan/O=dev/O=example.org"


CN → Username
O → Group name

Step 3: Verify Files
ls | grep sravan

Step 4: Sign CSR Using Kubernetes CA
openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -days 365 -in sravan.csr -out sravan.crt


Now the user certificate is generated.

6. Add User to kubeconfig
kubectl config set-credentials sravan --client-certificate=sravan.crt  --client-key=sravan.key


Verify in kubeconfig:

kubectl config view

7. Create Context for the User
kubectl config set-context sravan-kube --cluster=kubernetes --user=sravan --namespace=default


Verify:
kubectl config get-contexts


Switch context:
kubectl config use-context sravan-kube


8. Access Test (Without Permissions)
kubectl get pods

❌ Output: Forbidden

Reason:
User is authenticated
User is not authorized
No RBAC permissions assigned yet


9. Role and RoleBinding (Namespace Level)

Role YAML (read pods)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]


Apply:

kubectl apply -f role.yaml
kubectl get roles

RoleBinding YAML
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: User
  name: sravan
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io


Apply:
kubectl apply -f role-binding.yaml

Switch to Sravan user and Test as User
kubectl get pods
✅ Allowed

kubectl run nginx --image=nginx
❌ Not allowed (Sravan user as only read-only access)

10.Switch back to Kubernetes 

kubectl create namspace test
In this namspace pod will created

Kubectl run nginx --image=nginx -n test
The same pod will create with the test namespace

kubectl get pods -n test
You will see the pod with test namespace

11. Switch to the Sravan User

kubectl get pods -n test
❌ Forbidden

Namespace Scope Behavior
Roles and RoleBindings are namespace scoped
User permissions apply only to the namespace where RoleBinding exists



Switch Back to the Kubernetes
12. ClusterRole and ClusterRoleBinding

To avoid creating RoleBindings in every namespace:
Define Roles and Role bindings at the cluster level instead of namespaces
level, that where cluster role and cluster rol bindingcomes

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-cluster
rules:
- apiGroups: [""]
  resources:
    - pods
    - pods/log
  verbs:
    - get
    - list
    - watch

Apply ClusterRole
kubectl apply -f cluster-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sravan-pod-reader-cluster
subjects:
- kind: User
  name: sravan
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-reader-cluster
  apiGroup: rbac.authorization.k8s.io

Apply ClusterRoleBinding
kubectl apply -f cluster-role-binding.yaml


13.Switch back to the sravan user:

Kubectl get pods -n test
Now user can access pod across namespaces, because we given at cluster level

kubectl get pods -A


14. Groups in RBAC

Instead of assigning permissions to individual users, users can be added to groups
Groups are referenced in RoleBinding / ClusterRoleBinding

Example:

subjects:
- kind: Group
  name: dev

15. Service Accounts (Application Users)

Key Points
=>Service Accounts are application users

=>Automatically created when a namespace is created

=>Pods use Service Accounts to authenticate with the API Server

I=>f not specified, the pod uses the default Service Account


16. Creating a Custom Service Account

kubectl get sa
kubectl get sa -n test
kubectl create sa test-sa
kubectl get sa


17. Pod Using Default Service Account

Create pod:

apiVersion: v1
kind: Pod
metadata:
  name: kubectl-pod
spec:
  serviceAccountName: test-sa
  containers:
  - name: kubectl
    image: bitnami/kubectl
    command:
      - sleep
      - "2000"


kubectl apply -f pod.yaml


Inside pod:

kubectl exec -it pod -- bash
kubectl get pods


❌ Error
Reason: Default Service Account has no permissions

18. Pod Using Custom Service Account

Update pod.yaml:

spec:
  serviceAccountName: test-sa

Delete and recreate pod.

Test again:

kubectl exec -it pod -- bash
kubectl get pods


❌ Still error
Reason: Service Account exists, but no RoleBinding assigned

19. Grant Permissions to Service Account

Update RoleBinding:

subjects:
- kind: ServiceAccount
  name: test-sa


Apply:

kubectl apply -f role-binding.yaml


Test again:

kubectl get pods


✅ Success

20. Permission Validation
kubectl auth can-i get pods --as=system:serviceaccount:default:test-sa


19. Key Takeaways

Authentication ≠ Authorization

Role / RoleBinding → Namespace level

ClusterRole / ClusterRoleBinding → Cluster level

Users → Humans

Service Accounts → Applications

Pods authenticate using Service Accounts