---
title: Additional ambient configurations
description: Additional configuration options for using kgateway as the ingress gateway in an Istio ambient mesh.
weight: 20
---

{{< reuse "kgw-docs/pages/integrations/istio/ambient/additional-settings.md" >}}

## Migrate from kgateway waypoints {#migrate-from-kgateway-waypoints}

Running {{< reuse "/kgw-docs/snippets/kgateway.md" >}} as an Istio ambient waypoint is no longer supported. The controller no longer reconciles GatewayClass resources named `kgateway-waypoint`, GatewayClass resources with the `ambient.istio.io/waypoint-inbound-binding` annotation, or Gateways that use those classes. Existing waypoint Deployments and Services stay in the cluster, but their status is no longer updated. A restarted waypoint pod does not get new configuration.

In this release, the controller ClusterRole still grants `get`, `list`, and `watch` on `authorizationpolicies.security.istio.io` so older pods can upgrade safely. If you maintain a custom ClusterRole, plan to remove that rule in the next minor release because {{< reuse "/kgw-docs/snippets/kgateway.md" >}} no longer uses it.

Before you upgrade, move each workload from its kgateway waypoint to an Istio waypoint.

1. Create an Istio waypoint Gateway near the old Gateway.

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: Gateway
   metadata:
     name: httpbin-istio-waypoint
     namespace: httpbin
   spec:
     gatewayClassName: istio-waypoint
     listeners:
       - name: mesh
         port: 15008
         protocol: HBONE
   ```

2. Move routes and policies that target the old waypoint Gateway to the Istio waypoint Gateway. Retarget Istio AuthorizationPolicy resources that target the old `kgateway-waypoint` GatewayClass to `istio-waypoint`. {{< reuse "/kgw-docs/snippets/kgateway.md" >}} policies do not apply to Istio waypoints.

3. Change the `istio.io/use-waypoint` label on each namespace, Service, or ServiceEntry that used the old waypoint.

   ```sh
   kubectl label namespace httpbin istio.io/use-waypoint=httpbin-istio-waypoint --overwrite
   ```

4. Verify traffic through the new waypoint by checking the waypoint access logs or running `istioctl ztunnel-config workloads`.

5. After traffic reaches the new waypoint, delete the old Gateway. When no Gateway uses the old class, delete the `kgateway-waypoint` GatewayClass.

## Route ingress traffic through Istio waypoints {#ingress-use-waypoint}

Ingress traffic from {{< reuse "/kgw-docs/snippets/kgateway.md" >}} can still route through a backend Service or ServiceEntry waypoint. The ingress-use-waypoint integration is disabled by default. To enable the integration, set the `waypoint.enabled` Helm value, or set `KGW_INGRESS_USE_WAYPOINTS=true` when you do not use Helm. The deprecated `KGW_ENABLE_WAYPOINT=true` setting still enables the integration only when `KGW_INGRESS_USE_WAYPOINTS` is omitted.

With ingress-use-waypoint enabled, matching backend traffic goes to the backend's cluster IP. ztunnel can then redirect the traffic through the attached Istio waypoint. For ServiceEntries, the Istio integration must also be enabled with `KGW_ENABLE_ISTIO_INTEGRATION=true`.

1. Enable ingress-use-waypoint in your Helm values. If you set `KGW_INGRESS_USE_WAYPOINTS` in `controller.extraEnv`, that explicit value overrides `waypoint.enabled`.

   ```yaml
   waypoint:
     enabled: true
   ```

2. Make sure that the ingress gateway's pods are in the ambient mesh.

   ```sh
   kubectl label ns {{< reuse "kgw-docs/snippets/namespace.md" >}} istio.io/dataplane-mode=ambient --overwrite
   ```

3. Attach an Istio waypoint to the backend namespace, Service, or ServiceEntry.

   ```sh
   kubectl label namespace httpbin istio.io/use-waypoint=httpbin-istio-waypoint --overwrite
   ```

4. Opt the backend into ingress-use-waypoint routing. Do not add the `istio.io/ingress-use-waypoint=true` label to a headless Service, because a headless Service has no cluster IP and requests fail with HTTP `503`.

   ```sh
   kubectl label service httpbin -n httpbin istio.io/ingress-use-waypoint=true --overwrite
   ```
