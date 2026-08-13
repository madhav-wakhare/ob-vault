
> **Utilization tells you how busy a resource is. Saturation tells you whether demand is waiting because the resource is busy.**

A burst of high utilization can cause saturation and performance issues, even though utilization is _low_ when averaged over a long interval. This may be counter-intuitive!

I had an example where a customer had problems with CPU saturation (latency) even though their monitoring tools showed CPU utilization was never higher than 80%. The monitoring tool was reporting five minute averages, during which CPU utilization hit 100% for seconds at a time.

`The USE Method is most effective for resources that suffer performance degradation under high utilization or saturation, leading to a bottleneck. Caches _improve_ performance under high utilization.`

The USE Method helps you identify which metrics to use. After learning how to read them from the operating system, your next task is to interpret their current values. For some, interpretation may be obvious (and well documented). Others, not so obvious, and may depend on workload requirements or expectations.

The following are some general suggestions for interpreting metric types:

- **Utilization**: 100% utilization is usually a sign of a bottleneck (check saturation and its effect to confirm). High utilization (eg, beyond 70%) can begin to be a problem for a couple of reasons:

- When utilization is measured over a relatively long time period (multiple seconds or minutes), a total utilization of, say, 70% can hide short bursts of 100% utilization.
- Some system resources, such as hard disks, cannot be interrupted during an operation, even for higher-priority work. Once their utilization is over 70%, queueing delays can become more frequent and noticeable. Compare this to CPUs, which can be interrupted ("preempted") at almost any moment.

- **Saturation**: any degree of saturation can be a problem (non-zero). This may be measured as the length of a wait queue, or time spent waiting on the queue.
- **Errors**: non-zero error counters are worth investigating, especially if they are still increasing while performance is poor.

It's easy to interpret the negative case: low utilization, no saturation, no errors. This is more useful than it sounds - narrowing down the scope of an investigation can quickly bring focus to the problem area.