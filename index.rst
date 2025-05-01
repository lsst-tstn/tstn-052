##################################
Control System meltdown with kafka
##################################

.. abstract::

   This technote contains a content extract from OBS-795 which followed up our major issue with Kafka deployed at the summit, the subsequent work done to investigate it and, eventually, resolve the issue.

   In the end, the issue ended up being a throttling setting that was inadvertently included in the Kafka brokers configuration.
   Once this setting was removed, the issue went away and we were able to run the system even with a much higher throughput than it will normally generate.

Description
===========

The following is a (personal) recount of the incident that happened on Feb/28/2025 with the control system at the summit (production environment), which triggered the follow up analysis.

The first heartbeat alarms started around 02/28/2025 04:59:19 UTC, indicating that the system started to experience issues.
This happened soon after M1M3 was brought up for testing (around 02/28/2025 04:58:29).
M1M3 is the heaviest component of the system producing high data volumes (6-8 MB/s).
We had already experienced similar issues before when M1M3 was initialized, but at the time the system was able to stabilize after some time.

During our initial assessment of the situation we noticed a high volume of failed requests from a user Jupyter notebook instance on nublado.
After we shutdown that users instance and silenced M1M3, the system showed some signs of recovery.
We proceeded with the planned daytime activities, which consisted of soak tests with the MTMount (in CCW only mode running on level 3, no telescope telemetry) and the Rotator, with additional parallel tests being done on M1M3 support and thermal systems.

This didn’t last much longer and the system soon started to degrade again.
We then noticed that one of the three kafka brokers was experiencing high load.
The metrics system was not running and we had no clear view on whether the broker was being throttled by k8s or not.
We then decided to restart that particular broker.
The system again showed signs of recovery only to later descend again into a degraded state as another of the kafka brokers started to show signs of high load.
We again restarted the broker however, this time it made no impact in the system. 

Our next action was then to increase the resource allocation to the Kafka brokers, to ensure that the system was not being throttled.
This requires restarting the brokers, which we did with the system still running.
It also didn’t made any difference and the system was still in a degraded state.

At this time, without being able to determine what the problem was, and after attempting to restart some of the components that were showing signs of being more affected, we decided to bring the entire system down.
The system was in such a state that we were unable to gracefully shut it down, and had to resort to forcibly terminating the components.
We then made sure that the system was as silent as possible, tracking down any component that was left behind.
At this point we again stumbled upon a user nublado instance that was left behind.
There were also instances of the EUIs for M1M3 support system, thermal system and VMS running.
They were all brought down during this process.

With the system in a quiet state we started to bring back up components in groups and monitoring the state of the system.
We started with LOVE, then obssys, envsys, auxtel and maintel.
At each stage we would leave the system running (with the components in Standby) for some time before bringing up the next group, monitoring the system state in the process. 
After this, all the remaining components were brought up, including M1M3 support system, thermal system and VMS. 
At this point we started to enable some of the components like ESS etcs.
We attempted to enable M1M3 support system but the component had an interlock and could not be enabled.
The M1M3 thermal system was left in DISABLED state, which is enough to produce telemetry. 
Finally we enabled the Watcher.
The AuxTel and most of the SimonyiTel components were left in Standby.
With the lack of monitoring it is hard to pinpoint the exact cause of this issue.
We will have to investigate the system further afterwards when we are able to enable M1M3, hopefully with the monitoring systems running.

March 2 2025
============

We were pondering why we don’t see this behavior on BTS. We know the M1M3 simulator puts out less data than the real one at the summit so, our initial idea is that this could be due to the reduced throughput. 

I then decided to test what happens at the summit if we bring the m1m3 simulator up, noticing that the other MT components are running and producing data on BTS but not at the summit (with the exception of M2).

As soon as I brought the M1M3 simulator up, the system again started to degrade.
This points out to something else is going on at the summit we will need to investigate.
It is worth pointing out that the behavior is slightly different at with the simulator though.
The system is able to recover after some time when the simulator is taken down, which we did not observe with the real M1M3.

March 5 2025
============

Today we got a task force together with people from tssw, square and IT to investigate the issue.
We also cancelled observations with AuxTel and reserved the control system at the summit to run our debugging session, since the BTS has not manifested the issue.

There are basically 3 fronts we are investigating:

- Infrastructure

  This is mostly IT.
  They have monitoring on computing resources and network.
  All allocations are way within the resource allocation and there is no clear issue with the network, like package loss, filtering, etc.
  It seems that it is unlikely that it is some infrastructure issue at this point.

- Application

  Basically, this is ts_salobj.
  There could be some issue with how salobj is handling producing data that is causing the issue.
  In order to check this we created a simple minimum write python script that only creates a producer and publishes data to kafka.
  We could replicate the issue by running several of these in parallel, with the system completely silent.

- Kafka broker configuration

  The brokers have a rich set of configurations.
  We might be hitting some limit imposed to the system that we are not aware of.


During the day we were able to reliably reproduce the issue both by bringing up m1m3 and by running the minimum write python script in parallel.
We noticed that once we reach a throughput of about 12MB/s the issue starts to manifest.
During this testing period we also developed some metrics and tools to diagnose the system.
Initially we could not find anything that would indicate an issue.

Finally after we have wrapped up for the day, I started to investigate on producer metrics.
The producer can be configure to retrieve a series of metrics about its performance.
One metric in particular called my attention, “Broker throttling”.

I then updated the minimum write script to output broker metrics at a 5s interval and inspected throttling (as well as some other metrics).

It was evident then that, when the communication issue manifested, the producers were being throttled by the broker.
This basically explains why the effect is affecting the entire system.
After a certain threshold is reached, the brokers start to throttle all producers.
The figure below shows the throttling metrics for 2 different runs; one with 13 parallel writers (no throttling, system behaves as expected) and one with 14 parallel writers (broker is throttling the producers and we observe the missing heartbeat and communication breakdown issue).

.. image:: /_static/producer-throttle.png
   :target: ../_images/producer-throttle.png
   :alt: Kafka producer throttle metric.

Our next steps are:

- Ensure we can collect throttling metrics from the broker directly.
  The plots above are for producer/client throttling.
  We need to know what the brokers are pushing into the system.

- Find how to update the threshold such that the system is not throttled.

March 6 2025
============

On Wednesday we continued with the efforts on stabilizing kafka at the summit.
Now that we had found a smoking gun we at least had something to follow on to.
In the morning we spend some time organizing the metrics in our system and adding additional metrics to allow us to investigate the issue.
The most critical one was to track throttling metrics from the Kafka brokers.

Once we had that in place, we turned our attention to BTS.
We were curious to see how BTS would behave under load, especially considering that the brokers on BTS runs with the same configuration as the ones at the summit.

We were able to reproduce the behavior on BTS by running 20 parallel processes.
We didn’t spend much time trying to find the threshold as this was enough for us to make progress, now that we could reproduce the problem reliably on both BTS and Summit.

At the same time we noticed that the metric we were looking for, the brokers throttling, were activated once we crossed the threshold.
We were a bit confused as to why the brokers were throttling the system.
We had set the “quota” to 1 Gigabits / second and we were nowhere near that limit.

We then asked ourselves, why are we applying a quota in the first place?
We have plenty of resources available and we are nowhere near any resource limit.
We then decided to remove quota altogether.
According to the Strimzi documentation, if we don’t specify any quota, none will be applied (which is what we wanted).

Once we made this change, we executed the throughput tests again and there was no throttling!

We then decided to increase the throughput to as much as we could.
We tested with up to 100 parallel processes, generating close to 300MiB/s (steady state is around 10MiB/s) and the system survived without any issues, no throttle was applied to the system.

The following screen shots highlight some of the metrics we were monitoring during the test.
It shows the received bandwidth, transmit bandwidth, rate of packages received and transmitted before, during and after the stress test.
It is possible to see that we achieve close to 300MiB/s and how that scales with the rate of the steady state system (with all the major components in enabled publishing telemetry).


.. image:: /_static/bandwidth-metrics.png
   :target: ../_images/bandwidth-metrics.png
   :alt: Brokers bandwidth metrics.

This test was executed at both BTS and Summit.
At BTS we had both MTCS and ATCS enabled during the exercise, whereas, at the summit, we had ATCS and M1M3.
We could not enable more of the MTCS components because of the move of LSSTCam to the 8th floor.

Conclusion
==========

- The communication issue we observed was due to a produce quota being applied to the kafka brokers. 

  Once we surpassed a certain message throughput rate, the brokers were throttling all the producers, causing a system-wide effect.

- This configuration was imported from standard suggested configurations, and basically serve to protect against runaway applications producing too many messages (which is critical for public cloud applications).
  By removing this limit on the Kafka brokers we are able to eliminate the problem altogether. 
  We are confident now that the system can sustain a much higher throughput than desired and we currently have a lot of room for expansion.

- It would be advisable to increase the number of nodes available to the kafka brokers.

  We are currently running with 3 dedicated nodes.
  We are nowhere near any resource limit with the brokers, however, if one of the nodes were to fail we would be in a situation where the system would be crippled.
  The brokers require an even number of nodes so I suggest we add 2 more nodes to the cluster bringing the number to 5 nodes instead of 3.

For reference:

- `This <https://github.com/lsst-sqre/phalanx/pull/4310/files>`__ is the PR to remove the quotas from the brokers. 

- The script used to test the throughput can be found in `this <https://github.com/tribeiro/kafka-minimal-write>`__ git repo.


