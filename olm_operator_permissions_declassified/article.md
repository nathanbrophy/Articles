# OLM and Operator Permissions Declassified

If you are a cluster admin, an Operator developer, or a user of Operator software, then you have probably run into the following thought “How do I know what this Operator can do to my cluster and what is the recommended pattern for Operators to request permissions?”. This article will recommend a pattern for Operators to consume permissions on a cluster, and how a user can determine what permissions an Operator can perform.

![operator backed application install trail map](./images/operator%20backed%20application%20install%20trail%20mp.png)

## The “Too Long, Didn’t Read”

This article will explain why each bullet point below is a recommended practice, however, the key takeaways can be summarized with the following:

### Cluster Admins:

- Define an Operator Group with a Service Account that limits permissions to only those the Operator needs
- Install Operators in the minimum scope required to perform the task

### Operator Developers:

- Defer as many permissions to the Operator as possible (limit the number of permissions the Operand i.e., application, needs)
- Do not put Operand permissions or bindings to permissions in the Operator bundle
- Limit the amount of cluster scoped permissions used by the Operator or Operand


## Operator Lifecycle Manager (OLM) Operator Overview

The Operator Lifecycle Manager (OLM) does a great job of installing Operators to the cluster and automating the setup of resources an Operator needs (the CustomResourceDefinition, ServiceAccount, permissions). However, it is not always clear what OLM will apply to a cluster until after the install of an Operator completes. Compound this with the fact that OLM runs as cluster admin by default, and the importance of Operator permissions becomes apparent.

![olm and operator install relationship diagram](./images/olm%20operator%20relationship%20diagram.png)

### Operator Permissions and Scope (the Right Way)

Kubernetes permissions are enforced through Role Based Access Control (RBAC). RBAC means that every actor (user) has a role they perform, and every role has an associated set of minimum permissions needed to complete the function of that role. In order to prevent privilege escalation, an actor cannot create a role that has permissions they themselves do not possess. For example, an actor that performs the viewer role cannot create a role for an editor.

![The best practices diagram for the permission hierarchy through OLM](./images/operator%20permissions%20rings%20good.png)

The diagram shows the top of the hierarchy is the cluster admin. As we move down the hierarchy tree, the cluster admin could restrict the permissions of OLM using an Operator Group before installing the Operator. The installed Operator, then uses its permissions to create a set of manifests needed for the Operand to function. Any Kubernetes permissions the Operand needs to perform its role must be granted from the Operator (In the next section we will discuss how to breakout of this permission model).

Operators can be installed in multiple recommended modes:

1. All namespaces (Operator uses cluster scoped permissions to watch the entire cluster)
2. Own namespace (Operator uses a mix of namespace and cluster scoped permissions to watch its own namespace)

The aim of all Operator developers should be to reduce the number of cluster scoped permissions needed to the scope of the current namespace. Today, the scope of permissions and watch namespaces is decided at install time by the Operator installer. The future direction for install modes (and I think the right direction) is to have the Operator run in a control-plane namespace and use RBAC to allow an Operator to perform the functions of its role in allowed namespaces. At its core, an Operator is an actor that is an extension of the Kubernetes control plane to automate the install and day two operations for an Operand. Generally, Operands should not need access to the Kubernetes API (control-plane) and separating the controller namespace from the possible set of Operand namespaces provides a clear delineation of permissions.

## Operator Permissions (the Wrong Way)

As stated above, Kubernetes will prevent an actor from creating permissions they do not have. However, when installing through OLM there is a way for the Operand to “breakout” of the hierarchy and request more permissions than the Operator that created it.


![OLM permissions hierarchy when the bundle is used to store Operand permissions](./images/operator%20permissions%20rings%20bad.png)

OLM represents a version of an Operator as a bundle. A bundle can contain a discrete set of resources in the manifests directory. The notable resources a bundle can contain are:

- (cluster) Roles
- (cluster) Role Bindings
- Service Accounts

The permissions contained in the bundle cannot exceed the permissions of OLM, but they can exceed the permissions of the Operator. This means that an installed Operator may have very little interaction with the Kubernetes API, but the Operand created by the Operator does. This is an anti-practice because the Operand is not generally an extension of the control-plane. It is recommended to minimize the amount of Kubernetes permissions an Operand needs and defer the interaction with the control-plane to the Operator.

It is worth mentioning here that there are some valid use cases for injecting Operand permissions into the bundle of the Operator, but those use cases are highly specialized. In general, injecting Operand permissions into the bundle for the Operator is an anti-practice.

## What Permissions can an Operator Perform?

### After an Install

OLM will create a set of APIs (manifests) on the cluster as the result of an install. One of those APIs is the ClusterServiceVersion (CSV), and this manifest contains information needed to install a specific version of an Operator. How to get the CSV:

```raw
$ kubectl get csv my-example-operator.v1.0.0 -o yaml -n operator
```
Example Operator CSV Snippet:

![Example Operator CSV Snippet](./images/operator%20csv%20snippet.png)

The `spec.install.spec.permissions[]` field of the CSV holds the namespace scoped permissions OLM created for the Operator. These permissions define a role manifest in the Operator namespace.

The `spec.install.spec.clusterPermissions[]` field of the CSV holds the cluster scoped permissions OLM created for the Operator. These permissions define a clusterRole manifest on the cluster.

### Before an Install

There is not an “easy button” for determining the permissions of an Operator before an install. OLM uses a catalog image to install an Operator. The catalog image is a docker container that can be ran locally, and the exposed GRP API can be inspected on the local machine:

```raw
$ docker pull example.io/test/operator-catalog:latest

$ docker run --name catalog -d -p 50051:50051 example.io/test/operator-catalog:latest

$ grpcurl -plaintext -d ‘{“pkgName”: “example-operator”, “channelName”: “stable”}’ localhost:50051 api.Registry.GetBundleForChannel
```

This command will return the CSV information that would be used to install the Operator on an actual cluster. Note that the results may vary because this path does not contain the context of existing Operator installs on a live cluster. The CSV data can then be inspected (as it was in the above section) to determine a list of permissions the Operator will be granted through an install.

## Summary

Determining the permissions an Operator can perform on a cluster is not a trivial task. However, by developing Operators under a set of common best practices and refining the permissions OLM has through Operator Groups, you can help make understanding Operator permissions a delightful process.

## Special Thanks and Considerations

Special thanks to contributions made from my colleague, Chris Johnson, for initial collaboration in the core ideas and principles of the article.

Special thanks to my colleagues, Michele Chilanti, and, Denilson Nastacio, for providing suggestions and reviews to the article content.

## References and Useful Links
https://kubernetes.io/docs/tutorials/kubernetes-basics/
https://kubernetes.io/docs/reference/access-authn-authz/rbac/
https://operatorframework.io/
https://olm.operatorframework.io/docs/tasks/creating-operator-bundle/
https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/