
1. Codebase : version controlled  
     
2. Dependencies : Explicitly declare and isolate dependencies, Should be in different files & not integrated directly in code.  
    
3. Config : Varies env to env  
    
4. Backing Services : Redis, Databases, S3 etc. It helps making app logic stateless.  
    
5. Build, release and run :  Build : code into executable bundle(Build),  release: Code + Config related to specific environment, run: run stage (runtime) runs the app in execution environment, by launching some set of app processes against selected release.  
      
    Every release should have a unique id.  Release cannot be mutated once it is created.
6. Processes: Execute the app as one or more stateless processes, Stateless and share nothing. Any data that needs to persisted should be stored in backing services.  
      
    The twelve-factor app never assumes that anything cached in memory or on disk will be available on a future request or job. Sticky sessions are a violation of twelve-factor and should never be used or relied upon.   
      
    Session state data is a good candidate for a datastore that offers time-expiration, such as [Memcached](http://memcached.org/) or [Redis](http://redis.io/).  
    
7. Port Binding : Export services via port binding.  
      
    **The twelve-factor app is completely self-contained** and does not rely on runtime injection of a webserver into the execution environment to create a web-facing service. The web app **exports HTTP as a service by binding to a port**, and listening to requests coming in on that port.  
      
    Note also that the port-binding approach means that one app can become the [backing service](https://12factor.net/backing-services) for another app, by providing the URL to the backing app as a resource handle in the [config](https://12factor.net/config) for the consuming app.  
    
8. Concurrency : Scale out via the process model.  
    Any computer program, once run, is represented by one or more processes.   
    A developer can architect their app to handle diverse workloads by assigning each type of work to a _process type_. For example, HTTP requests may be handled by a web process, and long-running background tasks handled by a worker process.  
      
    The process model truly shines when it comes time to scale out. The [share-nothing, horizontally partitionable nature of twelve-factor app processes](https://12factor.net/processes) means that adding more concurrency is a simple and reliable operation. The array of process types and number of processes of each type is known as the _process formation_.  
      
    Twelve-factor app processes [should never daemonize](http://dustin.github.com/2010/02/28/running-processes.html) or write PID files. Instead, rely on the operating system’s process manager (such as [systemd](https://www.freedesktop.org/wiki/Software/systemd/), a distributed process manager on a cloud platform, or a tool like [Foreman](http://blog.daviddollar.org/2011/05/06/introducing-foreman.html) in development) to manage [output streams](https://12factor.net/logs), respond to crashed processes, and handle user-initiated restarts and shutdowns.  
    
9. Disposablilty: Maximize robustness with fast startup and graceful shutdown  
      
    **The twelve-factor app’s** [**processes**](https://12factor.net/processes) **are** **_disposable_****, meaning they can be started or stopped at a moment’s notice.** This facilitates fast elastic scaling, rapid deployment of [code](https://12factor.net/codebase) or [config](https://12factor.net/config) changes, and robustness of production deploys.  
      
    Processes **shut down gracefully when they receive a** [**SIGTERM**](http://en.wikipedia.org/wiki/SIGTERM) signal from the process manager.  
    
10. Dev/Prod Parity : Keep development, staging and prod as similar as possible.  
      
    **The twelve-factor app is designed for** [**continuous deployment**](http://avc.com/2011/02/continuous-deployment/) **by keeping the gap between development and production small.  
    The twelve-factor developer resists the urge to use different backing services between development and production. (**mean that tiny incompatibilities**)  
    **
11. Logs : Treat logs as event streams  
      
    _Logs_ provide visibility into the behavior of a running app. In server-based environments they are commonly written to a file on disk (a “logfile”); but this is only an output format.  
      
    **A twelve-factor app never concerns itself with routing or storage of its output stream.**  Instead, each running process writes its event stream, unbuffered, to stdout.  
    
12. Admin Processes : Run admin/management task as one-off processes.  
      
    Admin tasks (like migrations or scripts) should be treated like part of the same app, not separate random programs, so everything stays consistent and predictable.