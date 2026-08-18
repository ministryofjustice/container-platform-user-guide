# Web application firewall

The Container Platform provides a Web Application Firewall (WAF) using [Coraza](https://coraza.io/) and the [OWASP Core Rule Set (CRS)](https://coreruleset.org/).

The WAF inspects HTTP requests before they reach your application and helps protect against common web attacks, such as SQL injection and cross-site scripting (XSS).

## WAF protection is enabled by default

Applications using the Container Platform Gateway are protected by Coraza in **blocking mode by default**.

You do not need to configure anything to enable the WAF.

If your `HTTPRoute` does not have an application-specific WAF policy, it uses the platform default:

- Coraza WAF enabled
- OWASP Core Rule Set enabled
- blocking mode enabled

When a request triggers a blocking WAF rule:

1. the request is not forwarded to your application
2. the client receives an HTTP `403 Forbidden` response
3. the WAF event and triggered rule are logged

## Application-specific WAF policies

You can create an `EnvoyExtensionPolicy` for an individual `HTTPRoute` if you need to change the default WAF behaviour or provide application-specific Coraza configuration.

> **Important:** An `EnvoyExtensionPolicy` attached to your `HTTPRoute` takes precedence over the default WAF policy configured on the platform Gateway.
>
> The platform default and your application-specific WAF configuration are **not merged**. When you create a WAF policy for your `HTTPRoute`, you are responsible for defining the complete Coraza configuration for that route.

For example:

```text
Platform Gateway
│
├── Default Coraza + OWASP CRS
│
├── HTTPRoute A
│   └── no application policy
│       └── uses platform default WAF
│
└── HTTPRoute B
    └── EnvoyExtensionPolicy
        └── application policy takes precedence
```

If you want to retain the standard OWASP CRS protection while adding your own configuration, your policy must include the standard Coraza and OWASP CRS configuration:

```yaml
directives:
  - Include @coraza.conf
  - SecRuleEngine On
  - Include @crs-setup.conf
  - Include @owasp_crs/*.conf

  # Add application-specific configuration below
```

For example, creating a policy containing only:

```yaml
directives:
  - Include @coraza.conf
  - SecRuleEngine On
```

does **not** add these directives to the platform default.

This becomes the WAF configuration for your `HTTPRoute`, and the OWASP CRS rules are not loaded unless you include them.

## WAF modes

You can configure the WAF behaviour for an individual `HTTPRoute`.

| Mode | Behaviour |
|---|---|
| `On` | Detects and blocks malicious requests. This is the platform default. |
| `DetectionOnly` | Detects and logs rule matches but forwards requests to your application. |
| `Off` | Disables WAF rule processing for the route. |

Changing the WAF configuration for your `HTTPRoute` does not change the WAF configuration of other applications using the shared Gateway.

## Use detection-only mode

Detection-only mode is useful when you want to identify requests that trigger WAF rules without blocking them.

For example, you can use detection-only mode while investigating false positives or testing application-specific WAF configuration.

Create an `EnvoyExtensionPolicy` in the same namespace as your `HTTPRoute`:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyExtensionPolicy
metadata:
  name: my-application-waf
  namespace: my-namespace
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: my-application
  dynamicModule:
    - name: composer
      filterName: coraza-waf
      config:
        directives:
          - Include @coraza.conf
          - SecRuleEngine DetectionOnly
          - Include @crs-setup.conf
          - Include @owasp_crs/*.conf
```

Replace:

- `my-namespace` with your application namespace
- `my-application` with the name of your `HTTPRoute`

Apply the configuration:

```shell
kubectl apply -f waf-policy.yaml
```

This route-level policy takes precedence over the platform Gateway WAF policy.

The configuration above explicitly loads the OWASP Core Rule Set but changes `SecRuleEngine` from the platform default of `On` to `DetectionOnly`.

Requests that trigger WAF rules are logged but continue to your application.

Other `HTTPRoute` resources continue to use their own effective WAF configuration and are not changed by this policy.

## Use blocking mode

Blocking mode is enabled by default, so you do not normally need to create an `EnvoyExtensionPolicy`.

If you have an application-specific WAF configuration and want it to operate in blocking mode, configure:

```yaml
config:
  directives:
    - Include @coraza.conf
    - SecRuleEngine On
    - Include @crs-setup.conf
    - Include @owasp_crs/*.conf
```

Requests that trigger blocking rules receive an HTTP `403 Forbidden` response and are not forwarded to your application.

## Add application-specific WAF rules

You can add application-specific Coraza rules to your `EnvoyExtensionPolicy`.

If you want to retain the platform's standard protection, start your route-level policy with the standard Coraza and OWASP CRS configuration:

```yaml
config:
  directives:
    - Include @coraza.conf
    - SecRuleEngine On
    - Include @crs-setup.conf
    - Include @owasp_crs/*.conf

    # Add application-specific rules below
```

For example, if you need to exclude a specific OWASP CRS rule:

```yaml
config:
  directives:
    - Include @coraza.conf
    - SecRuleEngine On
    - Include @crs-setup.conf
    - Include @owasp_crs/*.conf
    - SecRuleRemoveById 942100
```

Remember that this is the complete WAF configuration for your `HTTPRoute`. It is not appended to the platform default.

Removing an OWASP CRS rule reduces the protection provided by the WAF. Where possible, make exclusions as specific as possible rather than disabling a rule for your entire application.

We recommend testing application-specific changes using `DetectionOnly` before enabling blocking mode.

For more information about writing Coraza rules, see the [Coraza documentation](https://coraza.io/docs/).

## Disable the WAF

You can disable WAF rule processing for an individual `HTTPRoute`.

Create an application-specific policy:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyExtensionPolicy
metadata:
  name: my-application-waf
  namespace: my-namespace
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: my-application
  dynamicModule:
    - name: composer
      filterName: coraza-waf
      config:
        directives:
          - Include @coraza.conf
          - SecRuleEngine Off
```

Because the route-level `EnvoyExtensionPolicy` takes precedence over the platform Gateway policy, `SecRuleEngine Off` disables WAF rule processing for this `HTTPRoute`.

Disabling the WAF removes the application-layer protection provided by the platform WAF.

Consider using `DetectionOnly` instead if you are troubleshooting false positives.

## Invalid WAF configuration

The Container Platform validates application-specific Coraza configuration before it is applied to Envoy.

If your `EnvoyExtensionPolicy` contains invalid Coraza syntax, Kubernetes rejects the change.

For example, an invalid configuration may return:

```text
Error from server:
admission webhook "coraza-validator..."
denied the request:

invalid Coraza configuration:
failed to compile directive "secaction"
```

The invalid `EnvoyExtensionPolicy` is not applied and does not change the WAF configuration used by your application.

Correct the configuration and apply your policy again.

## View WAF events

Coraza records WAF events when requests trigger rules.

In blocking mode:

```text
request
  │
  ▼
WAF rule triggered
  │
  ├── event logged
  │
  └── request blocked
          │
          ▼
       HTTP 403
```

In detection-only mode:

```text
request
  │
  ▼
WAF rule triggered
  │
  ├── event logged
  │
  └── request forwarded
          │
          ▼
      application
```

WAF events include the triggered rule ID. You can use this ID to identify the OWASP CRS rule when investigating false positives.

> **Note:** Add Container Platform log-search instructions here once the WAF logging integration is finalised.

## Investigate false positives

A false positive occurs when the WAF identifies legitimate application traffic as malicious.

If you believe a WAF rule is blocking legitimate traffic:

1. Configure your `HTTPRoute` to use `DetectionOnly`.
2. Reproduce representative application traffic.
3. Review the WAF events.
4. Identify the rule ID causing the match.
5. Determine why the legitimate request triggered the rule.
6. Add an application-specific exclusion if required.
7. Test the change.
8. Change `SecRuleEngine` back to `On`.

Avoid disabling the entire WAF when a specific rule exclusion can resolve the issue.

## Test your WAF

You can test that the WAF is operating by sending a request designed to trigger an OWASP CRS rule.

For example:

```shell
curl -i "https://<your-hostname>/?id=1%27%20OR%20%271%27=%271"
```

In blocking mode, you should receive an HTTP `403 Forbidden` response.

In detection-only mode, the request should reach your application and the matching WAF event should be logged.

## Migrating from ModSecurity

The previous Cloud Platform uses ModSecurity with ingress-nginx.

The Container Platform uses Coraza with Envoy Gateway.

| Cloud Platform | Container Platform |
|---|---|
| ingress-nginx | Envoy Gateway |
| `Ingress` | `HTTPRoute` |
| ModSecurity | Coraza |
| Ingress annotations | `EnvoyExtensionPolicy` |
| OWASP Core Rule Set | OWASP Core Rule Set |

Do not copy ModSecurity ingress annotations into your Container Platform configuration.

Coraza supports the ModSecurity SecLang rule language, so many WAF concepts and rule directives will be familiar when migrating application-specific configuration.

## Further information

For more information, see:

- [Coraza documentation](https://coraza.io/docs/)
- [OWASP Core Rule Set documentation](https://coreruleset.org/docs/)
- [Kubernetes Gateway API documentation](https://gateway-api.sigs.k8s.io/)