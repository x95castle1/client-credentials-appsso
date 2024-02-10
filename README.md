# client-credentials-appsso-resource-server

This is a sample springboot resource server application that requires a bearer token from a TAP AuthServer via the `client_credentials` grant_type to access secured endpoints within the app.

## Getting Started

A workload.yaml sample has been provided. You can apply this in your TAP developer namespace to run the sample application.

* Note: The service claim for `tap-ca` is only needed if you are running in an environment with selfsigned or custom certificates for exposed endpoints. The service claim will package those custom certificates into the image to avoid X509 fun. 

### Application Changes

You will need to update your issuer-uri to point to the AuthServer endpoint in your `application.yml` or allow the spring cloud bindings to override via a ServiceBinding.

```Yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://xxxx.com
```

### TAP Workload

```Yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  labels:
    app.kubernetes.io/part-of: resource-server-client-credentials
    apps.tanzu.vmware.com/workload-type: web
  name: resource-server-client-credentials
  namespace: dev
spec:
  params:
    - name: annotations
      value:
        autoscaling.knative.dev/minScale: "1"
  build:
    env:
    - name: BP_JVM_VERSION
      value: "17"
  serviceClaims:
  - name: ca-cert
    ref:
      apiVersion: v1 
      kind: Secret
      name: tap-ca 
  # This will allow us to inject the issuer-uri into the application via a servicebinding.
  # This requires a using a BindingsPropertiesProcessor to inject the property at start time.
  - name: appsso-demo-client-registration
    ref:
      apiVersion: services.apps.tanzu.vmware.com/v1alpha1
      kind: ResourceClaim
      name: appsso-demo-client-registration
  source:
    git:
      ref:
        branch: main
      url: https://github.com/x95castle1/client-credentials-appsso-resource-server
```

### OAuth2BindingsPropertiesProcessor

[Spring Cloud Bindings](https://github.com/spring-cloud/spring-cloud-bindings?tab=readme-ov-file#spring-security-oauth2) will not detect the `spring.security.oauth2.resourceserver.jwt.issuer-uri` property and override the value of the property via the mount servicebinding (Unfortunately, this property is not part of Spring Cloud Bindings). However, you can use a `BindingsPropertiesProcessor` to look up the property in any mounted servicebindings. This example uses this method via the `OAuth2BindingsPropertiesProcessor` class. If a ResourceClaim in a Workload is made against a secret with the type of `servicebinding.io/oauth2` then `OAuth2BindingsPropertiesProcessor` will look up the `issuer-uri` property and inject the value into the `spring.security.oauth2.resourceserver.jwt.issuer-uri` property.

### Endpoints

There are 5 endpoints to test with:

* `/api/public` - This endpoint has no security and is open to the public.
* `/api/private` - You must be authenticated against the AuthServer to access.
* `/api/private-read` - You must be authenticated against the AuthServer with a scope of `developer.read`
* `/api/private-write` - You must be authenticated against the AuthServer with a scope of `developer.write`
* `/api/private-admin` - You must be authenticated against the AuthServer with a scope of `developer.admin`

## Retrieve a Bearer Token to Test

First verify that your AuthServer is reachable by checking the `.well-known/openid-configuration` end point. 

```Bash
curl -k -X GET https://<authserver-URL>/.well-known/openid-configuration | jq 
```

The SSO Client that calls the resource-server will be setup on TAP with a ClientRegistration using `client_credentials` grant_type. The next two commands will find the ClientID and ClientSecret generated for the Service to Service `client_credentials` grant_type. These will be used to grab the Access Token from the AuthServer with the requested scopes.

```Bash
export APPSSO_TEST_CLIENT_ID=$(kubectl get secret client-credentials-client-registration -n dev -o jsonpath="{.data['client-id']}" | base64 --decode)

export APPSSO_TEST_CLIENT_SECRET=$(kubectl get secret client-credentials-client-registration -n dev -o jsonpath="{.data['client-secret']}" | base64 --decode)
```

The AuthServer can be queried against the token-uri endpoint to then retrieve an access token. Make sure to request the scopes needed for each endpoint. 

```Bash
curl -k \
 --request POST \
 --location "https://xxx/oauth2/token" \
 --header "Content-Type: application/x-www-form-urlencoded" \
 --header "Accept: application/json" \
 --data "grant_type=client_credentials" \
 --data "scope=developer.read" \
 --basic \
 --user $APPSSO_TEST_CLIENT_ID:$APPSSO_TEST_CLIENT_SECRET | jq

```

Sample Output

```Bash
{
  "access_token": "eyJraWQiOiJhdXRoMC1hdXRoc2VydmVyLXNpZ25pbmcta2V5IiwiYWxnIjoiUlMyNTYifQ.eyJzdWI.........",
  "scope": "developer.write developer.read",
  "token_type": "Bearer",
  "expires_in": 43199
}
```

The `access_token` can then be passed in the `Authorization` Header of  the request to the resource server to access the authenticated endpoint with the proper scope.