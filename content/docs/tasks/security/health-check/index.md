---
title: Citadel health checking
description: Shows how to enable Citadel health checking with Kubernetes.
weight: 70
keywords: [security,health-check]
---

This task shows how to enable Kubernetes health checking for Citadel. Note this is an Alpha feature.

Since Istio 0.6, Citadel has a health checking feature that can be optionally enabled.
By default, the normal Istio deployment process does not enable this feature.
Currently, the health checking feature is able to detect the failures of Citadel CSR signing service,
by periodically sending CSRs to the API. More health checking features are coming shortly.

Citadel contains a _prober client_ module that periodically checks Citadel's status (currently only the health
status of the gRPC server).
If Citadel is healthy, the _prober client_ updates the _modification time_ of the _health status file_
(the file is always empty). Otherwise, it does nothing. Citadel relies on a
[K8s liveness and readiness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
with command line to check the _modification time_ of the _health status file_ on the pod.
If the file is not updated for a period, the probe will be triggered and Kubelet will restart the Citadel container.

Note: because Citadel health checking currently only monitors the health status of CSR service API,
this feature is not needed if the production setup is not using the
[Istio Mesh Expansion](/docs/setup/kubernetes/mesh-expansion/) (which requires the CSR service API).

## Before you begin

* Set up Istio by following the instructions in the
  [quick start](/docs/setup/kubernetes/quick-start/) with global mutual TLS enabled.
  It is recommended to install using [Helm](/docs/setup/kubernetes/helm-install/) with the Helm arguments `global.mtls.enabled` and `security.healthCheckEnabled` set as `true`.

> Starting with Istio 0.7, you can use [authentication policy](/docs/concepts/security/#anatomy-of-an-authentication-policy) to configure mutual TLS for all/selected services in a namespace (repeated for all namespaces to get global setting). See [authentication policy task](/docs/tasks/security/authn-policy/)

## Deploying Citadel with health checking

Deploy Citadel with health checking enabled by setting the Helm argument `security.healthCheckEnabled` as `true`.

Deploy the `istio-citadel` service so that the CSR service can be found by the health checker.

{{< text bash >}}
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Service
metadata:
  name: istio-citadel
  namespace: istio-system
  labels:
    istio: citadel
spec:
  ports:
    - port: 8060
  selector:
    istio: citadel
EOF
{{< /text >}}

## Verifying the health checker is working

Citadel will log the health checking results. Run the following in command line:

{{< text bash >}}
$ kubectl logs `kubectl get po -n istio-system | grep istio-citadel | awk '{print $1}'` -n istio-system
{{< /text >}}

You will see the output similar to:

{{< text plain >}}
...
2018-02-27T04:29:56.128081Z     info    CSR successfully signed.
...
2018-02-27T04:30:11.081791Z     info    CSR successfully signed.
...
2018-02-27T04:30:25.485315Z     info    CSR successfully signed.
...
{{< /text >}}

The log above indicates the periodic health checking is working.
Observe that the health checking interval is about 15 seconds, which is the default health checking interval.

## (Optional) Configuring the health checking

Optionally, adjust the health checking configuration to meet your own needs. Open the `istio.yaml` generated via [helm template](/docs/setup/kubernetes/helm-install/#option-1-install-with-helm-via-helm-template), and locate the following lines.

{{< text plain >}}
...
  - --liveness-probe-path=/tmp/ca.liveness # path to the liveness health checking status file
  - --liveness-probe-interval=60s # interval for health checking file update
  - --probe-check-interval=15s    # interval for health status check
livenessProbe:
  exec:
    command:
    - /usr/local/bin/istio_ca
    - probe
    - --probe-path=/tmp/ca.liveness # path to the liveness health checking status file
    - --interval=125s               # the maximum time gap allowed between the file mtime and the current sys clock.
  initialDelaySeconds: 60
  periodSeconds: 60
...
{{< /text >}}

The `liveness-probe-path` and `probe-path` are the path to the health status file, configured at Citadel and the
prober;
the `liveness-probe-interval` is the interval to update the health status file, if Citadel is healthy;
the `probe-check-interval` is the interval for Citadel health checking.
The `interval` is the maximum time elapsed since the last update of the health status file, for the prober to consider
Citadel as healthy.
`initialDelaySeconds` and `periodSeconds` are the initial delay and the probe running period.

Prolonging `probe-check-interval` will reduce the health checking overhead, but there will be a greater lagging for the
prober to get notified on the unhealthy status.
To avoid the prober restarting Citadel due to temporary unavailability, the `interval` on the prober can be
configured to be more than `N` times of the `liveness-probe-interval`. This will allow the prober to tolerate `N-1`
continuously failed health checks.

## Cleanup

*   To disable health checking on Citadel, deploy Citadel with health checking disabled by setting the Helm argument `security.healthCheckEnabled` as `false`.

*   To remove Citadel deployment and service:

    {{< text bash >}}
    $ kubectl delete deploy istio-citadel -n istio-system
    $ kubectl delete svc istio-citadel -n istio-system
    {{< /text >}}
