##  Authentication vs Authorization in Kubernetes

Picture this:
You're an employee trying to enter your office building.

At the entrance, security checks your **ID card** — this is **authentication**: proving you belong.
Once inside, your **access card** only lets you into certain areas — that’s **authorization**: controlling what you’re allowed to do.

Simple, right?

* 🛂 **Authentication** = Who you are
* 🚪 **Authorization** = What you can access

Both are required — no valid ID? You’re stuck at the gate. Wrong permissions? You can't reach your desk.

---

##  In Kubernetes Terms

Kubernetes follows the same model:

* ✅ **Authentication** confirms your identity (certs, tokens, IAM).
* ✅ **Authorization** (via **RBAC**) defines what actions you can perform and where.

---

###  **Interview Question**

**Q:** Suppose in your team, a new member has joined and you need to give them access to the Kubernetes cluster. How will you do it?

**A:**
There are multiple ways to provide Kubernetes cluster access. In our case, we’re using **Amazon EKS**, so we prefer the **IAM-based access** method — it’s secure, scalable, and integrates well with AWS Identity management.

We can also use a manual method involving **Certificate Signing Request (CSR)** and a **custom kubeconfig**, but IAM-based access is generally more secure and manageable.

---

**Here’s how we typically handle access:**

**IAM-Based Access (EKS)**

1. Create or use an existing IAM user/role.
2. Update the `aws-auth` ConfigMap in the cluster to map the IAM identity to a Kubernetes group.
3. Assign appropriate **RBAC roles** (e.g., `view`, `edit`, `admin`) to that group using RoleBindings or ClusterRoleBindings.

**Manual CSR + kubeconfig**

1. Generate a private key and CSR for the user.
2. Submit the CSR to the Kubernetes API and approve it.
3. Extract the issued certificate.
4. Create a custom kubeconfig pointing to the user’s cert and key.
5. Define RBAC roles and bind them to the username.

---

We mostly rely on **IAM-based access** for production clusters, but I’ve also worked with **manual CSR setup** in dev/test environments where IAM integration wasn't available.

---

> Now let’s do both — all steps are included below with detailed instructions and visuals to guide you through each method.

---

#### **IAM-Based Access (EKS)**

For **authentication**, we use **IAM users**. In most organizations, IAM users are linked with **Single Sign-On (SSO)**, so a single login gives access to multiple AWS services, including EKS.

For **authorization**, we use the following Kubernetes RBAC resources:

* **Role** – Defines permissions inside a specific namespace, similar to an IAM policy for a user.
  *E.g., allow reading pods in the `dev` namespace.*

* **RoleBinding** – Attaches a Role to a user or group, like attaching a policy to an IAM user or group.
  *E.g., give `dev-user` read-only access in `dev` namespace.*

* **ClusterRole** – Grants cluster-wide permissions. Since a cluster can have multiple namespaces (used for microservices or projects), this lets you define access to all or specific resources across namespaces.
  *E.g., allow viewing pods in all namespaces.*

* **ClusterRoleBinding** – Attaches a ClusterRole to a user or group, giving them access at the cluster level.
  *E.g., give the `qa-team` read access to the entire cluster.*

---

###  Set Up Your Environment

Before configuring access, first create a Kubernetes cluster on EKS using `eksctl`.

> ⚠️ *You can use an EC2 instance with an administrator IAM role for this setup (not recommended for production).*

#### **Step 1: Install Required Tools on EC2 Instance**

* `eksctl`
* `kubectl`
* AWS CLI - check access

```bash
[ec2-user@ip-172-31-81-241 ~]$ eksctl version
0.210.0
[ec2-user@ip-172-31-81-241 ~]$ kubectl version --client
Client Version: v1.33.0-eks-802817d
Kustomize Version: v5.6.0
[ec2-user@ip-172-31-81-241 ~]$ aws s3 mb s3://k8s.aravi.project
make_bucket: k8s.aravi.project
```

#### **Step 2: Create a Cluster Config File (eks.yml)**

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: project
  region: us-east-1

managedNodeGroups:
  - name: project
    instanceType: t3.medium
    desiredCapacity: 3
    spot: true
```

#### **Step 3: Launch the EKS Cluster**

```bash
eksctl create cluster --config-file=eks.yml
```

> 📎 *Note: Cluster creation may take a few minutes.*

---

📘 **Need Full Setup?**
For a complete step-by-step EKS cluster creation guide, check out my detailed post here:

🔗 [Full Project Setup Guide – GitHub | Docker | EKS](https://www.linkedin.com/posts/aravindbasava_full-project-setup-guide-github-docker-activity-7340076843821076480-45Do?utm_source=share&utm_medium=member_desktop&rcm=ACoAAFStkm8ByBZVTiaoJaammsNqJjRfcKUCgVA)

---

```bash
2025-07-06 06:22:31 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2025-07-06 06:22:31 [✔]  EKS cluster "project" in "us-east-1" region is ready
[ec2-user@ip-172-31-81-241 ~]$ kubectl get nodes
NAME                             STATUS   ROLES    AGE     VERSION
ip-192-168-17-226.ec2.internal   Ready    <none>   2m43s   v1.32.3-eks-473151a
ip-192-168-38-241.ec2.internal   Ready    <none>   2m41s   v1.32.3-eks-473151a
ip-192-168-42-164.ec2.internal   Ready    <none>   2m40s   v1.32.3-eks-473151a
[ec2-user@ip-172-31-81-241 ~]$
```

Once the cluster is up and the nodes are in a **`Ready`** state, the next step is to **create an IAM policy** for the user. This policy will allow the user to **describe the EKS cluster**, which is necessary for accessing it using tools like `kubectl`.

---

#### 1.  **Create IAM Policy for EKS Access**

Follow these steps:

1. **Go to the AWS Console**
2. Search for **IAM** and open it
3. Click on **Policies** → **Create policy**
4. Under the **Service** section, choose **EKS**
5. In **Access level**, enable **DescribeCluster**
6. In the **Resources** section:

   * Choose **Specific**
   * Provide your **cluster name**
   * Add the **EKS cluster ARN** (recommended for security)
7. Click **Next**
8. Give the policy a meaningful name, e.g., `EksDescProject`
9. Click **Create policy**

> 💡 This policy allows the IAM user to run `kubectl` and describe EKS cluster details — it's the minimal requirement for authentication.

---

###  **2: Create IAM User and Attach the Policy**

Follow these steps:

1. Go to the **IAM** section in the AWS Console
2. Click on **Users** → **Create user**
3. Enter a **username** (e.g., `dev-user`) and click **Next**
4. Under **Permissions**, choose **Attach policies directly**
5. Search for the policy you created earlier (e.g., `EksDescProject`)
6. Select the policy and click **Next**
7. Review and click **Create user**

> ✅ This IAM user can now be mapped to your EKS cluster for controlled access using RBAC.

---
Offcial documentation links to create rbacs and aws config-ath map for authrozation with template yaml files to set up 

Grant IAM users access to Kubernetes with a ConfigMap
https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html

k8s Documentation for creating roles:-
https://kubernetes.io/docs/reference/access-authn-authz/rbac/

----

### **3.create a namespace and Role and Rolebing files and apply it.**

```bash
#namespace.yml
kind: Namespace
apiVersion: v1
metadata:
  name: project
  labels:
    name: project
    environment: dev
```

```bash
#rbac.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: project
  name: ReadOnlyProject
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: AttachReadOnlyProject
  namespace: project
subjects:
# You can specify more than one "subject"
- kind: User
  name: Aravind # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: ReadOnlyProject # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

In above manifest file in the first part 
api Resources - k8s resources are group into lot of groups and you can list with below command, you can specify on which api-group you need to give access.

```bash
[ec2-user@ip-172-31-81-241 ~]$ kubectl api-resources | wc -l
66
[ec2-user@ip-172-31-81-241 ~]$
```

resources - for example in "" core API group we have multip resoucres like pos , service , deployment, service. here we can specify which specic resources we need to give access so fine-grained access

verbs: here we can givw what action he can have like CRUD(Create, READ, UPDATE, DELATE) operations 
here we only giving read access.

second part
-------
in first part we set permissions now here we need to mention which use that permisstion to have



###  **Step 4: update aws-auth config file**

The `aws-auth` **ConfigMap** in an Amazon EKS (Elastic Kubernetes Service) cluster is a **critical configuration file** that manages **authentication and access control** for users and roles in the cluster. In this file we need add IAM user details like user name user ARN . based on this it will ablw to authorize the user, and decides  he is authorized or not.


```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

you'll get below similar output

```bash
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::115690362715:role/eksctl-project-nodegroup-k8s-rbac-NodeInstanceRole-FtGJ9sEaUvy1
      username: system:node:{{EC2PrivateDNSName}}
kind: ConfigMap
metadata:
  creationTimestamp: "2025-07-06T06:20:09Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "1339"
  uid: bd29dd6b-8f69-4799-8337-2588cf734a6d
```

copy and create a New yaml  with includeing IAM user details like below
---


```yaml
#aws-auth.yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::115690362715:role/eksctl-project-nodegroup-k8s-rbac-NodeInstanceRole-FtGJ9sEaUvy1
      username: system:node:{{EC2PrivateDNSName}}
# here you need to give role-name, IAM user name and ARN details 
  mapUsers: |
    - groups:
      -  ReadOnlyProject
      userarn: arn:aws:iam::115690362715:user/Aravind   
      username: Aravind 
kind: ConfigMap
# remove reource version but don't remove uid
metadata:
  creationTimestamp: "2025-07-06T06:20:09Z"
  name: aws-auth
  namespace: kube-system
#  resourceVersion: "1339"
  uid: bd29dd6b-8f69-4799-8337-2588cf734a6d
```

Now apply all file 



---




