---
layout: post
title:  "On Performance testing"
date:   2020-12-5 00:00:00
categories: testing
---

### Brief
- What is performance testing
- Deciding on the type of performance test
- Deciding on tools and metrics
- Creating a synthetic load
- Collect metrics by running the synthetic load
- Analyzing results and fixing bottlenecks

### Introduction
A month ago I was tasked with analyzing the performance of internal and user facing APIs at HomeLane and fix any bottlenecks. The only prior experience I had with performance testing was with my college final year project wherein I had to create demand on a microservice to trigger Kubernetes's auto-scaling. It's been a great learning experience so heres my 1.50 Rupees on the topic.

### What?
Performance testing is an umbrella term for a number of different system performance testing methods like load testing, stress testing and soak testing etc. where the aforementioned 3 are the most common ones you'll come across. The aim of performance testing is to determine, as the name suggests the runtime performance of a system in terms of metrics like latency, throughput, responsiveness and any other project/system specific metric. Analysis of these metrics helps in determining the real world reliability and  performance of the system, its breaking point and where the bottlenecks might lie. Performance testing becomes even more crucial with the current adoption of the microservices architecture and cloud computing as communication overheads become even more significant as a single microservice may have to communicate with one or more microservices or DB's (residing on the same or another machine/region/zone, which will add to network overhead ) to get a task done. In this article I'll solely focus on analyzing the performance of  REST API's.

### The Steps
- So, to begin with performance testing, we first need to decide as to what are we trying to learn about the system, i.e are we trying to know if it’ll be able to handle the expected load for which it was designed? (load testing) or at what load will the system performance become unacceptable for its users (stress testing) or will the system give away in the long run at a standard expected load (soak testing).
If your plan is to gain performance insights at different loads (which is what many would want to begin with), then a step based load test is the way to go where we gradually increase the load in steps on the system from 0 till the metrics become unacceptable (i.e the load test turns into a stress test ). This gives a holistic understanding of the capability of the system and later on this information can be used to fine tune the hardware/software resource allocation or configure load balancers and scaling.

- The next consideration is about the synthetic load. Ideally the load needs to be representative of what the system will experience in production. This brings us to two important questions :
    - How do we ensure that the API calls frequency of the synthetic load corresponds to that of the real world ? and
    - How do we enforce the relative ordering between API calls ? ( you don’t want to operate on resources that haven't been created yet i.e calling PUT/PATCH/GET on a resource before POST )
There are ways to go about doing it but the most convenient and straightforward way for web applications  is to to record the API requests while a user navigates the application. You can do this using Chrome’s or Firefox’s “save as HAR”  from the console or use JMeter’s proxy server or Blazemeters free recording plugin (note that the latter are applicable only if you use JMeter). You could invite a few people to try out your web application and record a few unique sessions and then replicate these to create the artificial load.

- To run the load we need a tool which will simulate users (aka virtual users). JMeter and K6 are two open source tools worth looking into. They are opposites in the way you write tests with them. While JMeter has a nice GUI to work with, k6 uses ECMA script ( Javascript) as a DSL to write test plans (a sequence of http requests). JMeter is great tool to start with if you are getting just into performance testing as it comes batteries included with features like extractors ( to extract id’s, auth tokens etc from responses), control blocks, scripting( supports scripting in Groovy), visualizers etc. JMeter is what I used for my tests with JSON extractor and View Results Tree Listener along with a script to parse and aggregate the results from JMeter.

- Note: While testing make sure the servers you will run the test against won’t cause disruptions to other organizational activities. Also make sure the test servers hardware and software configuration is the same as that of the production environment. Also make sure to run the tests from a beefy system or a cluster or else the bottleneck at the load running machine might skew the final results

- We are now ready to pummel the servers, so , lets get down to the specifics of how to conduct the test. Define metrics that you’d like to measure. API avg RTT (Round Trip Time)( consider collecting percentiles too, discussed about it later) & invocation frequency, throughput,  CPU & memory utilization at the server are some metrics to get started with. Feel free to define your own metrics as per your needs. You don’t have to measure all of them in the first run. RTT and frequency are the ones I started with, but having CPU and memory averages are helpful to know at what capacity is the backend operating. Add  in extra metrics as you deem fit as you gain a better understanding of system performance and start tracing bottlenecks. Start at 0 users and increase the no. of users in steps after each run instead of starting with an arbitrary no. of users. This way you get a holistic picture of how system performance remains nearly stable and then degrades with additional load which will help in deciding load balancing and scaling strategies and configurations. Continue increasing the users until the metrics become unacceptable for the user. Divide the session recordings equally among the virtual users, i.e if you plan on simulating 4000 users concurrently and you have 4 sessions, then launch each session 1000 times in parallel. Again you can play with this distribution once you have a better understanding of the user base.

As with RTT, consider working with percentiles along with averages. Averages hide users who might have experienced a much higher than acceptable latency. 70th percentile RTT is what I think is a good statistic to calculate.


### Analyzing
You could stop above with the results. They should give you a good idea of what the system’s real world performance would be but if it isn’t up to expectations, then the results should be analyzed to determine bottlenecks in the system. Here’s a checklist based on my experience for bottlenecks in the system and how to detect them. The list can be divided into CPU bound and IO bound bottlenecks
- CPU bound
    - Unoptimized code : profiling
- IO Bound
    - Too many DB calls : application logs, db profiling
    - Unoptimized db queries causing long query execution times : db profiling
    - Long API calls to external services : application logs
    - Runtime constraints (ex. Java heap space limits + large long living objects + GC) : profiling
    - Network overhead 

In CPU bound bottlenecks, you'll see high (90% - 100%) average CPU utilization while in IO bound bottlenecks you'll see idle threads in your profiler usually accompanied by low CPU utilization ,just waiting on an IO operation to complete. At some point in your journey of resolving bottlenecks you'll run into a state in which the API can't be optimized any further due to the inherent complexity of the API. At this point its time to look into other API's to optimize further instead of doing micro-optimizations (unless its warranted and backed by solid proof)

As to the question which API should you start with first, I prefer going with MAX([frequency * avg. RTT ]), basically the one where most of the time was spent.


### Pitfalls
Here are the 2 of the pitfalls I came across that'll likely skew the final results
- Caching at the DB can skew your results. A way to alleviate this is to create new records each time and not operate on existing records.
- A more general version of the above is that caching at any point in the system can skew you results, now reading this one wonders that, the purpose of a cache is to make R/W faster, but in the methodology I described above, we have the same session recording running repeatedly with a lot of repeated data. This repeated data will be cached at some point in the system leading to skewed results. So whats a workaround for this? Either try to limit the repetition of queries for the same dat or perhaps disable caching throughout the system. The results again will not be reflective of the real world but at least the results will be on the worse side so you can expect the real world performance to be equally bad or better. Its a tough nut to crack :/. 
- If your backend stack uses JIT compilation (like the JVM) , then warm it up before testing. 
- Make sure the machine running the load has ample n/w and h/w resources so that it doesn't choke. If the load can't maxes out a single machine then consider running the load across multiple machines.

### My Takeaways
The hardest part of performance testing is that which comes after performance testing which is resolving the bottlenecks. Its pretty straight forward to collect metrics for a system once you get the knack of it but the hard part is in pinpointing the bottlenecks as they aren't easy to detect. One piece of advice I'd like to offer is to automate the process of running the tests. This will save you a lot of time as you'll be running the tests to collect metrics and then again after fixing the bottlenecks to verify fixes. This will save a ton of time and prevent user errors.  Another thought I have is that it'd be a great idea to incorporate and automate performance testing into the CI pipeline. This will give a steady stream of metrics on the performance of API's and any in case of significant deterioration of the metrics, they can be caught early and be easily blamed on the latest pull/merge. But, the idea is a bit tricky to implement.

That's all folks, Happy performance testing!



