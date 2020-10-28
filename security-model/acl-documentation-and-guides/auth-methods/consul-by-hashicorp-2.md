# Consul by HashiCorp

This feature is available in [Consul Enterprise](https://www.hashicorp.com/products/consul/) version 1.8.0 and newer.

The `oidc` auth method can be used to authenticate with Consul using [OIDC](https://en.wikipedia.org/wiki/OpenID_Connect). This method allows authentication via a configured OIDC provider using the user's web browser. This method may be initiated from the Consul UI or the command line.

This page assumes general knowledge of [OIDC concepts](https://developer.okta.com/blog/2017/07/25/oidc-primer-part-1) and the concepts described in the main [auth method documentation](https://www.consul.io/docs/acl/auth-methods).

Both the [`jwt`](https://www.consul.io/docs/acl/auth-methods/jwt) and the [`oidc`](https://www.consul.io/docs/acl/auth-methods/oidc) auth method types allow additional processing of the claims data in the JWT.

## [»](consul-by-hashicorp-2.md#jwt-vs-oidc-auth-methods)JWT vs OIDC Auth Methods

Since both the `oidc` and `jwt` auth methods ultimately operate on JWTs as bearer tokens, it may be confusing to know which is right for a given use case.

* **JWT**: The user or application performing the Consul login must already be in possession of a valid JWT to begin. There is no browser interaction required. This is ideal for machine-oriented headless login where an operator may have already arranged for a valid JWT to be dropped on a VM or provided to a container.
* **OIDC**: The user performing the Consul login does not have a JWT nor do they even need to know what that means. This is ideal for human-oriented interactive login where an operator or administrator may have deployed SSO widely and doesn't want to have the burden of tracking and distributing Consul ACL tokens to any authorized coworker who may need to have access to a Consul instance. Browser interaction is required. **This is only available in** [**Consul Enterprise**](https://www.hashicorp.com/products/consul/).

## [»](consul-by-hashicorp-2.md#config-parameters)Config Parameters

The following auth method [`Config`](https://www.consul.io/api/acl/auth-methods#config) parameters are required to properly configure an auth method of type `oidc`:

* [`OIDCDiscoveryURL`](consul-by-hashicorp-2.md#oidcdiscoveryurl) `(string:)` - The OIDC Discovery URL, without any .well-known component \(base path\).
* [`OIDCDiscoveryCACert`](consul-by-hashicorp-2.md#oidcdiscoverycacert) `(string: "")` - PEM encoded CA cert for use by the TLS client used to talk with the OIDC Discovery URL. NOTE: Every line must end with a newline \(`\n`\). If not set, system certificates are used.
* [`OIDCClientID`](consul-by-hashicorp-2.md#oidcclientid) `(string:)` - The OAuth Client ID configured with your OIDC provider.
* [`OIDCClientSecret`](consul-by-hashicorp-2.md#oidcclientsecret) `(string:)` - The OAuth Client Secret configured with your OIDC provider.
* [`AllowedRedirectURIs`](consul-by-hashicorp-2.md#allowedredirecturis) `(array)` - Comma-separated list of allowed values for `redirect_uri`. Must be non-empty.
* [`ClaimMappings`](consul-by-hashicorp-2.md#claimmappings) `(map[string]string)` - Mappings of claims \(key\) that [will be copied to a metadata field](consul-by-hashicorp-2.md#trusted-identity-attributes-via-claim-mappings) \(value\). Use this if the claim you are capturing is singular \(such as an attribute\).

  When mapped, the values can be any of a number, string, or boolean and will all be stringified when returned.

* [`ListClaimMappings`](consul-by-hashicorp-2.md#listclaimmappings) `(map[string]string)` - Mappings of claims \(key\) [will be copied to a metadata field](consul-by-hashicorp-2.md#trusted-identity-attributes-via-claim-mappings) \(value\). Use this if the claim you are capturing is list-like \(such as groups\).

  When mapped, the values in each list can be any of a number, string, or boolean and will all be stringified when returned.

* [`OIDCScopes`](consul-by-hashicorp-2.md#oidcscopes) `(array)` - Comma-separated list of OIDC scopes.
* [`JWTSupportedAlgs`](consul-by-hashicorp-2.md#jwtsupportedalgs) `(array)` - JWTSupportedAlgs is a list of supported signing algorithms. Defaults to `RS256`. \([Available algorithms](https://github.com/hashicorp/consul/blob/master/vendor/github.com/coreos/go-oidc/jose.go#L7)\)
* [`BoundAudiences`](consul-by-hashicorp-2.md#boundaudiences) `(array)` - List of `aud` claims that are valid for login; any match is sufficient.
* [`VerboseOIDCLogging`](consul-by-hashicorp-2.md#verboseoidclogging) `(bool: false)` - Log received OIDC tokens and claims when debug-level logging is active. Not recommended in production since sensitive information may be present in OIDC responses.

### [»](consul-by-hashicorp-2.md#sample-config)Sample Config

```text
{
    ...other fields...
    "Config": {
        "AllowedRedirectURIs": [
            "http://localhost:8550/oidc/callback",
            "http://localhost:8500/ui/oidc/callback"
        ],
        "BoundAudiences": [
            "V1RPi2MYptMV1RPi2MYptMV1RPi2MYpt"
        ],
        "ClaimMappings": {
            "http://example.com/first_name": "first_name",
            "http://example.com/last_name": "last_name"
        },
        "ListClaimMappings": {
            "http://consul.com/groups": "groups"
        },
        "OIDCClientID": "V1RPi2MYptMV1RPi2MYptMV1RPi2MYpt",
        "OIDCClientSecret": "...(omitted)...",
        "OIDCDiscoveryURL": "https://my-corp-app-name.auth0.com/"
    }
}
```

## [»](consul-by-hashicorp-2.md#jwt-verification)JWT Verification

JWT signatures will be verified against public keys from the issuer via OIDC discovery. Keys will be fetched from the OIDC Discovery URL during authentication and OIDC validation criteria \(e.g. `iss`, `aud`, etc.\) will be applied.

## [»](consul-by-hashicorp-2.md#oidc-authentication)OIDC Authentication

Consul includes two built-in OIDC login flows: the Consul UI, and the CLI using [`consul login`](https://www.consul.io/commands/login).

### [»](consul-by-hashicorp-2.md#redirect-uris)Redirect URIs

An important part of OIDC auth method configuration is properly setting redirect URIs. This must be done both in Consul and with the OIDC provider, and these configurations must align. The redirect URIs are specified for an auth method with the [`AllowedRedirectURIs`](consul-by-hashicorp-2.md#allowedredirecturis) parameter. There are different redirect URIs to configure the Consul UI and CLI flows, so one or both will need to be set up depending on the installation.

#### [»](consul-by-hashicorp-2.md#consul-ui)Consul UI

Logging in via the Consul UI requires a redirect URI of the form: `http://localhost:8500/ui/oidc/callback` or `https://{host:port}/ui/oidc/callback`

The "host:port" must be correct for the Consul agent serving the Consul UI.

#### [»](consul-by-hashicorp-2.md#cli)CLI

If you plan to support authentication via `consul login -type=oidc -method=`, a localhost redirect URI must be set \(usually this is `http://localhost:8550/oidc/callback`\). Logins via the CLI may specify a different host and/or listening port if needed, and a URI with this host/port must match one of the configured redirected URIs. These same "localhost" URIs must be added to the provider as well.

### [»](consul-by-hashicorp-2.md#oidc-login)OIDC Login

#### [»](consul-by-hashicorp-2.md#consul-ui-1)Consul UI

1. Click the "Log in" link at the top right of the menu bar.
2. Click one of the "Continue with..." buttons for your OIDC auth method of choice.
3. Complete the authentication with the configured provider.

#### [»](consul-by-hashicorp-2.md#cli-1)CLI

```text
$ consul login -method=oidc -type=oidc -token-sink-file=consul.token

Complete the login via your OIDC provider. Launching browser to:

    https://myco.auth0.com/authorize?redirect_uri=http%3A%2F%2Flocalhost%3A8550%2Foidc%2Fcallback&client_id=r3qXc2bix9eF...
```

The browser will open to the generated URL to complete the provider's login. The URL may be entered manually if the browser cannot be automatically opened.

The callback listener may be customized with the following optional parameters. These are typically not required to be set:

The callback listener defaults to listen on `localhost:8550`. If you want to customize that use the optional flag [`-oidc-callback-listen-addr=`](https://www.consul.io/commands/login#oidc-callback-listen-addr).

## [»](consul-by-hashicorp-2.md#oidc-configuration-troubleshooting)OIDC Configuration Troubleshooting

The amount of configuration required for OIDC is relatively small, but it can be tricky to debug why things aren't working. Some tips for setting up OIDC:

* Monitor the log output for the Consul servers. Important information about OIDC validation failures will be emitted.
* Ensure Redirect URIs are correct in Consul and on the provider. They need to match exactly. Check: http/https, 127.0.0.1/localhost, port numbers, whether trailing slashes are present.
* [`BoundAudiences`](consul-by-hashicorp-2.md#boundaudiences) is optional and typically not required. OIDC providers will use the `client_id` as the audience and OIDC validation expects this.
* Check your provider for what scopes are required in order to receive all of the information you need. The scopes "profile" and "groups" often need to be requested, and can be added by setting `[OIDCScopes](#oidcscopes)="profile,groups"` on the auth method.
* If you're seeing claim-related errors in logs, review the provider's docs very carefully to see how they're naming and structuring their claims. Depending on the provider, you may be able to construct a simple `curl` [implicit grant](https://developer.okta.com/blog/2018/05/24/what-is-the-oauth2-implicit-grant-type) request to obtain a JWT that you can inspect. An example of how to decode the JWT \(in this case located in the `access_token` field of a JSON response\):

  ```text
  cat jwt.json | jq -r .access_token | cut -d. -f2 | base64 -D
  ```

* The [`VerboseOIDCLogging`](consul-by-hashicorp-2.md#verboseoidclogging) option is available which will log the received OIDC token if debug level logging is enabled. This can be helpful when debugging provider setup and verifying that the received claims are what you expect. Since claims data is logged verbatim and may contain sensitive information, this option should not be used in production.

## [»](consul-by-hashicorp-2.md#trusted-identity-attributes-via-claim-mappings)Trusted Identity Attributes via Claim Mappings

Data from JWT claims can be returned from the authentication step as trusted identity attributes for use in binding rule selectors and bind name interpolation.

Control of which claims are mapped to which identity attributes is governed by the [`ClaimMappings`](consul-by-hashicorp-2.md#claimmappings) and [`ListClaimMappings`](consul-by-hashicorp-2.md#listclaimmappings). These are both maps of items to copy with elements of the form: `"":""`.

The only difference between these two types of mappings is that `ClaimMappings` is used to map singular values \(such as a name, department, or team\) while `ListClaimMappings` is used to map lists of values.

The singular values mapped by `ClaimMappings` can be interpolated in a binding rule, and the lists of values mapped by `ListClaimMappings` cannot.

Assume this is your config snippet:

```text
{ ...other fields...
  "ClaimMappings": {
    "givenName": "first_name",
    "surname": "last_name"
  },
  "ListClaimMappings": {
    "groups": "groups"
  }
}
```

This specifies that the values in the JWT claims `"givenName"` and `"surname"` should be copied to attributes named `"value.first_name"` and `"value.last_name"` respectively. Additionally the list of values in the JWT claim `"groups"` should be copied to an attribute named `"list.groups"`.

The following table shows the resulting attributes that will be extracted, and the ways they may be used in Rule Bindings:

| Attributes | Supported Selector Operations | Can be Interpolated |
| :--- | :--- | :--- |
| `value.first_name` | Equal, Not Equal, In, Not In, Matches, Not Matches | yes |
| `value.last_name` | Equal, Not Equal, In, Not In, Matches, Not Matches | yes |
| `list.groups` | In, Not In, Is Empty, Is Not Empty | no |

### [»](consul-by-hashicorp-2.md#claim-specifications-and-json-pointer)Claim Specifications and JSON Pointer

The [`ClaimMappings`](consul-by-hashicorp-2.md#claimmappings) and [`ListClaimMappings`](consul-by-hashicorp-2.md#listclaimmappings) fields are used to point to data within the JWT. If the desired key is at the top of level of the JWT, the name can be provided directly. If it is nested at a lower level, a JSON Pointer may be used.

Assume the following JWT claims are decoded:

```text
{
  "division": "North America",
  "groups": {
    "primary": "Engineering",
    "secondary": "Software"
  },
  "iss": "https://my-corp-app-name.auth0.com/",
  "sub": "auth0|eiw7OWoh5ieSh7ieyahC3ief0uyuraphaengae9d",
  "aud": "V1RPi2MYptMV1RPi2MYptMV1RPi2MYpt",
  "iat": 1589224148,
  "exp": 1589260148,
  "nonce": "eKiihooH3Fah8Ieshah4leeti6ien3"
}
```

A parameter of `"division"` will reference `"North America"`, as this is a top level key. A parameter `"/groups/primary"` uses JSON Pointer syntax to reference `"Engineering"` at a lower level. Any valid JSON Pointer can be used as a selector. Refer to the [JSON Pointer RFC](https://tools.ietf.org/html/rfc6901) for a full description of the syntax

