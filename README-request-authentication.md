# Istio Security

Request authentication policies specify the values needed to validate a JSON Web Token (JWT). These values include, among others, the following:

- The location of the token in the request
- The issuer or the request
- The public JSON Web Key Set (JWKS)

Istio checks the token in the request, if the token is presented, it checks against the rules in the request authentication policy, and rejects requests with invalid tokens. When requests carry no token, they are accepted by default. 

To reject requests without tokens, configure authorization rules that specify the restrictions for specific operations, for example, paths or actions.

> **Note: Exercise in this repo requires to install Istio on your Kubernetes cluster with the default configuration profile.**


## Requirements

- IKS 1.16 or later with 3 workers (b3c.4x16 or better)
- Add-on Istio disabled
- CLI Client: 
    * ibmcloud
    * kubectl
    * istioctl


## Lab Flow

During the lab, you are going to

- Step 1. Clone the Repo
- Step 2. Download and Prepare Istio Installation Package
- Step 3. Connect to IBM Cloud and IKS Cluster
- Step 4. Install Istio to IKS Cluster
    - 4.1 - Explore Istio Security
- Step 5. Peer Authentication
    - 5.1 - Setup
    - 5.2 - Make Service-to-Service Calls in Container
    - 5.3 - Is Data-in-Motion Secured? - Network Traffic Monitoring
    - 5.4 - Make Service Call to Workload httpbin
    - 5.5 - Istio Auto Mutual TLS feature
    - 5.6 Is Data-in-Motion Secured?
    - 5.7 - Easy Method Identifying Mutual TLS
    - 5.8 - Peer Authentication without Change
    - 5.9 - Enabe Mesh-Wide Istio Mutual TLS in STRICT Mode
    - 5.10 - Enable Mutual TLS for a Namespace
    - 5.11 - Enable Mutual TLS for a Workload
    - 5.12 - Enable Mutual TLS for a Port
    - 5.13 - Policy Precedence
    - 5.14 - Peer Authentication Summary
- Step 6. Request Authentication
- Step 7. Authorization





### Step 1. Setup

For convenience, expose workload `httpbin.foo` via `ingressgateway`.

1. You are still in the 1st terminal window. You should be in folder `/tmp/intro-istio-security` or your repo download folder.

1. Deploy the `Ingressgateway`.

    ```
    $ scripts/deploy-ingress-gateway.sh

    -----Deploy Ingressgateway-----
    gateway.networking.istio.io/httpbin-gateway created
    ```

1. Expose Workload `httpbin.foo` via Ingressgateway.

    ```
    $ scripts/expose-httpbin-foo-ingress-gateway.sh

    -----Expose Workload httpbin.foo via Ingressgateway-----
    virtualservice.networking.istio.io/httpbin created
    ```

1. Get Ingress IP.

    ```
    $ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    ```

1. Access service `httpbin.foo` via Ingressgateway

    ```
    $ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"

    200
    ```

1. Access another end point of the service `httpbin.foo` via Ingressgateway

    ```
    $ curl $INGRESS_HOST/ip -s -o /dev/null -w "%{http_code}\n"

    200
    ```


### Step 2. - Create a Request Authentication Policy

To explore request authentication feature, you need a valid JWT. The JWT must correspond to the JWKS endpoint you want to use for the request. See a sample `request authentication policy` below.

```
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-example"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/jwks.json"
```

Apply the policy to the scope of the workload, `ingressgateway` in this case. The namespace `istio-system` indicates the policy applies to the entire mesh.

If you provide a token in the authorization header, its implicitly default location, Istio validates the token using the public key set, and rejects requests if the bearer token is invalid. However, requests without tokens are accepted. To observe this behavior, retry the request without a token, with a bad token, and with a valid token.

1. You are still in the 1st terminal window. You should be in folder `/tmp/intro-istio-security` or your repo download folder.

1. Create a `request authentication policy`.

    ```
    $ scripts/create-basic-request-auth-policy.sh

    -----Create a Basic Request Authentication Policy-----
    requestauthentication.security.istio.io/jwt-example created
    ```

1. Verify the new `request authentication policy` was created.

    ```
    $ kubectl get requestauthentication --all-namespaces

    NAMESPACE      NAME          AGE
    istio-system   jwt-example   2m1s
    ```


### Step 3. Test Request Authentication

This section uses the test token `JWT test` and `JWKS endpoint` from the Istio code base.

1. You are still in the 1st terminal window. You should be in folder `/tmp/intro-istio-security` or your repo download folder.

1. Retrieve the test token `JWT test`.

    ```
    $ TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/demo.jwt -s)
    ```

1. Submit a service request with a valid token. Istio validates the token using the public key set. Requests with tokens are accepted.

    ```
    $ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"

    200
    ```

    > Note: the valid token is submitted as part of service request in the form of `--header "Authorization: Bearer $TOKEN"`.

1. Submit a service request with a invalid token. Istio validates the token using the public key set, and rejects requests if the bearer token is invalid.

    ```
    $ curl --header "Authorization: Bearer invalid" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"

    401
    ```

   > Note: the invalid token is submitted as part of service request in the form of `--header "Authorization: Bearer invalid"`. `invalid` is submitted as the toekn in this case.

1. Submit a service request without a token. Requests without tokens are accepted by default.

    ```
    $ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"

    200
    ```
   > Note: no `--header "Authorization: Bearer $TOKEN"` is included in the service request.


### Step 4. Deny Requests without Token

Requests without tokens are accepted by default. You run a test case in the previous section.

To reject requests without valid tokens(including not token scenario), you add an authorization policy with a rule specifying a `DENY` action for requests without request principals, shown as `notRequestPrincipals: ["*"]` in the following example. Request principals are available only when valid JWT tokens are provided. The rule therefore denies requests without valid tokens or no token at all.

Apply the policy to the scope of the workload, `ingressgateway` in this case. The namespace `istio-system` indicates the policy applies to the entire mesh.

```
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "frontend-ingress"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
```

To deny requests without valid tokens or no token at all,

1. You are still in the 1st terminal window. You should be in folder `/tmp/intro-istio-security` or your repo download folder.

1. Create an `authorization policy` to reject requests without tokens.

    ```
    $ scripts/create-authorization-policy-notoken.sh

    -----Create an Authorization Policy to Deny Requests without Token-----
    authorizationpolicy.security.istio.io/frontend-ingress created
    ```

1. Verify the new `authorization policy` was created.

    ```
    $ kubectl get authorizationpolicy --all-namespaces

    NAMESPACE      NAME               AGE
    istio-system   frontend-ingress   71s
    ```

1. Submit a service request without a token. 

    ```
    $ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"

    403
    ```

1. The request fails after you created an authorization policy to reject requests without a valid token or without a token at all. When you submitted the same request in the previous section, it was accepted and processed successfully.

1. To verify all end pointSubmit a different service request without a token. 

    ```
    $ curl $INGRESS_HOST/ip -s -o /dev/null -w "%{http_code}\n"

    403
    ```

### Step 5. Deny Requests without Token

You can refine authorization policy to mandate a token requirement for a host, path, or method. The sample .yaml file below changes the authorization policy to only require JWT on /headers. 

```
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "frontend-ingress"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
    to:
    - operation:
        paths: ["/headers"]
```

When this authorization rule takes effect, requests to $INGRESS_HOST/headers fail with the error code 403. Requests to all other paths succeed, for example $INGRESS_HOST/ip.

1. You are still in the 1st terminal window. You should be in folder `/tmp/intro-istio-security` or your repo download folder.

1. Refine the `authorization policy` to only reject requests to /headers without tokens.

    ```
    $ scripts/create-authorization-policy-notoken-headers.sh

    -----Create an Authorization Policy to only Deny Requests to /headers without Token-----
    authorizationpolicy.security.istio.io/frontend-ingress configured
    ```

1. Verify the new `authorization policy` was created.

    ```
    $ kubectl get authorizationpolicy --all-namespaces

    NAMESPACE      NAME               AGE
    istio-system   frontend-ingress   71s
    ```

1. Submit a service request to `$INGRESS_HOST/headers` without a token. The request is still rejected as the refined authorization policy mandates a token for the path `/headers`.

    ```
    $ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"

    403
    ```

1. Submit a service request to `$INGRESS_HOST/ip` without a token. The request succeeds as the refined authorization policy only mandates a token for the path `/headers`. No token requirement for `/ip`.

    ```
    $ curl $INGRESS_HOST/ip -s -o /dev/null -w "%{http_code}\n"

    200
    ```


#### Step 6. Clean Up

To clean up the environment setup for `peer authentication` and `request authentication` exercises,

1. You are still in the 1st terminal window. You should be in folder `/tmp/intro-istio-security` or your repo download folder.

1. Remove authentication policy.

    ```
    $ kubectl -n istio-system delete requestauthentication jwt-example

    requestauthentication.security.istio.io "jwt-example" deleted
    ```
    
1. Remove authorization policy.

    ```
    $ kubectl -n istio-system delete authorizationpolicy frontend-ingress

    authorizationpolicy.security.istio.io "frontend-ingress" deleted
    ```

1. Delete test namespaces.

    ```
    $ kubectl delete ns foo bar legacy

    namespace "foo" deleted
    namespace "bar" deleted
    namespace "legacy" deleted
    ```


## Related Links

- [Open Cybersecurity Alliance](https://opencybersecurityalliance.org/)
- [Istio Concepts - Security](https://istio.io/docs/concepts/security/)
- [Istio Tasks - Security](https://istio.io/docs/tasks/security/)



