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

###  In Kubernetes Terms

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

## **IAM-Based Access (EKS)**

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

🐳 K8s – 3/3
 Kubernetes Project – 1/2
🔗 (https://www.linkedin.com/posts/aravindbasava_full-project-setup-guide-github-docker-activity-7340076843821076480-45Doutm_source=share&utm_medium=member_desktop&rcm=ACoAAFStkm8ByBZVTiaoJaammsNqJjRfcKUCgVA)

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

Reference screenhots:- 

<img width="2556" height="1362" alt="Image" src="https://github.com/user-attachments/assets/a58cda5e-090c-46c7-944d-ac4021d526ac" />
<img width="2543" height="923" alt="Image" src="https://github.com/user-attachments/assets/0b97d940-1481-4d56-b50b-f441fa0a6b29" />

---

####  **2: Create IAM User and Attach the Policy**

Follow these steps:

1. Go to the **IAM** section in the AWS Console
2. Click on **Users** → **Create user**
3. Enter a **username** (e.g., `dev-user`) and click **Next**
4. Under **Permissions**, choose **Attach policies directly**
5. Search for the policy you created earlier (e.g., `EksDescProject`)
6. Select the policy and click **Next**
7. Review and click **Create user**

> ✅ This IAM user can now be mapped to your EKS cluster for controlled access using RBAC.

<img width="2538" height="818" alt="Image" src="https://github.com/user-attachments/assets/980b4572-f980-4d6e-a619-5f0f9fa97dca" />

<img width="2557" height="921" alt="Image" src="https://github.com/user-attachments/assets/2ad6940e-aef5-4db5-9b21-add38b13e1dd" />

<img width="2559" height="1213" alt="Image" src="https://github.com/user-attachments/assets/4395d977-3839-4938-974a-dd3e1f9d0f07" />

<img width="2518" height="577" alt="Image" src="https://github.com/user-attachments/assets/81abc6a3-79a5-44eb-bb88-a441fda0242a" />


---

#### **3. Create a Namespace, Role, and RoleBinding and apply them**

Refer to official documentations:

`Grant IAM users access via ConfigMap (AWS Docs)`

https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html


`Kubernetes RBAC Reference`
https://kubernetes.io/docs/reference/access-authn-authz/rbac/

----

####  Create the Kubernetes Manifests

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

### Explanation

In the above manifest:

#### First part (Role):

* **apiGroups** – Kubernetes resources are grouped into multiple API groups. You can view the list with the command:

  ```bash
  kubectl api-resources | wc -l
  ```

  ```bash
  [ec2-user@ip-172-31-81-241 ~]$ kubectl api-resources | wc -l  
  66  
  [ec2-user@ip-172-31-81-241 ~]$
  ```

* **resources** – For example, the core API group (`""`) includes resources like `pods`, `services`, `deployments`, etc. You can specify which exact resource access is needed — this enables **fine-grained permission control**.

* **verbs** – Specifies which actions the user can perform: `create`, `read`, `update`, `delete`.
  In this example, we’re only granting **read** access (`get`, `watch`, `list`).

#### Second part (RoleBinding):

* In the Role, we defined **what permissions** are available.
* In the RoleBinding, we assign **which user** should have those permissions. Here, it's given to user `Aravind`.



###  **Step 4: update aws-auth config file**

The `aws-auth` **ConfigMap** in an Amazon EKS cluster is a **critical configuration file** that manages both **authentication and authorization** for IAM users and roles.

In this file, we need to add IAM user details — like **username** and **ARN**. This lets the cluster authenticate the IAM user and determine whether the user is authorized.

<img width="2548" height="954" alt="Image" src="https://github.com/user-attachments/assets/02cc4e27-5d3c-4780-a65d-1528aad1e948" />


```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

You’ll get an output similar to:

```yaml
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

Now copy this output and create a new file including IAM user details:

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

### Apply All Files and Verify

```bash
[ec2-user@ip-172-31-81-241 RBAC]$ kubectl apply -f 01-namespace.yml
namespace/project created
[ec2-user@ip-172-31-81-241 RBAC]$ kubectl apply -f 02-rbac.yaml
role.rbac.authorization.k8s.io/ReadOnlyProject created
rolebinding.rbac.authorization.k8s.io/AttachReadOnlyProject created
[ec2-user@ip-172-31-81-241 RBAC]$ kubectl apply -f 03-aws-auth.yaml
Warning: resource configmaps/aws-auth is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
configmap/aws-auth configured
[ec2-user@ip-172-31-81-241 RBAC]$ kubectl get configmap aws-auth -n kube-system -o yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::115690362715:role/eksctl-project-nodegroup-k8s-rbac-NodeInstanceRole-FtGJ9sEaUvy1
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: "- groups:\n  -  ReadOnlyProject\n  userarn: arn:aws:iam::115690362715:user/Aravind
    \  \n  username: Aravind \n"
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"mapRoles":"- groups:\n  - system:bootstrappers\n  - system:nodes\n  rolearn: arn:aws:iam::115690362715:role/eksctl-project-nodegroup-k8s-rbac-NodeInstanceRole-FtGJ9sEaUvy1\n  username: system:node:{{EC2PrivateDNSName}}\n","mapUsers":"- groups:\n  -  ReadOnlyProject\n  userarn: arn:aws:iam::115690362715:user/Aravind   \n  username: Aravind \n"},"kind":"ConfigMap","metadata":{"annotations":{},"creationTimestamp":"2025-07-06T06:20:09Z","name":"aws-auth","namespace":"kube-system","uid":"bd29dd6b-8f69-4799-8337-2588cf734a6d"}}
  creationTimestamp: "2025-07-06T06:20:09Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "16519"
  uid: bd29dd6b-8f69-4799-8337-2588cf734a6d
[ec2-user@ip-172-31-81-241 RBAC]$

```
---

Final check , as i created eksctl with ec2-user, root user don't have access to cluster

```bash
[ec2-user@ip-172-31-81-241 ~]$ sudo su -
-bash: kubectll: command not found
[root@ip-172-31-81-241 ~]# kubectl get all
E0706 08:04:08.788097   31193 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E0706 08:04:08.789687   31193 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E0706 08:04:08.791150   31193 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E0706 08:04:08.792613   31193 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E0706 08:04:08.794057   31193 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
The connection to the server localhost:8080 was refused - did you specify the right host or port?
[root@ip-172-31-81-241 ~]#
```

----
### ✅ Now Let’s Configure AWS CLI with IAM User and Access Kubernetes Resources

---

####  Important: Who Has Access?

In our setup:

* The **EKS cluster** was created using the **`ec2-user`**, and this user is treated as the **cluster admin**.
* If you switch to **`root` user**, you'll notice that `kubectl` **doesn't have access** by default — because access is tied to the user's identity in the kubeconfig and aws-auth mapping.

---

####  Generate Access Keys for IAM User

1. Go to the **IAM dashboard** → select the user **Aravind**
2. Navigate to **Security credentials** → click **Create access keys**
3. Choose **Command Line Interface (CLI)**, acknowledge the prompt, and click **Create Access Key**
4. **Copy the credentials** — these will be used to authenticate via AWS CLI

<img width="2559" height="1164" alt="Image" src="https://github.com/user-attachments/assets/a0cd4391-25f8-4a33-8ae4-4da12f576572" />

----

###  kubeconfig File — Why It’s Needed?

So far, we have:

* Created an **IAM policy**
* Created an **IAM user**
* Attached the policy to the user
* Created **RBAC manifests** (Role, RoleBinding)
* Updated the **aws-auth ConfigMap** for authorization

But we still need one more file — the **`kubeconfig` file**.

This file contains all the **authentication** and **authorization** info required by the `kubectl` CLI to talk to the cluster.

 **Every `kubectl` command checks the kubeconfig file first.**
If the config allows access, the command executes. Otherwise, it's denied.

 **Tip:**
If you copy the kubeconfig file from `ec2-user` (admin) to `root`, then `root` can also access the cluster — even with **admin rights**. So copying is **quick but risky** in production.


**Default kubeconfig location:**

```bash
$HOME/.kube/config
```

**Reference to create kubeconfig using IAM user:**

https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html


###  Set up kubeconfig using AWS CLI (as root user)

```bash
[root@ip-172-31-81-241 ~]# aws sts get-caller-identity
{
    "UserId": "AIDARV35O25N7WT647MLX",
    "Account": "115690362715",
    "Arn": "arn:aws:iam::115690362715:user/Aravind"
}
[root@ip-172-31-81-241 ~]# aws eks update-kubeconfig --region us-east-1 --name project
Added new context arn:aws:eks:us-east-1:115690362715:cluster/project to /root/.kube/config
[root@ip-172-31-81-241 ~]#
```

 Now kubeconfig is created in **root user’s home directory**, but access is still limited by RBAC.

Let’s test access:

```bash
[root@ip-172-31-81-241 ~]# kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "Aravind" cannot list resource "nodes" in API group "" at the cluster scope

[root@ip-172-31-81-241 ~]# kubectl get pod -n project
No resources found in project namespace.

[root@ip-172-31-81-241 ~]# kubectl get pod -n kube-system
Error from server (Forbidden): pods is forbidden: User "Aravind" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

 This confirms our IAM and RBAC setup is working — **Aravind** has access only to the `project` namespace and cannot view cluster-level resources yet.
now we can access only pods details only in single namespacae and not able to get other cluster details like nodes 

###  Grant Cluster-Level Permissions to Aravind

Let’s now switch back to **`ec2-user` (the admin)** and assign **cluster-level read access** using a `ClusterRole` and `ClusterRoleBinding`.

```bash
#ClusterLevel.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: ClusterReadOnly
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets","persistentvolumes","nodes"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
metadata:
  name: AttachClusterReadOnly
subjects:
- kind: User
  name: Aravind # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: ClusterReadOnly
  apiGroup: rbac.authorization.k8s.io
```

###  Apply the Cluster-Level Access Manifests

```bash
[ec2-user@ip-172-31-81-241 ~]$ vi ClusterLevel.yml
[ec2-user@ip-172-31-81-241 ~]$ kubectl apply -f ClusterLevel.yml
clusterrole.rbac.authorization.k8s.io/ClusterReadOnly created
clusterrolebinding.rbac.authorization.k8s.io/AttachClusterReadOnly created
[ec2-user@ip-172-31-81-241 ~]$ sudo su -
Last login: Sun Jul  6 08:03:50 UTC 2025 on pts/6
[root@ip-172-31-81-241 ~]# kubectl get nodes
NAME                             STATUS   ROLES    AGE    VERSION
ip-192-168-17-226.ec2.internal   Ready    <none>   150m   v1.32.3-eks-473151a
ip-192-168-38-241.ec2.internal   Ready    <none>   150m   v1.32.3-eks-473151a
ip-192-168-42-164.ec2.internal   Ready    <none>   150m   v1.32.3-eks-473151a
[root@ip-172-31-81-241 ~]#
```

Success! Now the `Aravind` IAM user has **read-only access to cluster-level resources**, and your RBAC setup is working cleanly and securely.


🔥 **Absolutely perfect twist to end your post with both a warning and a real-world “hacking-style” trick!** Here’s your section cleaned up just slightly for grammar and flow — while keeping your tone, structure, commands, and the "playground" vibe intact.

---

##  Let’s Complicate Things a Bit — Because It’s My Playground 😎

Previously, the `root` user didn’t have access to the Kubernetes cluster because it wasn't the user who created the EKS cluster. The actual cluster admin is `ec2-user`.

But hey... root is root.

Let’s do what only **root** can do — create a new Linux user and sneak admin access by copying the kubeconfig from `ec2-user`.

### 🔨 Steps

```bash
[ec2-user@ip-172-31-81-241 ~]$ sudo su -
Last login: Sun Jul  6 08:50:43 UTC 2025 on pts/2

# Create a new user
[root@ip-172-31-81-241 ~]# useradd -m basava

# Switch to that user's home directory
[root@ip-172-31-81-241 ~]# cd /home/basava/
[root@ip-172-31-81-241 basava]# ls -a
.  ..  .bash_logout  .bash_profile  .bashrc

# Create a .kube folder
[root@ip-172-31-81-241 basava]# mkdir .kube

# Switch to new user
[root@ip-172-31-81-241 basava]# sudo su - basava

# Try kubectl (expected to fail)
[basava@ip-172-31-81-241 ~]$ kubectl get nodes
E0706 ... connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?

# Exit back to root
[basava@ip-172-31-81-241 ~]$ exit
logout

# Copy kubeconfig from ec2-user (admin) to basava
[root@ip-172-31-81-241 basava]# cp -r /home/ec2-user/.kube/config /home/basava/.kube/
```

Now switch user or test as root again:

```bash
[root@ip-172-31-81-241 ~]# kubectl get nodes
NAME                             STATUS   ROLES    AGE    VERSION
ip-192-168-17-226.ec2.internal   Ready    <none>   166m   v1.32.3-eks-473151a
ip-192-168-38-241.ec2.internal   Ready    <none>   166m   v1.32.3-eks-473151a
ip-192-168-42-164.ec2.internal   Ready    <none>   166m   v1.32.3-eks-473151a
```

 Boom! Now `basava` has full cluster access — all thanks to copying the kubeconfig file of the admin (`ec2-user`).

---

### ⚠️ Root Knows All — Be Careful!

> Previously, `root` had minimal Kubernetes access. But root can do **anything** on the system — including creating users and **copying the kubeconfig of an admin user**.

 This is why:

* **You should never give root privileges to anyone casually**
* Avoid storing admin-level kubeconfig files in shared or easily accessible paths
* Use **fine-grained Linux user permissions** and audit logins regularly

---

###  Ending Note

This was a fun but realistic example of why RBAC + IAM control alone isn't enough — **system-level security matters too**. Kubernetes RBAC can’t stop `root` from copying files at the OS level.

> Security is a multi-layered game — and in this playground, the root always wins. 💥

**clean up eks-cluster**

 eksctl delete cluster --config-file=eks.yml


---


##  Manual Access Method — CSR + Kubeconfig Setup

As of now, we’ve completed **IAM-Based Access (EKS)** and successfully granted cluster access using it.

Now let’s explore the **manual method** — where we create **Certificate Signing Request (CSR)** files and generate a **kubeconfig** file manually. This approach is useful for **testing environments**, labs, or where IAM integration is not available.

Let’s walk through it step-by-step below. 

---
Create a Testing env with kops or kueadm cluster

Going with Kops

Install Kops package

```bash
curl -LO https://github.com/kubernetes/kops/releases/download/v1.28.0/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
kops version
```

IAM role previously added with administrator policies

create a s3 bucket

aws s3  mb s3://k8s-rbac.aravi


set environment variables

```bash
export KOPS_STATE_STORE=s3://k8s-rbac.aravi
export NAME=rbac.k8s.local
```

create cluster

```bash
kops create cluster \
  --name=$NAME \
  --zones=us-east-1a,us-east-1b \
  --node-count=2 \
  --cloud=aws \
  --yes
```

validate cluster

```bash
kops validate cluster --wait 10m


INSTANCE GROUPS
NAME                            ROLE            MACHINETYPE     MIN     MAX     SUBNETS
control-plane-us-east-1a        ControlPlane    t3.medium       1       1       us-east-1a
nodes-us-east-1a                Node            t3.medium       1       1       us-east-1a
nodes-us-east-1b                Node            t3.medium       1       1       us-east-1b

NODE STATUS
NAME                    ROLE            READY
i-007fd47e738b7cb00     control-plane   True
i-0deee415f09c2f64f     node            True
i-0f0c91d686e6cc279     node            True

Your cluster rbac.k8s.local is ready
[ec2-user@ip-172-31-81-241 ~]$
```

> ✅ Create a Kubernetes user `test-env` with a signed client certificate
> ✅ Generate a `kubeconfig` for `test-env`
> ✅ Apply a **read-only RBAC role** (e.g., allow only `get`, `list`, `watch` across all namespaces)

---

## ✅ Step-by-Step for User `test-env`

---

### 🔹 **1. Generate Private Key & CSR**

```bash
openssl genrsa -out test-env.key 2048

openssl req -new -key test-env.key -out test-env.csr -subj "/CN=test-env/O=readers"
```

---

### 🔹 **2. Base64 Encode the CSR**

```bash
cat test-env.csr | base64 | tr -d '\n'
```

Copy the output.

---

### 🔹 **3. Create a Kubernetes CSR YAML (`test-env-csr.yaml`)**

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: test-env
spec:
  request: <PASTE_BASE64_HERE>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
```

Replace `<PASTE_BASE64_HERE>` with the base64-encoded CSR string.

Save as `test-env-csr.yaml`.

---

### 🔹 **4. Apply and Approve the CSR**

```bash
kubectl apply -f test-env-csr.yaml
kubectl certificate approve test-env
```

---

### 🔹 **5. Retrieve the Signed Certificate**

```bash
kubectl get csr test-env -o jsonpath='{.status.certificate}' | base64 -d > test-env.crt
```

Now you have:

* `test-env.key`
* `test-env.crt`

---

### 🔹 **6. Export Cluster Info for Kubeconfig**

```bash
CLUSTER_NAME=aja.k8s.local
CLUSTER_SERVER=$(kubectl config view -o jsonpath="{.clusters[?(@.name==\"$CLUSTER_NAME\")].cluster.server}")
CLUSTER_CA=$(kubectl config view --raw -o jsonpath="{.clusters[?(@.name==\"$CLUSTER_NAME\")].cluster.certificate-authority-data}")
```

---

### 🔹 **7. Create `kubeconfig` File for `test-env`**

```bash
cat <<EOF > test-env.kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: ${CLUSTER_SERVER}
    certificate-authority-data: ${CLUSTER_CA}
  name: ${CLUSTER_NAME}
users:
- name: test-env
  user:
    client-certificate-data: $(base64 -w 0 test-env.crt)
    client-key-data: $(base64 -w 0 test-env.key)
contexts:
- context:
    cluster: ${CLUSTER_NAME}
    user: test-env
  name: test-env-context
current-context: test-env-context
EOF
```

> This `kubeconfig` is fully portable. Save it as `test-env.kubeconfig`.

---

### 🔹 **8. Apply RBAC Read-Only Role**

Create a file `test-env-readonly-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints", "configmaps", "secrets", "namespaces", "nodes"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: test-env-readonly-binding
subjects:
- kind: User
  name: test-env
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: readonly
  apiGroup: rbac.authorization.k8s.io
```

Apply it:

```bash
kubectl apply -f test-env-readonly-rbac.yaml
```

---

Now we can test:

```bash
KUBECONFIG=./test-env.kubeconfig kubectl get pods --all-namespaces
KUBECONFIG=./test-env.kubeconfig kubectl delete pod <pod-name>  # ❌ should be denied
```

---

##  **Kubernetes User Access Methods – Complete List**

###  1. **IAM-based Access (EKS Specific)**

* Used in: **Amazon EKS**
* Auth via: **IAM user or role**
* Managed by: `aws-iam-authenticator` + `aws-auth` ConfigMap
* Tools: `eksctl`, `kubectl`
*  Secure, AWS-native
*  Ideal for: EKS production clusters

---

###  2. **OIDC-based Access (SSO / SAML Integration)**

* Used in: EKS, GKE, AKS, self-managed clusters
* Auth via: **Identity Providers** (Okta, Azure AD, Google, etc.)
* Integration via: OIDC plugins / `kubectl` with OIDC tokens
* Tools: `Dex`, `Keycloak`, `kube-oidc-proxy`, `pinniped`
*  Enables SSO, MFA, group-based access
*  Ideal for: Enterprise environments

---

###  3. **Client Certificate-Based Access (Manual CSR method)**

* Used in: Any K8s cluster (Kops, kubeadm, on-prem)
* Auth via: **x.509 certificate**
* Steps: Generate key + CSR → approve → get cert → make kubeconfig
* Very manual, but good for labs, non-cloud clusters
* Ideal for: Learning internals, on-prem clusters

---

###  4. **Static Token-Based Access**

* Auth via: Token specified in kubeconfig
* Created manually or via static file referenced in API server
*  Not recommended for production
*  Good for: Testing, demos, bootstrapping

---

###  5. **ServiceAccounts (For Pods / Automation)**

* Not for human users
* Mounted automatically in pods for in-cluster access
* Can also be used by automation (e.g., GitLab Runner, ArgoCD)
*  Ideal for: CI/CD, apps needing API access

---

###  6. **Kube API Gateway Solutions (Optional Layer)**

* Tools: **Teleport**, **KubeGate**, **Kubeapps**, **Rancher**, etc.
* Features: Web UI + RBAC + SSO + Audit
* Helps provide:

  * Centralized access
  * Session tracking
  * MFA, Role switching
*  Ideal for: Teams managing access across multiple clusters

---

##  Optional Mentions (if your audience is more advanced)

| Method                               | Description                                                      | When Useful                    |
| ------------------------------------ | ---------------------------------------------------------------- | ------------------------------ |
| **RBAC Manager Tools**               | Tools like `rbac-manager`, `rbac-sync`, or `rbac-lookup`         | Simplifies role/user mapping   |
| **External Authentication Webhooks** | Custom webhook that the API server uses to authenticate users    | Rare; used in regulated setups |
| **VPN + Internal Cluster DNS**       | Combined with private kubeconfigs for access to private clusters | Common in on-prem clusters     |

---

**comparison table** 

| Method           | Recommended For       | Pros              | Cons            |
| ---------------- | --------------------- | ----------------- | --------------- |
| IAM via `eksctl` | EKS clusters          | Secure, native    | AWS-specific    |
| OIDC / SSO       | Enterprises           | MFA, central auth | Requires setup  |
| Manual Cert/CSR  | Self-managed clusters | Full control      | Manual, complex |
| Static Token     | Demos/testing         | Simple            | Insecure        |
| ServiceAccount   | Apps/automation       | Easy to bind      | Not for humans  |

---

