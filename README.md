# argo-rollouts-kayenta

## Prepare

The manifests in this repository assume the use of the `argo-rollouts` namespace.
```bash
k create ns argo-rollouts
k config set-context --current --namespace argo-rollouts
```

## Installing Kayenta

```bash
cd kayenta
kustomize build | kubectl apply -f -
cd -
```

You should be able to visit the Swagger API documentation.
> http://<kayenta-service-external-ip>:8090/swagger-ui.html

## Installing Istio

```bash
istioctl install --set profile=demo -f istio-overlay.yml -y
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.9/samples/addons/kiali.yaml
k label namespace argo-rollouts istio-injection=enabled
k apply -f istio-monitors.yml
```

## Installing Prometheus

```bash
k apply -f https://raw.githubusercontent.com/istio/istio/master/samples/addons/prometheus.yaml
k apply -f https://raw.githubusercontent.com/istio/istio/master/samples/addons/grafana.yaml
```

## Demo flow

### Cleanup
```bash
k delete ro --all -A
k delete -f basic-rollout.yml -f basic-service.yml -f experiment.yml -f istio-gateway.yml -f istio-rollout.yml -f istio-services.yml -f istio-virtualservice.yml
```

### Manual cutover + incremental rollout

```bash
k apply -f basic-rollout.yml
k apply -f basic-service.yml
k argo rollouts get rollout rollouts-demo --watch
k argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
k argo rollouts promote rollouts-demo
```

### Abort rollout

```bash
k argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:red
k argo rollouts abort rollouts-demo
k argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
```

### Istio

```bash
k delete ro --all -A
k apply -f istio-rollout.yml -f istio-services.yml -f istio-virtualservice.yml -f istio-gateway.yml
k argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
k describe VirtualService rollouts-demo-vsvc
k argo rollouts promote rollouts-demo
```

### Background analysis

```bash
k delete ro --all -A
k apply -f background-analysistemplate.yml -f background-rollout.yml
open -a "Google Chrome" http://$(kubectl -n istio-system get svc istio-ingressgateway -o json | jq -r .status.loadBalancer.ingress[0].ip)
kpf svc/grafana 3000 -n istio-system
open -a "Google Chrome" http://localhost:3000
k argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
k describe VirtualService rollouts-demo-vsvc
k argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:red
```

### Cleanup

```bash
k delete ro --all -A
```


