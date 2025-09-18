### 

### Q: You deleted a deployment, but the pods are still running. How?**

_A: The associated ReplicaSet wouldn't have been deleted. If the Deployment was paused or not cleaned up properly, the ReplicaSet would continue maintaining the pods._

### Q: A pod with no memory limit defined is still getting OOMKilled. What could be the reason?

_A: Without a memory limit, the pod gets BestEffort QoS and can overconsume node memory. If the node runs out, the kubelet OOMKills such pods first to reclaim memory._

### Q: A HPA is in place, but even under load, replicas remain at 1. What could cause this?

_A: The metrics server might not be reporting CPU or memory usage properly. Without metric data, the HPA controller would not initiate any scale out actions._

### Q: A Deployment was rolled out, but pods are not created. What should be checked first?

_A: The Deployment might have failed admission due to validation or mutation webhooks. Events or kubectl describe would show webhook related errors that blocked pod creation._

### Q: You exposed a pod via NodePort, but it's still unreachable externally. What’s likely missing?

_A: The node level firewall or cloud NACL might be blocking the assigned port. Even though Kubernetes exposes it, the OS would still drop the traffic if not allowed._

### Q: A container image is updated in the registry, but pods are still pulling the old image. Why?

_A: The deployment might be using an immutable image digest or relying on the default imagePullPolicy: IfNotPresent. In that case, existing nodes would cache the image and avoid pulling the new version._

### Q: You use emptyDir for temporary data, but after a pod restart, all the data is gone. Why?

_A: That’s expected. emptyDir would be wiped whenever the pod is terminated or rescheduled since it’s tied to the pod lifecycle._
