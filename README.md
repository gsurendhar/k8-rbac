# Kubernetes RBAC & AWS IAM Integration with EKS

## 🔐 What is RBAC?

**RBAC (Role-Based Access Control)** is a method of regulating access to resources in Kubernetes based on the roles assigned to users or service accounts.

RBAC in Kubernetes is implemented using four key objects:

- **Role** – A set of permissions (verbs on resources) within a namespace.
- **ClusterRole** – Like a Role, but applies across the entire cluster.
- **RoleBinding** – Grants permissions defined in a Role to a user or group in a namespace.
- **ClusterRoleBinding** – Grants permissions defined in a ClusterRole cluster-wide.

---

## 🧠 Key Concepts

| **Term** | **Description** |
|---------|------------------|
| **Verbs** | Actions you want to allow: `get`, `list`, `create`, `update`, `delete` |
| **Resources** | Kubernetes objects like `pods`, `services`, `deployments` |
| **Subjects** | Users, Groups, or ServiceAccounts that are granted permissions |

---

## 🛠️ Example: Roles for Different Users

| **User Type** | **Permissions** |
|--------------|-----------------|
| `Trainee`    | Read all resources in `roboshop` namespace |
| `Junior`     | Create pods |
| `Senior`     | Create and update resources |
| `Team Lead`  | Full permissions, including delete |

These permissions are assigned using **Roles** and **RoleBindings**.

---

## 🔧 kubectl Commands to Inspect RBAC

```bash
# List all roles in a namespace
kubectl get roles -n roboshop

# List all rolebindings in a namespace
kubectl get rolebindings -n roboshop
````

---

## 🔗 Integrating AWS IAM with Kubernetes RBAC (EKS)

Let’s say a new team member `Suresh` joins and needs access to the `roboshop` namespace in EKS.

### Step-by-step: Give IAM User Access to EKS

---

### 1. ✅ Create IAM User in AWS Console

* Go to IAM → Users → Create User `suresh`
* Do not assign permissions yet.

---

### 2. ✅ Create IAM Policy to Access EKS

* Go to IAM → Policies → Create Policy
* **Service**: `EKS`
* **Actions**: `DescribeCluster`
* **Resources**: Provide ARN of your EKS cluster (e.g. `arn:aws:eks:region:account-id:cluster/roboshop-dev`)
* Name it: `RoboshopEKSDescribe`
* Create and attach this policy to the user `suresh`

---

### 3. ✅ Update aws-auth ConfigMap

To allow `suresh` to authenticate to the cluster, update the `aws-auth` ConfigMap.

#### Option 1: Manually Edit ConfigMap

```bash
kubectl edit configmap aws-auth -n kube-system
```

Add the following under `mapUsers`:

```yaml
mapUsers:
  - userarn: arn:aws:iam::315069574700:user/suresh
    username: suresh
    groups:
      - roboshop-trainee
```

#### Option 2: Using eksctl

You can also use `eksctl` to add IAM users to the config map.

```bash
eksctl create iamidentitymapping \
  --cluster roboshop-dev \
  --region us-east-1 \
  --arn arn:aws:iam::315069574700:user/suresh \
  --username suresh \
  --group roboshop-trainee
```

---

### 4. ✅ Configure kubectl Access for Suresh

On Suresh's machine:

```bash
# Check current IAM identity
aws sts get-caller-identity

# Update kubeconfig for the cluster
aws eks update-kubeconfig --region us-east-1 --name roboshop-dev

# View the kubeconfig file
cat ~/.kube/config
```

---

## 📄 Notes on Role and RoleBinding

Below is an example Role that allows `get`, `list`, `watch` on pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: roboshop
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

Bind it to a user (or group) using RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: roboshop
subjects:
- kind: User
  name: suresh
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔐 ServiceAccount and IAM (IRSA) - Optional but Important

For **Pods to access AWS services securely**, use **IAM Roles for Service Accounts (IRSA)**:

* Create an OIDC identity provider in your EKS cluster.
* Create an IAM role with a trust policy that allows a ServiceAccount to assume it.
* Annotate the Kubernetes ServiceAccount with the IAM role ARN.
* Now the Pod using that ServiceAccount can access AWS resources using that IAM Role.

This is **much safer** than passing credentials manually.

More info: [IRSA Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

---

## 🧪 Tips and Troubleshooting

* `aws sts get-caller-identity` → Check current IAM identity.
* `kubectl auth can-i <verb> <resource> --as <user>` → Verify if a user has specific access.
* Always audit the `aws-auth` config map carefully—misconfiguration can lock out all users.

---

## 📚 Resources

* [Kubernetes RBAC Official Docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* [AWS EKS Authentication](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
* [IRSA - IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

---