# rbac, namespaces and cluster roles

Now we have learnt how to secure our pods, its a good idea to separate our applications from each other via namespaces and give them the appropriate level of access   
within the cluster to run. So we will first look at namespaces to separate our applications.

## Namespaces

Lets look at the default namespaces available to us.  
We do this by issuing `kubectl get namespaces`
In the last lab we deployed our deployment to the default namespace as we did not define anything.
Kubernetes will place any pods in the default namespace unless another one is specified.

For the next part of the lab we will create a namespace to use for the rest of the lab. We will do that by issuing  
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-namespace
EOF
```

Then if we check our namespaces again via `kubectl get namespaces` if we were successful then we should see the new namespace.

## Cluster roles, Service accounts and Role bindings

Now we have our namespace set up we are going to create a service account and give it full access to that namespace only.

We are now going to create a service account for the namespace that we created earlier.

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-service-account
  namespace: webapp-namespace
EOF
```
Then we will create a role giving us full permissions to the namespace

```
cat <<EOF | kubectl apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: webapp-role
  namespace: webapp-namespace
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
EOF
```
Then we will create a role binding to tie it all together

```
cat <<EOF | kubectl apply -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: webapp-role-binding
  namespace: webapp-namespace
subjects:
  - kind: ServiceAccount
    name: webapp-service-account
    namespace: webapp-namespace
roleRef:
  kind: Role
  name: webapp-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

Now lets deploy our application into our new namespace.

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: webapp-deployment
  namespace: webapp-namespace
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: scottyc/webapp:latest
        ports:
        - containerPort: 3000
          hostPort: 3000
EOF
```

Then we can check our pods simulating the privileges of the service-account:

`kubectl get pods --namespace=webapp-namespace --as system:serviceaccount:webapp-namespace:webapp-service-account`

(Check [here](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-subjects) for more info about rbac subjects)

Now we have limited the blast radius of our application to only the namespace that it resides in. 
So there will be no way that we can leak configmaps or secrets from other applications that are not in this namespace.

## Users and Certificates

ServiceAcccounts are for services inside Kubernetes, to authenticate Users we can use "User" instead

```
cat <<EOF | kubectl apply -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: webapp-role-binding
  namespace: webapp-namespace
subjects:
  - kind: User
    name: alice
    namespace: webapp-namespace
roleRef:
  kind: Role
  name: webapp-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

`kubectl get pods --namespace=webapp-namespace --as alice`

## Generate Certificate

Create a **private key** for `alice`:

```
$ openssl genrsa -out alice.key 2048
```

Create **certificate sign request** for `alice`:

```
$ openssl req -new -key alice.key -out alice.csr -subj "/CN=alice/O=myorg"
```

Sign the **certificate** for `alice`:

```
$ openssl x509 -req -in alice.csr -CA /etc/kubernetes/pki//ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out alice.crt -days 365
```


# Create Alice Context in Kubectl

Create `alice` User in `kubectl`:

```
kubectl config set-credentials alice --client-certificate="$(pwd)/alice.crt"  --client-key="$(pwd)/alice.key"
```

Create `alice` Context in `kubectl`:

```
kubectl config set-context alice --cluster=kubernetes --user=alice
```

Retrieve Contexts:

```
$ kubectl config get-contexts
CURRENT NAME      CLUSTER   AUTHINFO  NAMESPACE
*       kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
        alice     minikube  alice     blue
```
