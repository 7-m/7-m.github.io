---
layout: post
title:  "Programmatic access to on-demand profiling in Java"
date:   2021-05-22 00:00:00
categories: profiling
---

When it comes to performance optimization, profiling an application is the only way to gain detailed insights into CPU, memory and IO utilization. Things are pretty straightforward when working locally. You run your app in your favorite ide, hook your favorite profiler to it and life's good™. But, things soon get complicated when you want to profile apps deployed on another machine or in a container . Issues like firewalls, port forwarding, security, JMX etc. start cropping up and life isn't as it used to be. This is what I'll try to alleviate in this blog. 

#### Requirements:
- Java 11 +

#### Preface 
To solve tackle this problem we'll use the freely available tool Java Flight Recorder and Java Mission Control (they're now free!! (Please read license agreements carefully) thanks OpenJDK!). Flight Recorder was introduced by JEP 328 into OpenJDK.

Running Flight Recorder earlier was and still can be done through the `java` command with the appropriate flag as 

{% highlight bash %}
java -XX:+FlightRecorder\
  -XX:StartFlightRecording=duration=60s,\
  settings=default.jfc,filename=myrecording.jfr\
  MyApp
{% endhighlight %}

 This is convenient when profiling needs are fixed and you don't plan on changing the profiling configuration once the application is deployed but this is not the case always. Suppose you want to create an additional ad-hoc recording (say you are performance testing ) then you'll have to start and stop the recording during runtime. Now this is possible using `jcmd` the JVM diagnostics command and its all straight forward when creating recordings locally ( using the command `jcmd PID JFR.start`) but when working containerized deployments especially on the cloud, you have to resort to JMX and all the headaches that come with it (which is dealing with establishing and securing TCP connections and guessing ports) this is compounded when working with containers. This begs for a cleaner and leaner approach. One approach is to run `jcmd` programmatically (using `java.lang.ProcessBuilder` and friends) and use it to configure and control JFR. This approach does work in practice but will leave you wanting for an API.

#### How?
We'll leverage `jdk.jfr` package part of Java 11 onwards. The package exposes API's to create and manipulate recordings.

`jdk.jfr.Recording` (https://docs.oracle.com/en/java/javase/11/docs/api/jdk.jfr/jdk/jfr/Recording.html) exposes API to create, start, stop and save Flight Recorder recordings. Its pretty straight forward to create a recording. Here's an example

{% highlight java %}

import java.nio.file.Files;
import java.nio.file.Path;
import jdk.jfr.Configuration;
import jdk.jfr.Recording;

public class App {

  public static void main(String[] args) throws Exception {
    String dir = "/tmp/flight-recorder" ;
    Files.createDirectories(Path.of(dir));

    Configuration recordingConfig =
        Configuration.getConfiguration("default");
    Recording recording = new Recording(recordingConfig);
    recording.setDestination(Path.of(dir + "/myrecording.jfr"));
    
    recording.start();
    System.gc();
    recording.stop();
  }

}

{% endhighlight %}

We start by creating a directory to store the recording in, then create a configuration (when using `jcmd` we use the switch called `settings` and set it to `profile.jfc` or `default.jfc` or any custom configuration you might have ). We then create a `Recording` using the `Configuration`, set the storage path and start the `Recording`. A full GC is triggered using `System.gc()` to generate events for the recording to capture and finally we stop the recording. The recording starts on a separate thread.

The recording file `myrecording.jfr` can be viewed using Java Mission Control.

Its pretty easy to create custom events too. Just extend `jdk.jfr.Event` add some fields to the class and you're done. To use it, create objects of your custom event and call `Event#commit()`.

{% highlight java %}

 class AuthFailureEvent extends Event {
    String message;

    public MyEvent(String message) {
      this.message = message;
    }
    
  }
  
  // ....
  if(!authService.validate(token))
    new AuthFailureEvent("Invalid token : " + token).commit();
  // ....

{% endhighlight %}


The full documentation can be found here https://docs.oracle.com/en/java/javase/14/jfapi/flight-recorder-api-programmers-guide.pdf (Its for Java 14+)

So now what? First of all you can now instrument your Java application with

 `S U P E R   E A S E` 

 I mean think about being able to profile it in production and turning it off!.You could expose this over HTTP, the great thing being how seamlessly this would integrate within your existing backend infrastructure without the need for any extra network configuration or encryption (Assuming you are already running over SSL/TLS). You could also have the profiler data returned over HTTP (an extremely bad idea if not done correctly) or have it persisted to disk for later consumption. Its important to have them meaningfully named, perhaps with a timestamp and host name for easy identification. But, as uncle Ben said, `with great power comes great responsibility®`, designing these endpoints with great care is extremely important because
- You don't want to come back to a server running thousands of recording sessions because you didn't secure the endpoint enough and someone is now `pwning` your web app
- You just leaked super secret environment variables, class names, methods names etc. to the world because you returned profiler data over an insufficiently protected endpoint.

This sort of setup works well in VPN's.

That's all folks!
