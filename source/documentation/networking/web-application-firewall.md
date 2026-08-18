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

> **Note:** Validation of Coraza configuration before it is applied is planned but is not currently available.

Take care when adding custom Coraza directives to your `EnvoyExtensionPolicy`.

Kubernetes and Envoy Gateway do not validate the Coraza rule syntax when the `EnvoyExtensionPolicy` is created. This means Kubernetes may successfully accept an `EnvoyExtensionPolicy` containing invalid Coraza configuration.

For example:

```yaml
config:
  directives:
    - Include @coraza.conf
    - SecRuleEngine On
    - SecAction "id:1002,phase:1,pass,setvar:ip.requests=+1,expirevar:ip.requests=60"
```

The policy may be successfully created:

```shell
kubectl apply -f waf-policy.yaml
```

However, when Envoy receives the updated configuration, the Coraza dynamic module attempts to compile the WAF configuration.

If the configuration is invalid, Coraza cannot create the WAF and Envoy rejects the updated listener configuration.

You may see errors similar to the following in the Envoy logs:

```text
[warning][dynamic_modules] Failed to load configuration:
failed to create WAF from directives:
invalid WAF config from string:
failed to compile the directive "secaction":
invalid arguments, expected collection TX

[warning][config] delta config for
type.googleapis.com/envoy.config.listener.v3.Listener rejected:
Failed to create filter config:
Failed to initialize dynamic module
```

Envoy Gateway may also report that Envoy rejected the configuration update:

```text
Envoy rejected the last update with code 13:
Error adding/updating listener(s):
Failed to create filter config:
Failed to initialize dynamic module
```

When Envoy rejects the new listener configuration, it continues using the last successfully accepted configuration rather than applying the invalid update.

Because applications share platform Gateway infrastructure, an invalid application-specific WAF configuration can cause an update to the associated shared listener to be rejected.

This can prevent other configuration changes for that listener from becoming active until the invalid WAF configuration is corrected or removed.

If you apply an invalid WAF configuration:

1. check the Envoy logs for `Failed to create WAF from directives`
2. identify and correct the invalid Coraza directive
3. apply the corrected `EnvoyExtensionPolicy`
4. confirm that Envoy accepts the new listener configuration

We recommend testing custom Coraza configuration before applying it to production environments.

> **Planned improvement:** Container Platform will validate application-specific Coraza configuration during Kubernetes admission. Invalid Coraza configuration will then be rejected before it reaches Envoy Gateway, preventing an invalid WAF policy from affecting the shared listener configuration.

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