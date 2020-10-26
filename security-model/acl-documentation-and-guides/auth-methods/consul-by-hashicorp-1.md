# Consul by HashiCorp

**1.8.0+:** This feature is available in Consul versions 1.8.0 and newer.

The `jwt` auth method can be used to authenticate with Consul by providing a [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) directly. The JWT is cryptographically verified using locally-provided keys, or, if configured, an OIDC Discovery service can be used to fetch the appropriate keys.

This page assumes general knowledge of JWTs and the concepts described in the main [auth method documentation](https://www.consul.io/docs/acl/auth-methods).

Both the [`jwt`](https://www.consul.io/docs/acl/auth-methods/jwt) and the [`oidc`](https://www.consul.io/docs/acl/auth-methods/oidc) auth method types allow additional processing of the claims data in the JWT.

## [»](consul-by-hashicorp-1.md#jwt-vs-oidc-auth-methods)JWT vs OIDC Auth Methods

Since both the `oidc` and `jwt` auth methods ultimately operate on JWTs as bearer tokens, it may be confusing to know which is right for a given use case.

* **JWT**: The user or application performing the Consul login must already be in possession of a valid JWT to begin. There is no browser interaction required. This is ideal for machine-oriented headless login where an operator may have already arranged for a valid JWT to be dropped on a VM or provided to a container.
* **OIDC**: The user performing the Consul login does not have a JWT nor do they even need to know what that means. This is ideal for human-oriented interactive login where an operator or administrator may have deployed SSO widely and doesn't want to have the burden of tracking and distributing Consul ACL tokens to any authorized coworker who may need to have access to a Consul instance. Browser interaction is required. **This is only available in** [**Consul Enterprise**](https://www.hashicorp.com/products/consul/).

## [»](consul-by-hashicorp-1.md#config-parameters)Config Parameters

The following auth method [`Config`](https://www.consul.io/api/acl/auth-methods#config) parameters are required to properly configure an auth method of type `jwt`:

* [`JWTValidationPubKeys`](consul-by-hashicorp-1.md#jwtvalidationpubkeys) `(array)` - A list of PEM-encoded public keys to use to authenticate signatures locally.

  Exactly one of `JWKSURL` `JWTValidationPubKeys`, or `OIDCDiscoveryURL` is required.

* [`OIDCDiscoveryURL`](consul-by-hashicorp-1.md#oidcdiscoveryurl) `(string: "")` - The OIDC Discovery URL, without any .well-known component \(base path\).

  Exactly one of `JWKSURL` `JWTValidationPubKeys`, or `OIDCDiscoveryURL` is required.

* [`OIDCDiscoveryCACert`](consul-by-hashicorp-1.md#oidcdiscoverycacert) `(string: "")` - PEM encoded CA cert for use by the TLS client used to talk with the OIDC Discovery URL. NOTE: Every line must end with a newline \(`\n`\). If not set, system certificates are used.
* [`JWKSURL`](consul-by-hashicorp-1.md#jwksurl) `(string: "")` - The JWKS URL to use to authenticate signatures.

  Exactly one of `JWKSURL` `JWTValidationPubKeys`, or `OIDCDiscoveryURL` is required.

* [`JWKSCACert`](consul-by-hashicorp-1.md#jwkscacert) `(string: "")` - PEM encoded CA cert for use by the TLS client used to talk with the JWKS URL. NOTE: Every line must end with a newline \(`\n`\). If not set, system certificates are used.
* [`ClaimMappings`](consul-by-hashicorp-1.md#claimmappings) `(map[string]string)` - Mappings of claims \(key\) that [will be copied to a metadata field](consul-by-hashicorp-1.md#trusted-identity-attributes-via-claim-mappings) \(value\). Use this if the claim you are capturing is singular \(such as an attribute\).

  When mapped, the values can be any of a number, string, or boolean and will all be stringified when returned.

* [`ListClaimMappings`](consul-by-hashicorp-1.md#listclaimmappings) `(map[string]string)` - Mappings of claims \(key\) [will be copied to a metadata field](consul-by-hashicorp-1.md#trusted-identity-attributes-via-claim-mappings) \(value\). Use this if the claim you are capturing is list-like \(such as groups\).

  When mapped, the values in each list can be any of a number, string, or boolean and will all be stringified when returned.

* [`JWTSupportedAlgs`](consul-by-hashicorp-1.md#jwtsupportedalgs) `(array)` - JWTSupportedAlgs is a list of supported signing algorithms. Defaults to `RS256`.
* [`BoundAudiences`](consul-by-hashicorp-1.md#boundaudiences) `(array)` - List of `aud` claims that are valid for login; any match is sufficient.
* [`BoundIssuer`](consul-by-hashicorp-1.md#boundissuer) `(string: "")` - The value against which to match the `iss` claim in a JWT.
* [`ExpirationLeeway`](consul-by-hashicorp-1.md#expirationleeway) `(duration: 0s)` - Duration in seconds of leeway when validating expiration of a token to account for clock skew. Defaults to 150 \(2.5 minutes\) if set to 0 and can be disabled if set to -1.
* [`NotBeforeLeeway`](consul-by-hashicorp-1.md#notbeforeleeway) `(duration: 0s)` - Duration in seconds of leeway when validating not before values of a token to account for clock skew. Defaults to 150 \(2.5 minutes\) if set to 0 and can be disabled if set to -1.
* [`ClockSkewLeeway`](consul-by-hashicorp-1.md#clockskewleeway) `(duration: 0s)` - Duration in seconds of leeway when validating all claims to account for clock skew. Defaults to 60 \(1 minute\) if set to 0 and can be disabled if set to -1.

### [»](consul-by-hashicorp-1.md#sample-configs)Sample Configs

#### [»](consul-by-hashicorp-1.md#static-keys)Static Keys

```text
{
    ...other fields...
    "Config": {
        "BoundIssuer": "corp-issuer",
        "JWTValidationPubKeys": [
            ""
        ],
        "ClaimMappings": {
            "http://example.com/first_name": "first_name",
            "http://example.com/last_name": "last_name"
        },
        "ListClaimMappings": {
            "http://example.com/groups": "groups"
        }
    }
}
```

#### [»](consul-by-hashicorp-1.md#jwks)JWKS

```text
{
    ...other fields...
    "Config": {
        "JWKSURL": "https://my-corp-jwks-url.example.com/",
        "ClaimMappings": {
            "http://example.com/first_name": "first_name",
            "http://example.com/last_name": "last_name"
        },
        "ListClaimMappings": {
            "http://example.com/groups": "groups"
        }
    }
}
```

#### [»](consul-by-hashicorp-1.md#oidc-discovery)OIDC Discovery

```text
{
    ...other fields...
    "Config": {
        "BoundAudiences": [
            "V1RPi2MYptMV1RPi2MYptMV1RPi2MYpt"
        ],
        "OIDCDiscoveryURL": "https://my-corp-app-name.auth0.com/",
        "ClaimMappings": {
            "http://example.com/first_name": "first_name",
            "http://example.com/last_name": "last_name"
        },
        "ListClaimMappings": {
            "http://example.com/groups": "groups"
        }
    }
}
```

## [»](consul-by-hashicorp-1.md#jwt-verification)JWT Verification

JWT signatures will be verified against public keys from the issuer. This process can be done one of three ways:

* **Static Keys** - A set of public keys is stored directly in the configuration.
* **JWKS** - A JSON Web Key Set \([JWKS](https://tools.ietf.org/html/rfc7517)\) URL \(and optional certificate chain\) is configured. Keys will be fetched from this endpoint during authentication.
* **OIDC Discovery** - An OIDC Discovery URL \(and optional certificate chain\) is configured. Keys will be fetched from this URL during authentication. When OIDC Discovery is used, OIDC validation criteria \(e.g. `iss`, `aud`, etc.\) will be applied.

If multiple methods are needed, another auth method of this type may be created with a different name.

## [»](consul-by-hashicorp-1.md#trusted-identity-attributes-via-claim-mappings)Trusted Identity Attributes via Claim Mappings

Data from JWT claims can be returned from the authentication step as trusted identity attributes for use in binding rule selectors and bind name interpolation.

Control of which claims are mapped to which identity attributes is governed by the [`ClaimMappings`](consul-by-hashicorp-1.md#claimmappings) and [`ListClaimMappings`](consul-by-hashicorp-1.md#listclaimmappings). These are both maps of items to copy with elements of the form: `"":""`.

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

### [»](consul-by-hashicorp-1.md#claim-specifications-and-json-pointer)Claim Specifications and JSON Pointer

The [`ClaimMappings`](consul-by-hashicorp-1.md#claimmappings) and [`ListClaimMappings`](consul-by-hashicorp-1.md#listclaimmappings) fields are used to point to data within the JWT. If the desired key is at the top of level of the JWT, the name can be provided directly. If it is nested at a lower level, a JSON Pointer may be used.

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

