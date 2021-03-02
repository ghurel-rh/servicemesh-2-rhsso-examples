# Approach 2

## Cleaning from approach 1
If you come from the approach 1, you first need to delete `RequestAuthentication` object and `AuthorizationPolicy` object from `bookinfo`:
```
$ oc delete requestauthentication,authorizationpolicy -n bookinfo --all
requestauthentication.security.istio.io "jwt-rhsso" deleted
authorizationpolicy.security.istio.io "authpolicy" deleted

# Wait some seconds for Istio to update its config

# Test free access to bookinfo:
$ export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
$ curl -I http://$GATEWAY_URL/productpage
HTTP/1.1 200 OK
```

## Injecting oauth2-proxy container inside the Istio Ingress Gateway pod
First, ensure that `<CLUSTERNAME>` and `<BASEDOMAIN>` have been replaced with the appropriate values in `01_patch-istio-ingressgateway-deploy.yaml` (this has been done during the prerequisites phase).

Also, ensure the argument `--client-secret` inside the YAML file matches the secret of the istio client in RHSSO (see prerequisites).
```
# Check the client secret
$ cat 01_patch-istio-ingressgateway-deploy.yaml | grep 'client-secret'
```

Then, patch the Istio Ingress Gateway deployment:
```
$ oc patch deploy istio-ingressgateway -n istio-system --patch "$(cat 01_patch-istio-ingressgateway-deploy.yaml)"

# Check the istio-ingressgateway pod has now 2 containers:
$ oc get pod -n istio-system
[...]
istio-ingressgateway-946644c7f-7knb7   2/2     Running   0          48s
[...]
```

## Redirect Istio Ingress Gateway route to the oauth2-proxy container
In order to enforce authentication, the default ingress gateway route must now target the oauth2-proxy container instead of the istio-proxy container (which is the original container of the Istio Ingress Gateway pod). Once authenticated, the oauth2-proxy container will forward the request locally to the istio-proxy container since they are in the same pod.

```
# Add the oauth2-proxy port to the Istio Ingress Gateway service
$ oc patch svc istio-ingressgateway -n istio-system --patch "$(cat 02_patch-istio-ingressgateway-svc.yaml)" 
service/istio-ingressgateway patched

# Set edge TLS termination for the Istio Ingress Gateway route
# This is needed since RHSSO is configured by default to use HTTPS only.
$ oc patch route istio-ingressgateway -n istio-system --patch "$(cat 03_patch-istio-ingressgateway-route-edge.yaml)" 
route.route.openshift.io/istio-ingressgateway patched

# Make the Istio Ingress Gateway route target the oauth2-proxy container
$ oc patch route istio-ingressgateway -n istio-system --patch "$(cat 04_patch-istio-ingressgateway-route-oauth.yaml)" 
route.route.openshift.io/istio-ingressgateway patched
```

## Test the OIDC redirection workflow to access bookinfo
In a browser, in a private navigation mode, open https://istio-ingressgateway-istio-system.apps.<CLUSTERNAME>.<BASEDOMAIN>/productpage .
You are redirected to our RHSSO, and if you authenticate using `localuser:localuser` you are then successfully redirected to the `bookinfo` application.
