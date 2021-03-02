# Approach 3

We assume we start from a clean install of both Service Mesh and `bookinfo` application.

## Injecting oauth2-proxy container inside the Istio Ingress Gateway pod
First, replace `<CLUSTERNAME>` and `<BASEDOMAIN>` with the appropriate values in `01_patch-istio-ingressgateway-deploy.yaml`.
Then, patch the Istio Ingress Gateway deployment:
```
$ oc patch deploy istio-ingressgateway -n istio-system --patch "$(cat 01_patch-istio-ingressgateway-deploy.yaml)"

# Check the istio-ingressgateway pod has now 2 containers:
$ oc get pod -n istio-system
[...]
istio-ingressgateway-946644c7f-7knb7   2/2     Running   0          48s
[...]
```

Compared to approach 2, a new flag (`--pass-access-token=true`) has been set to the `oauth2-proxy` command-line in order to pass the JWT token to the upstream (`istio-proxy` container, which will in turn transmit that token to `istiod` for verification if required).

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

## Mount Openshift ingress routers CA certificate in istiod
When deploying RequestAuthentication object, Istiod will be in charge of verifying the connection to RHSSO. This step is performed during initialization only (RequestAuthentication object creation). Since RHSSO is exposed behind a route, we need to add the Openshift ingress routers CA certificate to istiod so it can verify the TLS certificate for the RHSSO route.

```
# Retrieve the CA certificate from secret in openshift-ingress-operator project 
$ oc extract secret/router-ca -n openshift-ingress-operator --to=/tmp/

# Create a secret from thie CA certificate in istio-system project
$ oc create secret generic openshift-wildcard --from-file=extra.pem=/tmp/tls.crt -n istio-system

# Mount the CA secret at the specific location '/cacerts/extra.pem' in istiod pod
$ oc set volumes -n istio-system deployment/istiod-basic --add  --name=extracacerts  --mount-path=/cacerts  --secret-name=openshift-wildcard  --containers=discovery
```

To check if the procedure has been done successfully:
```
$ oc project istio-system

$ oc get pod
[...]
istiod-basic-6f5bfd89bf-v49vf          1/1     Running   0          14s
[...]

# RSH to istiod pod
$ oc rsh istiod-basic-6f5bfd89bf-v49vf 

# Check connection to RHSSO without the CA
[pod] sh-4.4$ curl -I https://keycloak-rhsso.apps.<CLUSTERNAME>.<BASEDOMAIN>/auth/
curl: (60) SSL certificate problem: self signed certificate in certificate chain

# Check connection to RHSSO with the CA
[pod] sh-4.4$ curl --cacert /cacerts/extra.pem -I https://keycloak-rhsso.apps.<CLUSTERNAME>.<BASEDOMAIN>/auth/
HTTP/1.1 200 OK
```

## Create RequestAuthentication and AuthorizationPolicy for bookinfo

First, replace `<CLUSTERNAME>` and `<BASEDOMAIN>` with the appropriate values. Then apply the two CR:
```
$ oc apply -f 01_requestauthentication.yaml -n bookinfo
$ oc apply -f 02_authpolicy_allow_from_servicemesh-lab_realm.yaml -n bookinfo 
```

The created `RequestAuthentication` object tells Istio that only JWT tokens issued by our RHSSO will be used to authenticate/authorize user requests for `bookinfo`.

Compared to approach 1, the `RequestAuthentication` also tells Istio to look for the JWT tokens in the `x-forwarded-access-token` HTTP header.

The created `AuthenticationPolicy` object tells that the JWT token, to be valid, must be issued by our RHSSO and for any user with email address matching `shadowman@redhat.com` or `another@email.com`. During prerequisites phase, we created a user `localuser` inside RHSSO, and this user has the address `shadowman@redhat.com`.


## Test the OIDC redirection workflow to access bookinfo
In a browser, open https://istio-ingressgateway-istio-system.apps.<CLUSTERNAME>.<BASEDOMAIN>/productpage .
You are redirected to our RHSSO, and if you authenticate using `localuser:localuser` you are then successfully redirected to the `bookinfo` application.

Now, if you remove the `shadowman@redhat.com` line from the AuthorizationPolicy object (using `oc edit authorizationpolicy authpolicy -n bookinfo`) and try again to access https://istio-ingressgateway-istio-system.apps.<CLUSTERNAME>.<BASEDOMAIN>/productpage with `localuser`, access will be denied.


