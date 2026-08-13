
> **Utilization tells you how busy a resource is. Saturation tells you whether demand is waiting because the resource is busy.**

A burst of high utilization can cause saturation and performance issues, even though utilization is _low_ when averaged over a long interval. This may be counter-intuitive!

I had an example where a customer had problems with CPU saturation (latency) even though their monitoring tools showed CPU utilization was never higher than 80%. The monitoring tool was reporting five minute averages, during which CPU utilization hit 100% for seconds at a time.

`The USE Method is most effective for resources that suffer performance degradation under high utilization or saturation, leading to a bottleneck. Caches _improve_ performance under high utilization.`

