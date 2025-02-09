---
id: meshstack.openshift.index
title: Integration
---

meshStack supports management of RedHat OpenShift platforms. OpenShift has a [Kubernetes](meshstack.kubernetes.index.md) core and provides additional services. It is available in both Open Source flavors (OKD) as well as enterprise offerings by RedHat.

meshStack supports project creation, configuration, user management and SSO for OpenShift.

## Integration Overview

To enable integration with OpenShift, operators deploy and configure the meshStack OpenShift Connector. Operators can configure one or multiple `PlatformInstance`s of `PlatformType` OpenShift. This makes OpenShift available to meshProjects like any other cloud platform in meshStack.

meshStack automatically configures OpenShift Projects and Permissions to integrate SSO with [meshStack Identity Federation](./meshstack.identity-federation.md).

## Prerequisites

### OpenShift Versions

meshStack currently supports OpenShift version 3.7 as either Open-Source (OKD) or OpenShift Enterprise variants.

### IdP Configuration

In order to integrate with [meshStack Identity Federation](./meshstack.identity-federation.md), operators need to configure the meshStack Identity Broker as an [OpenID Identity Provider in OpenShift](https://docs.okd.io/latest/install_config/configuring_authentication.html#OpenID) using the following settings:

```yml
identityProviders:
  - name: meshfed_oidc
    challenge: false
    login: true
    mappingMethod: claim
    provider:
      apiVersion: v1
      kind: OpenIDIdentityProvider
      clientID: meshfed-oidc
      clientSecret: ignored
      extraScopes:
      - email
      - profile
      - skipIdentifierSelection
      extraAuthorizeParameters:
        include_granted_scopes: "true"
      claims:
        id:
        - sub
        preferredUsername:
        - sub
        name:
        - preferred_username
        email:
        - email
      urls:
        authorize: https://sso.example.meshcloud.io/auth/realms/meshfed/protocol/openid-connect/auth?response_mode=query
        token: https://sso.example.meshcloud.io/auth/realms/meshfed/protocol/openid-connect/token
        userInfo: https://sso.example.meshcloud.io/auth/realms/meshfed/protocol/openid-connect/userinfo
```

### meshStack Service Account

The meshStack OpenShift Module uses a dedicated OpenShift ServiceAccount to work with OpenShift APIs on behalf of meshStack.
To create these credentials, create the following objects via `oc create -f <file>` as a Cluster Administrator.

> You can also use `oc replace -f <file>` to update existing definitions.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: meshfed-service
  annotations:
    io.meshcloud/meshstack.replicator-openshift.version: "1.0"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: meshfed-service
  annotations:
    io.meshcloud/meshstack.replicator-openshift.version: "1.0"
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  - user.openshift.io
  resources:
  - groups
  - identities
  - useridentitymappings
  - users
  verbs:
  - create
  - delete
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  - project.openshift.io
  resources:
  - projectrequests
  - projects
  verbs:
  - create
  - delete
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - resourcequotas
  - resourcequotas/status
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  - template.openshift.io
  resources:
  - templates
  - templateinstances
  - templateconfigs
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  - quota.openshift.io
  resources:
  - appliedclusterresourcequotas
  - clusterresourcequotas
  - clusterresourcequotas/status
  verbs:
  - create
  - delete
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  - authorization.openshift.io
  resources:
  - roles
  - rolebindings
  - clusterroles
  - clusterrolebindings
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  - authorization.openshift.io
  resources:
  - rolebindings
  verbs:
  - create
  - delete
  - update
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  - authorization.openshift.io
  resources:
  - clusterroles
  verbs:
  - bind
  resourceNames:
  - admin
  - edit
  - view
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: meshfed-service
  annotations:
    io.meshcloud/meshstack.replicator-openshift.version: "1.0"
subjects:
- kind: ServiceAccount
  name: meshfed-service
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: meshfed-service
```

Next, retrieve the access token for the service account:

```bash
oc get serviceaccount meshfed-service -o json | jq '.secrets[].name' | grep token | xargs oc describe secret
```

Operators need to securely inject this access token into the configuration of the OpenShift module.

## Landing Zones

By defining a Landing Zone for OpenShift certain configurations can be enforced during replication.

### Resource Quota

With OpenShift [ResourceQuotas](https://docs.openshift.com/container-platform/3.11/dev_guide/compute_resources.html) the number of resources inside a namespace (meshProject) can be limited. In order to setup such a quota limit write, or drag and drop, your OpenShift ResourceQuota file into the respective field when creating a Landing Zone.

### Templates

Templates can also be synchronized automatically with OpenShift if they are put into a Landing Zone and assigned to a meshProject. This is similar to the ResourceQuotas handling. This template then shows up in the web console and can be applied manually by the user. Currently only one template can be uploaded and managed this way.

It is possible to automatically instance this template inside a project via meshStacks project replication procedure. However doing so requires currently that the service principal, created with the rights above, gets all the additional rights required to instance the template. Like for example creating Pods. This might be not desirable and because of this this function is disabled by default. To enable automatic Template instancing set the flag `enableTemplateInstancing` to true:

```yml
replicator-openshift:
  platforms:
    - platform: okd.eu-de-central
      enableTemplateInstancing: true
```

### Default Labels

Labels are used to identify existing Resources inside OpenShift or Kubernetes projects. It is possible to define a set of default labels which get applied to every resource created via meshStack. Usually these managed resources are created by a Landing Zone configuration (e.g. [ResourceQuotas](https://docs.openshift.com/container-platform/3.11/admin_guide/quota.html) or [Templates](https://docs.openshift.com/container-platform/3.11/dev_guide/templates.html)).

In order to set these default labels use the replicator OpenShift platform configuration and set the property `default-resource-labels` like so:

```yml
replicator-openshift:
  platforms:
    - platform: okd.eu-de-central
      default-resource-labels:
        my-label-1: sample-value-123
        my-label-2: sample-value-456
```

### Dynamic Labels

You can also specify certain labels which are populated via data from the meshProject spec. This can be for example a costcenter id. You need to map these fields via the replicator config as well:

```yml
replicator-openshift:
  platforms:
    - platform: okd.eu-de-central
      meta-label-mappings:
        costCenter: io.meshcloud/costCenter
```

This configuration will map the value from the specs metadata with the key `costCenter` to an OpenShift label with the key `io.meshcloud/costCenter`. The datapoints available in the specs for mapping might be differnt depending on the configured environment of your meshStack installation. Please contact the [support](https://support.meshcloud.io/hc/en-us/requests/new) for more details.
