# How Kubernetes Health Checks Brought Down a Payment Service

One afternoon while I was on-call, I received an alert that one of our team's Tier 2 payment services hadn't received any traffic for the past hour. I logged into the Kubernetes dashboard to investigate and immediately noticed that every pod backing the service was stuck in a continuous restart cycle. At the same time, CPU and memory utilization had spiked across every container.

To mitigate the initial impact, I contacted the platform team and looking at the memory spike they decided to provision more memory for the containers. The containers restarted successfully, and the service was restored within 15 minutes.

## The Investigation

Although it looked like the memory spike had caused the outage, something didn't add up. This service had never exhibited memory pressure before, and its traffic that day was consistent with its normal daily pattern. I reviewed the Datadog dashboards, looking for anything that could explain the sudden increase in memory usage like an unexpected traffic surge, a recent deployment, or a JVM leak but found nothing unusual. 

While increasing the memory allocation had restored the service, it felt more like a mitigation than a root-cause fix. I couldn't justify submitting a code change to permanently increase the service's memory limits in its Helm chart simply to match the temporary configuration the platform team had applied through GitOps during the incident. Before making that change, I wanted to understand why the service suddenly needed more memory in the first place.

## Looking at logs to narrow down the cause

Datadog metrics didn't help much to understand the root cause, so I started reading the service logs from an hour before the outage. I started noticing repeated downstream service health check failure logs starting roughly 30 minutes before the payment service outage.

However, the downstream failures did not appear to be causing the payment application to return errors, and the payment service itself looked healthy from an application perspective.

While investigating further, I noticed repeated Kubernetes probe failures in the logs. At the time, I did not have a deep understanding of how Kubernetes probes influenced application lifecycle behavior, so I decided to dig deeper into the different types of Kubernetes probes and how they were configured for the payment service.

## Deep Dive into Kubernetes Probes

According to the Kubernetes documentation, Kubernetes uses probes to continuously monitor the health of containers in a pod. The kubelet, an agent that runs on each node in a cluster periodically executes a probe such as an HTTP request, grPC request, a TCP check, or a command inside the container to determine its state. Based on the probe results, Kubernetes can restart unhealthy containers or stop sending traffic to containers until they are ready to serve requests again.

The kubelet can execute three types of probes each serving a different purpose.

**Startup probes:**  Startup probes verify whether the application within a container has successfully started. This probe is only executed at startup, unlike liveness and readiness probes, which the Kubelet runs periodically. If the startup probe fails, the Kubelet kills the container, and tries to restart the container.

**Liveness probes:** Liveness probes determine when to restart a container. For example, liveness probes could catch a failure, where an application is running, but unable to make progress. Kubelet tries to restart containers in such a state to help make the application more available despite the bugs.

**Readiness probes:** Readiness probes determine when a container is ready to accept traffic. These probes are also useful later in the container’s lifecycle, for example, when recovering from temporary failures. They run on the container during its whole lifecycle.

Since I can't share the actual production configuration, the following Helm configuration is an example adapted from the Kubernetes documentation. It illustrates the probe settings and behaviors relevant to this incident.

```
apiVersion: v1
kind: Pod
metadata:
  name: probe-example
spec:
  containers:
  - name: app
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5

```
While reviewing the logs, I noticed the kubelet executing the configured probes every 25 seconds, matching the probe configuration in the service's Helm chart. Since the containers were restarting roughly every minute, each restart triggered a new sequence of startup, readiness, and liveness probes. That observation became an important clue in understanding what Kubernetes was doing behind the scenes.

## Connecting the dots

The above knowledge on Kubernetes probes made it possible to correlate the two events I had been observing in the logs every minute. The application's health check was failing because it depended on the availability of a downstream service. As a result, the ```liveness probe``` also failed, causing the kubelet to restart the container. When the container came back up, Kubernetes started a new probe cycle. Since the downstream dependency was still unavailable, the liveness probe failed again, triggering yet another restart. The service had become stuck in a `CrashLoopBackOff state` not because the application itself was unhealthy, but because its liveness check was coupled to the health of a downstream dependency. That’s how the Kubernetes restart loop started making sense.

**The Restart loop**
```
Downstream service outage
          │
          ▼
Application's /health endpoint checks downstream
          │
          ▼
Liveness probe fails
          │
          ▼
Kubelet restarts container
          │
          ▼
Container starts again
          │
          ▼
Liveness checks downstream again
          │
          ▼
Restart...

```
**Why did provisioning more memory recover the service?**

At first, increasing the memory allocation appeared to have fixed the issue. However, the root cause was not memory exhaustion, it was a dependency health check failure. By the time the additional memory was provisioned by the platform team, the downstream service had already recovered. However, the impacted service containers were still struggling to restart because repeated restarts had pushed them into a resource-constrained state, causing CPU and memory pressure.

Increasing the memory and CPU limits gave the containers enough resources to complete their startup sequence successfully. Since the downstream dependency was also healthy again, all three Kubernetes probes passed, allowing the pods to initialize successfully and start receiving traffic again.

## Actions taken after the incident

**Separated application health from dependency health**

Removed all downstream dependency health checks which were not absolutely critical for the application to function.

**Monitoring dependency health through observability**

Created service-level dashboards in Datadog to monitor downstream dependency health, including latency, error rates, and traffic patterns. Configured Datadog alerts to notify PagerDuty if downstream services exhibit abnormal behavior, enabling faster detection and response during incidents.

**Documented and shared findings**

Documented the incident, including the root cause, lessons learned, and Kubernetes health check behaviors that contributed to the failure. Shared the findings across the organization to help teams avoid similar health check design pitfalls.

## Lessons learned

1. Understanding Kubernetes probes is essential to operating services in production as they directly influence how Kubernetes manages application lifecycle and recovery behavior during failures.
2. Health checks that depend on backend dependencies can cause cascading failures that can potentially lead to an outage. A downstream dependency outage can cause the application's liveness probe to fail, leading Kubernetes to restart containers repeatedly until the dependency recovers and the liveness check succeeds.

## References

- Kubernetes Documentation – Liveness, Readiness, and Startup Probes  
 https://kubernetes.io/docs/concepts/workloads/pods/probes/
