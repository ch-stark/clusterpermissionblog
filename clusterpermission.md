# How ClusterPermission is evolving in RHACM

Managing Kubernetes across many clusters can quickly get complicated. Red Hat Advanced Cluster Management (RHACM) makes this easier by giving you one central place to control everything. A key part of this control is the **ClusterPermission** resource. You can fine its upstream repository [here](https://github.com/stolostron/cluster-permission).
The ClusterPermission controller utilizes the ManifestWork API to ensure the creation, update, and removal of RBAC resources on each managed cluster. The ClusterPermission API safeguards these distributed RBAC resources against unintended modifications and removal.

With recent improvements and its growing use in core RHACM feature  ClusterPermission is becoming essential for managing **Role-Based Access Control (RBAC)** across multiple clusters.


## Why ClusterPermission has become a critical component in Multi-Cluster RBAC

Traditional Kubernetes RBAC defines who can do what and where. But when you're dealing with a fleet of clusters, setting up individual roles on each one quickly becomes difficult.

RHACM's ClusterPermission resource bridges this gap. It lets you define RBAC rules right from your central RHACM hub, and then those rules are automatically pulled and applied to your specific managed clusters.

What makes ClusterPermission so powerful for modern multi-cluster operations?

* **Centralized Control:** Define your access rules once on the hub, and RHACM applies them to as many clusters as you need. This cuts down on manual work significantly.
* **GitOps-Ready:** ClusterPermission objects are just YAML files. You can store them in Git, which means your access policies are version-controlled, auditable, and can be automated.
* **Least Privilege:** It helps you give applications or components on your managed clusters only the permissions they truly need, boosting your security.

### ClusterPermission in Action: Real-World Use Cases

ClusterPermission is already being used to power important features in RHACM:

Application Lifecycle (Push Model): 

While RHACM's ApplicationSet preferred setup uses a "pull model", there are scenarios when the hub needs to "push" resources directly to a managed cluster.
**NOTE**: This is the default way ArgoCD/Gitops-Operator works.
ClusterPermission ensures that the Service Account on those managed clusters have exactly the right permissions to handle its operations, without being over-privileged.

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: clusterpermission-msa-subject-sample
  namespace: cluster1
spec:
  roles:
  - namespace: mortgage
    rules:
    - apiGroups: ["apps"]
      resources: ["deployments"]
      verbs: ["get", "list", "create", "update", "delete", "patch"]  
  roleBindings:
  - namespace: mortgage
    roleRef:
      kind: Role
    subject:
      apiGroup: authentication.open-cluster-management.io
      kind: ManagedServiceAccount
      name: managed-sa-sample
  clusterRoleBinding:
    subject:
      apiGroup: authentication.open-cluster-management.io
      kind: ManagedServiceAccount
      name: managed-sa-sample
```
**NOTE**: Apart from the typical RBAC binding subject resources (Group, ServiceAccount, and User), ManagedServiceAccount can also serve as a subject. When the binding subject is a ManagedServiceAccount, the controller computes and generates RBAC resources based on the ServiceAccount managed by the ManagedServiceAccount (in the examples the ApplicationManager).
**NOTE**: ClusterPermission is a resource which is only applied on the Hub (in the namespace of the Managed-Cluster it should be applied to).

### Virtual Machine Actions (Kubevirt Integration):

This is a prime example of ClusterPermission's power. If you're managing Virtual Machines (VMs) and Virtual Machine Instances (VMIs) with Kubevirt through RHACM, ClusterPermission grants the precise permissions needed. For instance, it allows an automation Service Account to start, stop, restart, pause, or unpause VMs, without giving it broader admin access.

Here’s that VM actions example for a cluster (in this case aro-central):

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: vm-actions
  namespace: aro-central # Target managed cluster
spec:
  clusterRole:
    rules:
      - apiGroups:
          - subresources.kubevirt.io
        resources:
          - virtualmachines/start
          - virtualmachines/stop
          - virtualmachines/restart
          - virtualmachineinstances/pause
          - virtualmachineinstances/unpause
        verbs:
          - update
  clusterRoleBinding:
    subject:
      kind: ServiceAccount
      name: vm-actor
      namespace: open-cluster-management-agent-addon # The Service Account performing VM a
```

## The Evolution: More Flexible ClusterPermission

Recent updates have made the ClusterPermission resource much more versatile:

### Reference Existing Roles

Before, if you wanted to use ClusterPermission, you had to fully define every rule for the Role or ClusterRole you were creating. Now, you can simplify **reference existing Roles or ClusterRoles** on the managed cluster. This is a huge improvement because it lets you:

* **Use Built-in Roles:** Easily link to standard Kubernetes roles (like `admin`, `edit`, `view`) or specialized roles provided by other operators (like Kubevirt.io's VM management roles).
* **Simplify Definitions:** No more duplicating long lists of API groups, resources, and verbs. This makes your YAML cleaner and reduces errors.
* **Integrate Seamlessly:** It’s easier to work with existing RBAC setups on your clusters.

Here's an example of binding to an existing Kubevirt.io ClusterRole:

```
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: vm-admin-access # Make sure this is a regular space before 'name'
  namespace: my-prod-cluster # Make sure this is a regular space before 'namespace'
spec:
  clusterRoleBinding:
    name: vm-admin-crb
    roleRef:
      apiGroup: rbac.k8s.io
      kind: ClusterRole
      name: kubevirt.io:admin
    subject:
      kind: ServiceAccount
      name: vm-automation-sa
      namespace: vm-ops
```

### Bind to Multiple Users, Groups, or Service Accounts

A single ClusterPermission RoleBinding or ClusterRoleBinding can now connect a role to multiple subjects at once. This means you don't need separate ClusterPermission objects or duplicate bindings when several users, groups, or Service Accounts need the same set of permissions.

Take a look at this example that binds one role to a Service Account, a user, and a group:

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: app-dev-permissions
  namespace: dev-cluster-01 # This specifies the target managed cluster
spec:
  roleBindings:
    - namespace: app-dev
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: argocd-application-controller-custom # An existing role
      subjects: # List multiple subjects here
      - namespace: openshift-gitops
        kind: ServiceAccount
        name: sa-app-deployer
      - apiGroup: rbac.authorization.k8s.io
        kind: User
        name: dev-lead-bob
      - apiGroup: rbac.authorization.k8s.io
        kind: Group
        name: app-team-a
```
See also this example related to OpenShift Virtualization

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: kubevirt-edit
  namespace: jg-test-1
spec:
  roleBindings:
  - namespace: kubevirt-workspace-1
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: kubevirt.io:view
    subjects:
    - kind: User
      name: Bob
      apiGroup: rbac.authorization.k8s.io
    - kind: User
      name: Kike
      apiGroup: rbac.authorization.k8s.io      
  - namespace: kubevirt-workspace-1
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: kubevirt.io:edit
    subjects:
    - kind: User
      name: Bob
      apiGroup: rbac.authorization.k8s.io
    - kind: User
      name: Kike
      apiGroup: rbac.authorization.k8s.io      
  - namespace: kubevirt-workspace-2
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: kubevirt.io:view
    subjects:
    - kind: User
      name: Bob
      apiGroup: rbac.authorization.k8s.io
    - kind: User
      name: Kike
      apiGroup: rbac.authorization.k8s.io      
  - namespace: kubevirt-workspace-2
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: kubevirt.io:edit
    subjects:
    - kind: User
      name: Bob
      apiGroup: rbac.authorization.k8s.io
    - kind: User
      name: Kike
      apiGroup: rbac.authorization.k8s.io
```

## ClusterPermission and the Aggregated API Server

The increased importance of ClusterPermission is deeply connected to the aggregated API server in RHACM. This aggregated API server acts as a central hub where all permissions from your managed clusters (including those defined by ClusterPermission) are collected and unified. 
The Aggregated API Server on the Hub will be watching the ClusterPermission objects and summarize the results such that it can handle the RBAC API calls e.g. from RHACM Search feature.




## Beyond VMs: The Future of Fine-Grained Control

We expect ClusterPermission to be used in many more situations where specific, delegated permissions are needed for:

* **Smarter Search Results:** Imagine a user who only has access to `namespace-A` on `cluster1`. Thanks to ClusterPermission and the aggregated API, when they use the RHACM search, they will only see resources from `namespace-A` on `cluster1`, correctly filtered by their precise permissions. This is a big step up from just seeing everything on a cluster.
* **Fine-Grained RBAC for Hosted Control Planes (HCP):** If you're using HCP to manage your OpenShift clusters, ClusterPermission is essential. It allows you to enforce granular RBAC within those hosted clusters, even when users are interacting through the RHACM hub. For example, you can restrict a user's access to specific namespaces on a child cluster even if they have broader permissions on the HCP host cluster.

RHACM will offer a UI to define and validate the ClusterPermissions.
In the future the users will also coming from remote IDP (we might do something closely like GroupSyncOperator).


## Conclusion

The ClusterPermission resource is becoming a core part of how RHACM delivers truly comprehensive and secure multi-cluster management. With its new ability to reference existing roles and bind to multiple subjects, it offers incredible flexibility and efficiency for managing permissions at scale. As its use cases expand from application deployment to specialized workloads like Virtual Machines, ClusterPermission is set to become an indispensable tool for anyone operating a distributed Kubernetes environment with Red Hat Advanced Cluster Management.

