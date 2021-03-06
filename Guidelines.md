# Operator Guidelines

Kubernetes Operators extend Kubernetes API to manage third-party software as native Kubernetes objects. 
Number of Operators are being built for platform elements like databases, queues, loggers, etc. 
We are seeing more and more fit-for-purpose application platforms being created by composing multiple Operators together.

While working on such a custom platform for one of our customers, we observed challenges that arise when using multiple Operators together. 
For example, some of these Operators tend to introduce a new CLI for its end users. 
We reflected more on usability of Operators while building tools that work on multiple Operators 
e.g.: tools for [discovery](https://github.com/cloud-ark/kubediscovery) and [lineage tracking](https://github.com/cloud-ark/kubeprovenance)
of Custom Resources created by Operators. Examples of such usability challenges can be:

  * Some of the Operators introduce new CLIs. Usability becomes an issue when end users have to learn multiple CLIs to use more than one Operators in a cluster.

  * Some of the Operator type definitions do not follow OpenAPI Specification. This makes it hard to generate documentation for custom resources similar to native Kubernetes resources.

Our study of existing community Operators from this perspective led us to come up with Operator development guidelines that will improve overall usability of Operators. The primary goal of these guidelines is : cluster admin should be able to easily compose multiple Operators together to form a platform stack; and application developers should be able to discover and consume Operators effortlessly.


Here are those guidelines:

## Design guidelines

[1) Design your Operator with declarative API/s and avoid inputs as imperative actions](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#1-design-your-operator-with-declarative-apis-and-avoid-inputs-as-imperative-actions)

[2) Consider to use kubectl as the primary interaction mechanism](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#2-consider-to-use-kubectl-as-the-primary-interaction-mechanism)

[3) Decide your Custom Resource Metrics Collection strategy](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#3-decide-your-custom-resource-metrics-collection-strategy)

[4) Prefer to register CRDs as part of Operator Helm chart rather than in code](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#4-prefer-to-register-crds-as-part-of-operator-helm-chart-rather-than-in-code)

## Implementation guidelines

[5) Set OwnerReferences for underlying resources owned by your Custom Resource](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#5-set-ownerreferences-for-underlying-resources-owned-by-your-custom-resource)

[6) Use Helm chart or ConfigMap for Operator configurables](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#6-use-helm-chart-or-configmap-for-operator-configurables)

[7) Use ConfigMap or Annotation or Spec definition for Custom Resource configurables](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#7-use-configmap-or-annotation-or-spec-definition-for-custom-resource-configurables)

[8) Define underlying resources created by Custom Resource as Annotation on CRD registration YAML](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#8-define-underlying-resources-created-by-custom-resource-as-annotation-on-crd-registration-yaml)

[9) Make your Custom Resource type definitions compliant with Kube OpenAPI](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#9-make-your-custom-resource-type-definitions-compliant-with-kube-openapi)

## Packaging guidelines

[10) Generate Kube OpenAPI Spec for your Custom Resources](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#10-generate-kube-openapi-spec-for-your-custom-resources)

[11) Package Operator as Helm Chart](https://github.com/cloud-ark/kubeplus/blob/master/Guidelines.md#11-package-operator-as-helm-chart)

Detailed guidelines:

# Design guidelines

## 1) Design your Operator with declarative API/s and avoid inputs as imperative actions

A declarative API allows you to declare or specify the desired state of your custom resource. Prefer declarative state over any imperative actions in Custom Resource Spec Type definition. Custom controller code should be written such that it reconciles the current state with the desired state by performing diff of the current state with the desired state. This enables end users to use your custom resources just like any other Kubernetes resources with declarative state based inputs. For example, when writing a Postgres Operator, the custom controller should be written to perform diff of the existing value of ‘users’ with the desired value of ‘users’ based on the received desired state and perform the required actions (such as adding new users, deleting current users, etc.).


Note that the diff-based implementation approach for custom controllers is essentially an extension of
the level-triggered approach recommended in the [general guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/controllers.md) 
for developing Kubernetes controllers.

An example where underlying imperative actions are exposed in the Spec is this 
[MySQL Backup Custom Resource Spec](https://github.com/oracle/mysql-operator/blob/master/examples/backup/backup.yaml#L7).
Here the fact that MySQL Backup is done using mysqldump tool is exposed in the Spec.
In our view such internal details should not be exposed in the Spec as it prevents Custom Resource Type definition 
to evolve independently without affecting its users.


## 2) Consider to use kubectl as the primary interaction mechanism

When designing your Operator you should try to support most of its actions through kubectl. Kubernetes contains various mechanisms such as Custom Resource Definitions, Aggregated API servers, Custom Sub-resources. Before considering to introduce new CLI for your Operator, validate if you can use these mechanisms instead. 
Refer to [this blog post](https://medium.com/@cloudark/comparing-kubernetes-api-extension-mechanisms-of-custom-resource-definition-and-aggregated-api-64f4ca6d0966) to learn more about them.


## 3) Decide your Custom Resource Metrics Collection strategy

Plan for metrics collection of custom resources managed by your Operator. This information is useful for understanding effect of various actions on your custom resources over time and improving traceability. One option is to collect metrics inside your custom controller and then expose them with solutions like Prometheus. On the other hand, if you decide not to include this in your custom controller code, then the alternate is to leverage Kubernetes Audit Logs. You can use also external tooling like kubeprovenance that works with Kubernetes Audit Logs for this.


## 4) Prefer to register CRDs as part of Operator Helm chart rather than in code

Prefer to register CRDs as part of Operator Helm Chart rather than in code. This has following advantages:

  * All installation artifacts and dependencies will be in one place — the Operator’s Helm Chart.

  * It is easy to modify and/or evolve the CRD through Helm chart.


# Implementation guidelines

## 5) Set OwnerReferences for underlying resources owned by your Custom Resource

A custom resource instance will typically create one or more other Kubernetes resource instances, such as Deployment, Service, Secret etc., as part of its instantiation. Here this custom resource is the owner of its underlying resources that it manages. Custom controller should be written to set OwnerReference on such managed Kubernetes resources. They are key for correct garbage collection of custom resources. OwnerReferences also help with finding composition tree of your custom resource instances. 

Some examples of Operators that use OwnerReferences are: [Etcd Operator](https://github.com/coreos/etcd-operator/blob/master/pkg/cluster/cluster.go#L351),
[Postgres Operator](https://github.com/cloud-ark/kubeplus/blob/master/postgres-crd-v2/controller.go#L508), and 
[MySQL Operator](https://github.com/oracle/mysql-operator/blob/master/pkg/resources/services/service.go#L34).



## 6) Use Helm chart or ConfigMap for Operator configurables

Typically Operators will need to support some form of customization. For example, 
[this MySQL Operator](https://github.com/oracle/mysql-operator/blob/master/docs/tutorial.md#configuration) supports following customization settings: whether to deploy
the Operator cluster-wide or within a particular namespace, which version of MySQL should be installed, etc.
If you have followed previous guideline and have Helm chart for your Operator then use Helm's values YAML file to specify
such parameters. If not, use ConfigMap for this purpose. This guideline ensures that Kubernetes Administrators
can interact and use the Operator using Kubernetes native's interfaces.



## 7) Use ConfigMap or Annotation or Spec definition for Custom Resource configurables

An Operator generally needs to take inputs for underlying resource's configuration parameters. We have seen three different approaches being used towards this in the community and anyone should be fine to use based on your Operator design. They are - using ConfigMaps, using Annotations, or using Spec definition itself. 

[Nginx Custom Controller](https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/customization) supports both ConfigMap and Annotation.
[Oracle MySQL Operator](https://github.com/oracle/mysql-operator/blob/master/docs/user/clusters.md) uses ConfigMap.
[PressLabs MySQL Operator](https://github.com/presslabs/mysql-operator) uses Custom Resource [Spec definition](https://github.com/presslabs/mysql-operator/blob/master/examples/example-cluster.yaml#L22).

Similar to guideline #5, this guideline ensures that application developers can interact and use Custom Resources using Kubernetes's native interfaces.



## 8) Define underlying resources created by Custom Resource as Annotation on CRD registration YAML

Use an annotation on the Custom Resource Definition to specify the underlying Kubernetes resources that will be created and managed by the Custom Resource. An example of this can be seen for our Sample Postgres resource below:

```
  kind: CustomResourceDefinition
  metadata:
    name: postgreses.postgrescontroller
    annotations:
      composition: Deployment, Service
```

Otherwise this composition information will be available only in custom controller code and be hidden from end users in case they require it for traceability or any other reason. It is also possible to build tools like kubediscovery that show Object composition tree for custom resource instances by using this information.


## 9) Make your Custom Resource type definitions compliant with Kube OpenAPI

Kubernetes API details are documented using Swagger v1.2 and OpenAPI. Kube OpenAPI supports a subset of OpenAPI features to satisfy kubernetes use-cases. As Operators extend Kubernetes API, it is important to follow Kube OpenAPI features to provide consistent user experience. Following actions are required to comply with Kube OpenAPI.

Add documentation on your type definition and on the various fields in it.
The field names need to be defined using following pattern:
Kube OpenAPI name validation rules expect the field name in Go code and field name in JSON to be exactly the same with just the first letter in different case (Go code requires CamelCase, JSON requires camelCase).

When defining the types corresponding to your custom resources, you should use kube-openapi annotation — “+k8s:openapi-gen=true’’ in the type definition to enable generating OpenAPI Spec documentation for your custom resources. An example of this annotation on type definition can be seen on CloudARK sample Postgres custom resource.
```
  // +k8s:openapi-gen=true
  type Postgres struct {
    :
  }
```

# Packaging guidelines

## 10) Generate Kube OpenAPI Spec for your Custom Resources

We have developed a [tool](https://github.com/cloud-ark/kubeplus/tree/master/openapi-spec-generator) that you can use for generating Kube OpenAPI Spec for your custom resources. 
It wraps code available in [kube-openapi repository](https://github.com/kubernetes/kube-openapi) 
in an easy to use script. 
The generated Kube OpenAPI Spec documentation for sample Postgres custom resource is 
[here](https://github.com/cloud-ark/kubeplus/blob/master/postgres-crd-v2/postgres-crd-v2-chart/openapispec.json).


## 11) Package Operator as Helm Chart

Create a Helm chart for your Operator. The chart should include two things:

  * Registration of all Custom Resources managed by the Operator. Examples of this can be seen in 
CloudARK [sample Postgres Operator](https://github.com/cloud-ark/kubeplus/blob/master/postgres-crd-v2/postgres-crd-v2-chart/templates/deployment.yaml) and in 
[this MySQL Operator](https://github.com/oracle/mysql-operator/blob/master/mysql-operator/templates/01-resources.yaml).

  * Kube Open API Spec for your custom resources. The Spec will be useful as reference documentation and can be leveraged by different tools.



## Evaluation of community Operators

Here is a table showing conformance of different community Operators to above guidelines.

| Operator      | URL           | Guidelines satisfied  | Comments     |
| ------------- |:-------------:| ---------------------:| ------------:|
| Oracle MySQL Operator | https://github.com/oracle/mysql-operator | 2, 3, 4, 5, 6, 7 | 1: Not satisfied because of exposing mysqldump in Spec <br> 8: Not satisfied as composition of CRDs not defined <br>9, 10: Not satisfied, PR that addresses them: https://github.com/oracle/mysql-operator/pull/216 
| PressLabs MySQL Operator | https://github.com/presslabs/mysql-operator  | 1, 2, 3, 5, 6, 7, 9, 10, 11 | 4: Not satisfied because CRD installed in Code <br> 8: Not satisfied as composition of  CRDs not defined
| CloudARK sample Postgres Operator | https://github.com/cloud-ark/kubeplus/tree/master/postgres-crd-v2 | 1, 2, 3, 4, 5, 8, 9, 10, 11 | 6, 7: Work-in-Progress



If you are interested in getting your Operator checked against these guidelines, 
[create a Issue](https://github.com/cloud-ark/kubeplus/issues) with link to your Operator Code and we will analyze it.









