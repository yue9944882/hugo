---
title: "How to Monitor Kubernetes Operators By Distributed-Tracing?"
date: 2020-08-02T01:18:56+08:00
---


```
author: "@yue9944882" (Github)
```

Is it feasible to trace those operators running in kubernetes cluster? Currently 
there're two major approaches to monitor your operator instances -- metrics and 
logging. Metrics does well in measuring your operator in concrete numbers at runtime,
and by logging you can record running details into disk files in any format. However
neither of them are helpful in showing the correlation between instances/services. 
Distributed tracing is well-suited for microservice architectures, especially in 
which the services are serving in a synchronous protocol e.g. HTTP, gRPC. Sadly 
that doesn't apply to kubernetes operator pattern. An operator is working 
completely asynchronously, stimulated by the watching event feed from kubernetes 
apiserver. In this blog, we will get a deep insight into the feasibility of 
plumbing distributed tracing on the operator pattern.


# The "Headaches"

Life isn't always happy on the path of bringing operator patterns into practice. A
buggy operator can get stuck into any unexpected behaviors and leaving the developers
in the ocean of confusion:

- __Inresponsive operator__: What if the operator instance doesn't reactively reconcile 
the resource you're trying to modify? There're a number of reasons that can potentionally
cause the issue: (1) the resource is back-off'd by the operator's work-queue due to
previous errors. (2) the watch event can be missed becuase kubernetes' watch doesn't
guarantee every watch-event to be arriving at the operator. (3) The workers/consumers 
routines can be too busy to handle the event.

- __Locate performce bottleneck__: A slow operator can drag down the promptness of the 
whole platform. However the performance bottleneck can be caused by various reasons
and the diagnosing can also get more complicated in a case where more than one operators 
working in a chain/flow. Note that the performance issue is most likely happening in a 
larger cluster.


# Background

As a matter of fact, distributed tracing can also be working for asychronous services
if the propagating formats are properly standardized, e.g. instrumenting AMQP isn't
a tough work because it provides the extensibility to embed the span-context easily
into the asynchronous request payloads. So it looks that instrumenting the kubernetes 
operators with tracing is equally feasible, isn't it? But here's a few things to notice 
that before you actually get start with implementing the integration:

1. __Events processing/consuming is not linear__: 
  
  Operators doesn't directly consume the watch events from kubernetes cluster, the 
  underlying framework (informer, controller-runtime, etc.) does additional processing 
  before we map the event into actual reconciling tasks: (1) multiple watch events can 
  be squashed/dedup'd into single task, or even single event can branch out into multiple 
  tasks. (2) a same task can also be executed multiple times because of the retrying 
  strategy.
  
  Non-linear event processing makes the trace topology hard to visualize. In order to
  address this problem, OpenTelemetry introduced a new concept of [span-links](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md#links-between-spans) 
  to fit operator workflows but sadly it's not yet implemented by any vendors.
  

2. __Cyclic events generation__: 

  Mostly, an operator is expected to refresh/update the target resource's status 
  sub-resource after the reconciliation finishes --- that will respawn another watch 
  event and triggers a new task cyclically. Practically the operator should properly 
  discriminates these cyclic events and short-circuit accordingly, by either dropping 
  the events immediately upon their arrival or on the very beginning of reconciliation. 

  The cyclic event problem uncovered another outstanding issue in terms of tracing 
  operators: __*what's the end of a trace for operators?*__ Currently I presume the 
  answer to be __*when a resource reaches its logical desired state*__, but the test of 
  this condition can be way different for different operators' cases. It's hard to 
  measure blindly regardless of how the operator actually implemented.
  
  
  To see more previous discussion/thread around the topic, click [here](https://github.com/kubernetes/enhancements/pull/650).

  
  # Modelling
  
  
  ### Modelling one-time reconciliation
  
  A complete timeline for an operator to reconcile a single resource will be:
  
  - T1. Received watch event from the resource.
  - T2. Maps the event to one task request otherwise drop it.
  - T3. Pushes the task request into the work-queue.
  - T4. The task request popped from the queue.
  - T5. The task starts.
  - T6. The task ends.
  
  Then we can model the above series into the following spans:
  
  - T1-T2: Handling raw watch event
  - T3-T4: Queuing time cost
  - T5-T6: Reconciling time cost

  (Note that the gaps of T2-T3, T4-T5 are almost neglectable.)
  
  
  
  ### Modelling operator relations
  
  One operator can watch multiple resources at the same time but mostly there's supposed
  to be only one primary resource and we mapped the other secondary resources to its 
  related primary. Hence there's only one logical target resource to process for an 
  operator. For simplication, we're naming the target resource as "watching resource"
  in the following contents.
  
  Before we getting hands on working out a solution, let's 
  have a retrospect sorting operators into different classes:
  
  
  - __Type A - Read-only__: The operator keeps receiving watch events but does 
  nothing (for the most period of time). It sounds hilarious but a doing-nothing
  operator has to be the end-consumer of the event flow regardless of the event source
  or the information it carries. For read-only operators, we can simply model every 
  reconcilation into a "leaf" span.

  - __Type B - Writes third-party system__: Simlar to A, as long as the write target
  is not a kubernetes object, the operator doesn't have to handle cyclic events so every
  reconciliation can also be model'd as a "leaf" span in a trace.
  
  - __Type C - Writes only to its watching resource__: The write can either be upon the 
  resource or its status sub-resource. For the former case, imagine an operator merely 
  attache/dettaches finializers to the new resources, while for the latter, consider the 
  [CSR controller](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/certificates/certificate_controller.go) as an example. Because of the fact that there can be potentionally
  another operator watching the same resource, we're not sure if it's a "leaf" in the 
  picture. But what we're sure is that this operator will be observing the event which
  is triggered by its last write --- the operator can't escape from that. We can model 
  this strong connection into a "parent/child" relation between spans. As for the case 
  where the updates are watched by other system, I will defer the analysis to "Type E".
  
  - __Type D - Fans out writes only to non-watching resources__: [ServiceAccountToken controller](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/serviceaccount/tokens_controller.go) is an excellent example for this type. It can be either leaf or
  non-leaf, depending on whether its fanning-out writes is watched by other systems in 
  the picture. We can model this weak connection into "link" relation between spans.
  
  - __Typd E - Writes to any resources (combination of C+D)__: This is the most complicated
  and unfortunately the most common case. Let's take a look at how [Deployment controller](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go) works in a cluster. It primarily watches deployments, then
  writes to replicasets under certain conditions, these writes upon replicasets will
  go on triggering further works by [ReplicaSet controller](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go).
  For this type, by combining C and D, we can discriminate the writes by their target resource 
  into (1)watching and (2)non-watching, then branch out the spans accordingly: (1) to 
  "parent/child" span relation and (2) to "link" relation.
  
  
  
  
  # Invert the "Picture"
  

  ### From "BUS" to "DAG"
  
  Typically in a kubernetes cluster, all the operators are working in a flattened layer --- 
  kubernetes cluster is a centralized event center and the operators subscribes event 
  from it. The collaborating operators have to exchange data by sending events via kubernetes
  and the resource types are the protocol. 
  
  In conclusion, every operator is loosely-coupled unlike the microservices, so the crux of 
  tracing operator is to __formalize the relation between operators in a unified/standardized 
  term__. This may need a new protocol over kubernetes similar to AMQP, it can also be a 
  CustomResource as protocol.
  
  
  
