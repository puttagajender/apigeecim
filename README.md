# identity-apigee-proxy

A minimal, source-controlled Apigee API proxy bundle for the Identity Spring Boot service. It intentionally contains only reverse-proxy routing: no OAuth, API products, developer apps, consumer keys, quotas, spike arrest, mTLS, or CI/CD.

## Public API and backend mapping

| Client-facing API | Backend API |
| --- | --- |
| `GET /identity/v1/{companyNumber}/{customerNumber}` | `GET /identity/{companyNumber}/{customerNumber}` |

The TargetEndpoint URL is deliberately a non-working placeholder:

```text
https://IDENTITY_SERVICE_HOST/identity
```

Replace only `IDENTITY_SERVICE_HOST` when a real, reachable Identity service host is available. Do not commit a URL containing credentials or any secret material.

## Files

```text
identity-apigee-proxy/
├── .gitignore
├── README.md
└── apiproxy/
    ├── identity-apigee-proxy.xml
    ├── proxies/
    │   └── default.xml
    ├── targets/
    │   └── default.xml
    ├── policies/
    │   └── .gitkeep
    └── resources/
        └── .gitkeep
```

- `apiproxy/identity-apigee-proxy.xml` is the required root APIProxy descriptor. It names the proxy and lists its endpoints.
- `apiproxy/proxies/default.xml` is the **ProxyEndpoint**: the client-facing, inbound side of the proxy. It owns the public `BasePath`.
- `apiproxy/targets/default.xml` is the **TargetEndpoint**: the outbound side, which makes the HTTP request to the Identity service.
- `RouteRule` connects the ProxyEndpoint to the named TargetEndpoint. This proxy has one unconditional rule: `default` routes to `default`.
- `BasePath` is the path Apigee uses to select this proxy after the Apigee domain, here `/identity/v1`. It must be unique in the deployed Apigee environment.
- `policies/` is where future Apigee policy XML files belong (for example authentication or traffic management). It is intentionally empty now.
- `resources/` is where policy-referenced assets such as JavaScript, Java JARs, XSLT, or other resources belong. It is intentionally empty now.
- `.gitkeep` files keep the intentionally empty directories visible in Git; Apigee does not use them at runtime.

## Routing flow

For this request:

```http
GET /identity/v1/1/100123
```

1. Apigee matches `/identity/v1` in the ProxyEndpoint `BasePath`.
2. The path after the base path is the proxy path suffix: `/1/100123`.
3. The `RouteRule` sends the request to TargetEndpoint `default`.
4. The target base URL is `https://IDENTITY_SERVICE_HOST/identity`. Apigee appends the path suffix, preserving the HTTP method (and query parameters, if supplied).
5. Identity receives:

```http
GET https://IDENTITY_SERVICE_HOST/identity/1/100123
```

6. The Identity response returns through the TargetEndpoint and ProxyEndpoint to the client unchanged because this bundle attaches no policies.

## Complete XML

### `apiproxy/identity-apigee-proxy.xml`

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<APIProxy name="identity-apigee-proxy">
  <DisplayName>identity-apigee-proxy</DisplayName>
  <Description>Minimal reverse proxy for the Identity Spring Boot service.</Description>
  <ProxyEndpoints>
    <ProxyEndpoint>default</ProxyEndpoint>
  </ProxyEndpoints>
  <TargetEndpoints>
    <TargetEndpoint>default</TargetEndpoint>
  </TargetEndpoints>
</APIProxy>
```

### `apiproxy/proxies/default.xml`

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
  <PreFlow name="PreFlow"><Request/><Response/></PreFlow>
  <Flows/>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <HTTPProxyConnection>
    <BasePath>/identity/v1</BasePath>
    <Properties/>
  </HTTPProxyConnection>
  <RouteRule name="default">
    <TargetEndpoint>default</TargetEndpoint>
  </RouteRule>
</ProxyEndpoint>
```

### `apiproxy/targets/default.xml`

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="default">
  <PreFlow name="PreFlow"><Request/><Response/></PreFlow>
  <Flows/>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <HTTPTargetConnection>
    <URL>https://IDENTITY_SERVICE_HOST/identity</URL>
  </HTTPTargetConnection>
</TargetEndpoint>
```

## Local validation

There is no Apigee runtime emulator that can execute this proxy entirely locally. You can validate the XML is well-formed and inspect the bundle locally, but an end-to-end proxy request requires an Apigee environment and a reachable backend.

In PowerShell, from the repository root:

```powershell
[xml](Get-Content apiproxy/identity-apigee-proxy.xml) | Out-Null
[xml](Get-Content apiproxy/proxies/default.xml) | Out-Null
[xml](Get-Content apiproxy/targets/default.xml) | Out-Null
Compress-Archive -Path apiproxy -DestinationPath identity-apigee-proxy.zip -Force
```

Verify that `apiproxy/` is the top-level directory inside the ZIP. The generated ZIP is ignored by Git.

## Later import and deployment

1. Create the ZIP as above.
2. In Google Cloud console, open **Apigee > Proxy development > API proxies**, then import the bundle to create a revision; or use the Apigee API/CLI workflow used by your organization.
3. Deploy that revision to the intended Apigee environment.
4. Configure a real backend host before deploying, preferably through the environment-appropriate target configuration approach your team adopts later.

Apigee import creates a revision but does not make it serve traffic until you deploy it. This repository deliberately does not prescribe an environment, hostname, credential, or deployment automation.

## References

This bundle follows the current [Apigee API proxy configuration reference](https://cloud.google.com/apigee/docs/api-platform/reference/api-proxy-configuration-reference). Google documents that the target base path plus the proxy path suffix forms the backend request in its [request and response variables guide](https://cloud.google.com/apigee/docs/api-platform/fundamentals/understanding-handling-request-response-data), and specifies that an import ZIP must have `apiproxy/` at its top level in its [bundle upload guide](https://cloud.google.com/apigee/docs/api-platform/fundamentals/download-api-proxies).
