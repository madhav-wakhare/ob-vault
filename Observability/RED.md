
The RED Method defines the three key metrics you should measure for every microservice in your architecture. Those metrics are:

- (Request) **R**ate - the number of requests, per second, you services are serving.
- (Request) **E**rrors - the number of failed requests per second.
- (Request) **D**uration - distributions of the amount of time each request takes.

Another nice aspect of the RED method is that it helps you think about how to build your dashboards. You should bring these three metrics front-and-center for each service and error rate should be expressed as a proportion of request rate.

