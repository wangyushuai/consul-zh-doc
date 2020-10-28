# 概览

**1.5.0+:** This feature is available in Consul versions 1.5.0 and newer.

The `kubernetes` auth method type allows for a Kubernetes service account token to be used to authenticate to Consul. This method of authentication makes it easy to introduce a Consul token into a Kubernetes pod.

This page assumes general knowledge of [Kubernetes](https://kubernetes.io/) and the concepts described in the main [auth method documentation](https://www.consul.io/docs/acl/auth-methods).

## [»](consul-by-hashicorp.md#config-parameters)Config Parameters

The following auth method [`Config`](https://www.consul.io/api/acl/auth-methods#config) parameters are required to properly configure an auth method of type `kubernetes`:

* [`Host`](consul-by-hashicorp.md#host) `(string:)` - Must be a host string, a host:port pair, or a URL to the base of the Kubernetes API server.
* [`CACert`](consul-by-hashicorp.md#cacert) `(string:)` - PEM encoded CA cert for use by the TLS client used to talk with the Kubernetes API. NOTE: Every line must end with a newline \(`\n`\). If not set, system certificates are used.
* [`ServiceAccountJWT`](consul-by-hashicorp.md#serviceaccountjwt) `(string:)` - A Service Account Token \([JWT](https://jwt.io/)\) used by the Consul leader to validate application JWTs during login.
* [`MapNamespaces`](consul-by-hashicorp.md#mapnamespaces) `(bool:)`

  Enterprise - **Deprecated in Consul 1.8.0 in favor of** [**namespace rules**](https://www.consul.io/api/acl/auth-methods#namespacerules)**.** Indicates whether the auth method should attempt to map the Kubernetes namespace to a Consul namespace instead of creating tokens in the auth methods own namespace. Note that mapping namespaces requires the auth method to reside within the `default` namespace. Deprecated in Consul 1.8.0 in favor of [namespace rules](https://www.consul.io/api/acl/auth-methods#namespacerules).

* [`ConsulNamespacePrefix`](consul-by-hashicorp.md#consulnamespaceprefix) `(string:)`

  Enterprise - **Deprecated in Consul 1.8.0 in favor of** [**namespace rules**](https://www.consul.io/api/acl/auth-methods#namespacerules)**.** When `MapNamespaces` is enabled, this value will be prefixed to the Kubernetes namespace to determine the Consul namespace to create the new token within. Deprecated in Consul 1.8.0 in favor of [namespace rules](https://www.consul.io/api/acl/auth-methods#namespacerules).

* [`ConsulNamespaceOverrides`](consul-by-hashicorp.md#consulnamespaceoverrides) `(map:)`

  Enterprise - **Deprecated in Consul 1.8.0 in favor of** [**namespace rules**](https://www.consul.io/api/acl/auth-methods#namespacerules)**.** This field is a mapping of Kubernetes namespace names to Consul namespace names. If a Kubernetes namespace is present within this map, the value will be used without adding the `ConsulNamespacePrefix`. If the value in the map is `""` then the auth methods namespace will be used instead of attempting to determine an alternate namespace. Deprecated in Consul 1.8.0 in favor of [namespace rules](https://www.consul.io/api/acl/auth-methods#namespacerules).

### [»](consul-by-hashicorp.md#sample-config)Sample Config

```text
{
    ...other fields...
    "Config": {
        "Host": "https://192.0.2.42:8443",
        "CACert": "-----BEGIN CERTIFICATE-----\n...-----END CERTIFICATE-----\n",
        "ServiceAccountJWT": "eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9..."
    }
}
```

## [»](consul-by-hashicorp.md#rbac)RBAC

The Kubernetes service account corresponding to the configured [`ServiceAccountJWT`](https://www.consul.io/docs/acl/auth-methods/kubernetes#serviceaccountjwt) needs to have access to two Kubernetes APIs:

* [**TokenReview**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#create-tokenreview-v1-authentication-k8s-io)

  Kubernetes should be running with `--service-account-lookup`. This is defaulted to true in Kubernetes 1.7, but any versions prior should ensure the Kubernetes API server is started with this setting.

* [**ServiceAccount**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#read-serviceaccount-v1-core) \(`get`\)

The following is an example [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) configuration snippet to grant the necessary permissions to a service account named `consul-auth-method-example`:

```text
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: review-tokens
  namespace: default
subjects:
  - kind: ServiceAccount
    name: consul-auth-method-example
    namespace: default
roleRef:
  kind: ClusterRole
  name: system:auth-delegator
  apiGroup: rbac.authorization.k8s.io
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: service-account-getter
  namespace: default
rules:
  - apiGroups: ['']
    resources: ['serviceaccounts']
    verbs: ['get']
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: get-service-accounts
  namespace: default
subjects:
  - kind: ServiceAccount
    name: consul-auth-method-example
    namespace: default
roleRef:
  kind: ClusterRole
  name: service-account-getter
  apiGroup: rbac.authorization.k8s.io
```

## [»](consul-by-hashicorp.md#kubernetes-authentication-details)Kubernetes Authentication Details

Initially the [`ServiceAccountJWT`](https://www.consul.io/docs/acl/auth-methods/kubernetes#serviceaccountjwt) given to the Consul leader uses the TokenReview API to validate the provided JWT. The trusted attributes of `serviceaccount.namespace`, `serviceaccount.name`, and `serviceaccount.uid` are populated directly from the Service Account metadata.

The Consul leader makes an additional query, this time to the ServiceAccount API to check for the existence of an annotation of `consul.hashicorp.com/service-name` on the ServiceAccount object. If one is found its value will override the trusted attribute of `serviceaccount.name` for the purposes of evaluating any binding rules.

## [»](consul-by-hashicorp.md#trusted-identity-attributes)Trusted Identity Attributes

The authentication step returns the following trusted identity attributes for use in binding rule selectors and bind name interpolation.

| Attributes | Supported Selector Operations | Can be Interpolated |
| :--- | :--- | :--- |
| `serviceaccount.namespace` | Equal, Not Equal, In, Not In, Matches, Not Matches | yes |
| `serviceaccount.name` | Equal, Not Equal, In, Not In, Matches, Not Matches | yes |
| `serviceaccount.uid` | Equal, Not Equal, In, Not In, Matches, Not Matches | yes |
