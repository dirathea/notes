At my current company, we utilize Kubernetes on top of Google Cloud (GKE) as our platform of choice to runs our workloads. The core workload that we runs to serve our users is deployed as Kubernetes Job, created via client library of Kubernetes. Because the nature of our traffic, sometimes Kubernetes Job creation can be burst in high number, and many of the fails due to the following error

```
Operation cannot be fulfilled on resourcequotas "gke-resource-quotas": the object has been modified; please apply your changes to the latest version and try again
```

I can't deny but this error quite strange, especially this is the return value of the Kubernetes client library. Exploring what's `gke-resource-quotas` means, I found that its a ResourceQuota resource on Kubernetes, that automatically injected on our Kubernetes cluster.

# What's ResourceQuota

Based on the [official documentation](https://kubernetes.io/docs/concepts/policy/resource-quotas/), ResourceQuota object created for `provides constraints that limit aggregate resource consumption per namespace.` This explanation starts to make sense, maybe the burst easily hit the limits, thus the Kubernetes will reject the creation.

Checking the burst number that we have, it barely reaching the limit.

```yaml
production        gke-resource-quotas   498d   count/ingresses.extensions: 0/100k, count/http://ingresses.networking.k8s.io: 0/100k, count/jobs.batch: 539/50k, pods: 638/15k, services: 0/5k
```

This is strange, why we seems got bottlenecked by ResourceQuota object, while never reach the limits that it configured. Another suspicion that I have at that time is, the logs says about `object has been modified`, which we don't do it -- the client only send create job request.

# Kubernetes ResourceQuota issue

Quick googling about the issue on Kubernetes repo, I ended up found an open issue talking about [pod creation issue and ResourceQuota](https://github.com/kubernetes/kubernetes/issues/67761). Some folks said that the pod creation got error due to ResourceQuota error even it hasn't reach the limit. Some talk about the workaround by using retry logic when creating pod. I'm not satisfied with the suggested workaround.

Before going further, I remember correctly that we don't implement any ResourceQuota into our cluster, nor anyone who set up the cluster before I join the company. The object name is also has it's own uniqueness. It contains `gke` prefix, which seems like coming from Google Cloud itself.

Considering our usages, for now we don't really need this ResourceQuota implemented to our cluster, so I think in my case, we simply need to remove the ResourceQuota. Before deleting, I try to back it up first via `kubectl get resourcequota gke-resource-quotas -oyaml` just in case we need to restore this.

After backup, I straight go ahead delete the object. It's gone for a while, and comes back for no apparent reasons. This behavior makes me think, that the object added by some kind of controller, managed outside of my configuration. It could be coming from Google Cloud itself.

# Google Cloud Quotas and Limits

Google by default applied some limits on resource that we could spin up. We often encounter this kind of limit when we try to request more cpus, or memory into the project, leading to opening a ticket, asking for more quota for the project. While this seems aligned and well known, I don't see direct relevance between this and Kubernetes ResourceQuota.

Until I found [this link](https://cloud.google.com/kubernetes-engine/quotas#resource_quotas) that explain the exact situation (Kudos to Google Cloud team for clear documentation 🎉). By default, each of GKE cluster with below 100 nodes, will had this ResourceQuotas applied. This also explain why when we delete the ResourceQuota, it keep coming back after some times.

The key information comes from `below 100 nodes`. At that time, we currently had nearly 50 nodes with 80 cores each. We might use many CPUs on that cluster, but the number of nodes is below 100 nodes, thus we still suffer the same issue. That makes me thinking, what if we simply spawn another 50 nodes with smallest node possible, just to remove the ResourceQuota object.

After sharing my plan to the team, and get green light, I simply spawan a very small nodes e2-micro, pre emptible, with smallest disk possible 10G. It costs $111 per month, but remove our bottleneck. Once all the nodes are up, I try to remove the `gke-resource-quotas` and it works as expected 🎉. The error is gone, the retry logic never triggered (means the creation always success in the first attempt). On the other spectrum, we consult with our Google Cloud account manager, and they told us that the ResourceQuota has been disabled, regardless the number of nodes.

# What really happen

My hyphothesis is, when we send burst request to the Kubernetes API, it will try to fullfil those creation. But when multiple Jobs or Pods about to be created, it notify the ResourceQuota controller, check whether the request it allowed (still below the limit)

[![](https://mermaid.ink/img/pako:eNqVkbFuAjEMhl_F8gxDmaobkBDt0MIAnDoRhHwXAxEkuSbOUHG8ewOh6oZaT5a_z_9gn7H1mrHCfaDuAPOVcpBrejLsZD17ru8tzE2zgeFwDL0Dl2zDAfwOAn8mjtJDNiedWc9SBo6FI0wWb5sSVlhZngYm4R7efbN9eoxHj7Er-BZU6EenM4VaSFLsYcXRp9DyMnmhX3n0H9n9VcYBWg6WjM63PF9nCuXAlhVWudUUjgqVu2Qv3cJetREfsNrRKfIAKYmvv1yLlYTEP9KLofwXe7cu39erjNk)](https://mermaid.live/edit#pako:eNqVkbFuAjEMhl_F8gxDmaobkBDt0MIAnDoRhHwXAxEkuSbOUHG8ewOh6oZaT5a_z_9gn7H1mrHCfaDuAPOVcpBrejLsZD17ru8tzE2zgeFwDL0Dl2zDAfwOAn8mjtJDNiedWc9SBo6FI0wWb5sSVlhZngYm4R7efbN9eoxHj7Er-BZU6EenM4VaSFLsYcXRp9DyMnmhX3n0H9n9VcYBWg6WjM63PF9nCuXAlhVWudUUjgqVu2Qv3cJetREfsNrRKfIAKYmvv1yLlYTEP9KLofwXe7cu39erjNk)

# Conclusion

From my experience, some key takeaways are:

1. If ResourceQuota is a must, consider this limitation, and start adding retry on K8S API request. Some users said even with 5 pods per minutes it already got the error.
2. Completely remove the ResourceQuota for now to avoid bottleneck. For GKE specific, simply asks the account manager to remove it, or follow the documentation, which spawn up to 100 nodes to remove resource quota (but remember the costs, $111 per month for avoid K8S bug 😭)

Hope this post helpful! If any of my explanation is incorrect, feel free to reach me at https://dirathea.com/twitter.

Cheers! ✌️