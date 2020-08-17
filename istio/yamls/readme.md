## To enable mTLS between example nginx App (server) and client using curl
//Manual istio-proxy inject, without mutating webhook//
//Assuming you have istio running in istio-system namespace

1. kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}' > inject/inject-config.yaml
2. kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}' > inject/inject-values.yaml
3. kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > inject/mesh-config.yaml
-----------------------------------------------------------------------------

Deploy nginx app using Vault for PKI and Istio for mTLS

1. hcl and nginx configmaps

2. nginx service

3. nginx deployment (example-k8s-spec.yaml) using below command

:Inject envoy proxy manually
```hcl
istioctl kube-inject \
--injectConfigFile inject/inject-config.yaml \
--meshConfigFile inject/mesh-config.yaml \
--valuesFile inject/inject-values.yaml \
--filename example-k8s-spec.yaml \
| kubectl apply -f -

```

You should get the server.crt and key at /usr/share/nginx/html/ in nginx pod (fetched from Vault)

4. We will use istio ingress gateway to get the traffic inside k8s nginx pod
- create a nginx istio gateway (ingress-gateway-https.yaml) in PASSTHROUGH mode (works like HAproxy, and we can rencrypt traffic by adding gateway certs here)

- create a nginx istio virtual service (nginx-virtualservice.yaml), to map istio gateway with k8s nginx service

- port-forward your laptop port to istio-ingress service

kubectl port-forward service/istio-ingressgateway -n istio-system 31308:443 (31308 is https port of ingress)

5. Add PeerAuthentication for strict mTLS traffic (nginx-peerauth.yaml)

6. Open kiali dashboard to see the traffic and mTLS flow

kubectl port-forward pod/kiali-d45468dc4-42shk -n ist-system 8001:20001

7. Create client certs from vault-0 pod doing exec (or use any existing client cert)

cd /home/vault

```hcl
vault write -field certificate intca/issue/example \
    common_name=client-1.clients.alice.com alt_names=localhost \
    format=pem_bundle > client-1.pem
```

copy client-1.pem to your local laptop and run below curl to test mTLS

8. Test

curl https://client-4.clients.alice.com:31308 -vvv --cacert ca.crt --cert client.crt --key client.key    


Kiali dashboard

[![Screen-Shot-2020-08-11-at-14-49-59.png](https://i.postimg.cc/Z5ZZZGjD/Screen-Shot-2020-08-11-at-14-49-59.png)](https://postimg.cc/6yHFf1kd)


------------------
https://learn.hashicorp.com/tutorials/vault/kubernetes-cert-manager
- Install Cert-Manager for adding Certificates as k8s secrets and configure Vault as issuer, this is needed because ingressgateway cant dynamically fetch certs directly from Vault for each gateway yaml, and also istio gateway yaml expects a k8s secret for SIMPLE / MUTUAL tls

a) use Helm for cert-manager
b) configure vault issuer (not clusterissuer) in istio-system namespace (see - )

:Verify issuer:

kubectl get issuers vault-issuer-istio -n istio-system -o wide

c) create ingressgateway certificate yaml (see - )  

On Vault add following in policy and service-account association in issuer (Permissions)

```hcl
path "intca*"                { capabilities = ["read", "list"] }
path "intca/roles/example"   { capabilities = ["create", "update"] }
path "intca/sign/example"    { capabilities = ["create", "update"] }
path "intca/issue/example"   { capabilities = ["create", "update", "read", "list"] }
```

```hcl
vault write auth/kubernetes/role/example \
bound_service_account_names=vault,istio-ingressgateway-service-account \
bound_service_account_namespaces=development,istio-system \
policies=myapp-kv-ro \
ttl=24h
```

https://software.danielwatrous.com/istio-ingress-vs-kubernetes-ingress/

https://www.openshift.com/blog/self-serviced-end-to-end-encryption-approaches-for-applications-deployed-in-openshift

https://developers.redhat.com/blog/2017/11/22/dynamically-creating-java-keystores-openshift/

https://itnext.io/adding-security-layers-to-your-app-on-openshift-part-6-pki-as-a-service-with-vault-and-cert-e6dbbe7028c7

https://github.com/lbroudoux/secured-fruits-catalog-k8s/blob/master/k8s/mongodb-istio-virtualservice.yml

https://archive.istio.io/v1.5/pt-br/docs/tasks/traffic-management/ingress/secure-ingress-mount/


9. TODO - Try Rencrypt Mutual TLS (instead of PASSTHROUGH) between client and ingressgateway by using vault to issue server certs

Gateway Server certs
```hcl
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  namespace: istio-system
  name: nginx-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 443
      name: https
      protocol: https
    tls:
      mode: MUTUAL # enables HTTPS on this port
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
      caCertificates: /etc/istio/ingressgateway-ca-certs/ca-chain.crt
    hosts:
    - "*"

```

Gateway Client certs
```hcl
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: nginx-mtls
  namespace: development
spec:
  host: nginx-service.development.svc.cluster.local
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
      caCertificates: /etc/istio/ingressgateway-ca-certs/ca-chain.crt
    portLevelSettings:
    - port:
        number: 8000
      loadBalancer:
        simple: ROUND_ROBIN
```

This will terminate TLS at gateway and rencrypt with client certs to talk to backend service over tls (need DestinationRule for backend service)

```hcl
outside-client > ingress-Gateway (server and client certs) > Backend K8S service
            (Terminate)                    (Rencrypt using client certs)
```

For above we can directly use Vault agent (without cert-manager), but that will require to add Vault-agent as init/sidecar to istio ingress-gateway (sample istio-ingress.yaml)


-- Below will use oAuth Bearer Token --

10. TODO - RequestAuthentication on ingressgateway by introducing JWT token service via oAuth / OIDC using Microsoft AD server  

11. TODO - RequestAuthentication on nginx app by introducing JWT token service via oAuth / OIDC using Microsoft AD server
