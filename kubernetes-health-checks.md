# How Kubernetes Health Checks Brought Down a Payment Service

One afternoon while I was on-call, I received an alert that one of our team's Tier 2 payment services hadn't received any traffic for the past hour. I logged into the Kubernetes dashboard to investigate and immediately noticed that every pod backing the service was stuck in a continuous restart cycle. At the same time, CPU and memory utilization had spiked across every pod.

To mitigate the initial impact, I contacted the platform team and looking at the memory spike they decided to provision more memory for the service pods. All pods restarted and service was restored in 15 minutes.

## The Investigation

Although, it looked like the memory spike caused the outage. Memory spike had never happened before and the traffic pattern that day was consistent with every day numbers. There wasn’t anything suspicious on service’s datadog dashboards to indicate the spike in memory usage that day. I had no justification to propose a code change to provision more memory on the service’s helm chart to make it consistent with current changes done by platform team through GitOps.

## Looking at logs to narrow down the cause

Since, metrics didn’t reveal enough I started looking at service logs from an hour before the outage. I started querying for keywords like “error”, “error_codes”, “500” etc. I got nothing, so then I started reading every log. I had a very high level knowledge on this service, like who were the clients and the API’s that we expose, but never got the opportunity to work on any feature. By reading logs line by line, I started to understand the downstream dependencies and API flows a little better. I also noticed a downstream service health check failure logs in the service for a good 30 minutes right before the outage.

## Deep Dive into Kubernetes Probes

I was not able to connect the dots instantly. How would a downstream failure cause an upstream service outage ? Also, it looked that their health check failure did not cause the impacted service to return errors either. I wasn’t very knowledgeable on the Kubernetes probes back then. I noticed probe failures in the logs too. So, decided to deep dive into the various kuberenetes probes we were using. This is what I learned.