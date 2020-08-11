## To enable mTLS between example nginx App and Mongo
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

istioctl kube-inject \
--injectConfigFile inject/inject-config.yaml \
--meshConfigFile inject/mesh-config.yaml \
--valuesFile inject/inject-values.yaml \
--filename example-k8s-spec.yaml \
| kubectl apply -f -

You should get the server.crt and key at /usr/share/nginx/html/ in nginx pod (fetched from Vault)

4. We will use istio ingress gateway to get the traffic inside k8s nginx pod
- create a nginx istio gateway (ingress-gateway-https.yaml) in PASSTHROUGH mode
- create a nginx istio virtual service (nginx-virtualservice.yaml), to map istio gateway with k8s nginx service
- port-forward your laptop port to istio-ingress service
kubectl port-forward service/istio-ingressgateway -n istio-system 31308:443 (31308 is https port of ingress)

5. Add PeerAuthentication for strict mTLS traffic (nginx-peerauth.yaml)

6. Open kiali dashboard to see the traffic and mTLS flow
kubectl port-forward pod/kiali-d45468dc4-42shk -n ist-system 8001:20001

7. Create client certs from vault-0 pod doing exec (or use any existing client cert)
cd /home/vault
vault write -field certificate intca/issue/example \
    common_name=client-1.clients.alice.com alt_names=localhost \
    format=pem_bundle > client-1.pem

copy client-1.pem to your local laptop and run below curl to test mTLS

8. Test
curl https://client-4.clients.alice.com:31308 -vvv --cacert ca.crt --cert client.crt --key client.key    


Kiali dashboard

[![Screen-Shot-2020-08-11-at-14-49-59.png](https://i.postimg.cc/Z5ZZZGjD/Screen-Shot-2020-08-11-at-14-49-59.png)](https://postimg.cc/6yHFf1kd)
