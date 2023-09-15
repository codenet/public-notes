# MillWheel: fault-tolerant stream processing at internet scale
Keywords: 

## Summary:
Most interesting things to highlight for class:

1. "Strong productions" basically writes the local operator state remotely. 
Exactly-once processing de-duplicates by comparing row-id with already 
acknowledged ones using a bloom filter.

Writing remotely substantially increases the latency! From 30ms to 93.8ms!  They
don't show 99 percentile latency, it's likely much worse!

> Figure 13 shows the latency distribution for records when running over 200
CPUs. Median record delay is 3.6 milliseconds and 95th-percentile latency is 30
milliseconds, which easily fulfills the requirements for many streaming systems
at Google (even 95th percentile is within human reaction time). This test was
performed with strong productions and exactly-once disabled. With both of these
features enabled, median latency jumps up to 33.7 milliseconds and
95th-percentile latency to 93.8 milliseconds. This is a succinct demonstration
of how idempotent computations can decrease their latency by disabling these two
features.

2. As we saw, maintaining state locally is good for latency. But then continuous
operators suck at straggler mitigation! Even if one operators is slow, it can
slow down the whole streaming computation. 

> To verify that MillWheel’s latency profile scales well with the system’s
resource footprint, we ran the single-stage latency experiment with setups
ranging in size from 20 CPUs to 2000 CPUs, scaling input proportionally. Figure
14 shows that median latency stays roughly constant, regardless of system size.
99th-percentile latency does get significantly worse (though still on the order
of 100ms). However, tail latency is expected to degrade with scale more machines
mean that there are more opportunities for things to go wrong.
 
3. Deeper pipelines experience longer watermark lag. Due to late data and
watermarks, the aggregate computation can significantly lag behind real-time!
As much as 2 seconds!

> While some computations (like spike detection in Zeitgeist) do not need
timers, many computations (like dip detection) use timers to wait for the low
watermark to advance before outputting aggregates. For these computations, the
low watermark’s lag behind real time bounds the freshness of these aggregates.
Since the low watermark propagates from injectors through the computation graph,
we expect the lag of a computation’s low watermark to be proportional to its
maximum pipeline distance from an injector. We ran a simple three-stage
MillWheel pipeline on 200 CPUs, and polled each computation’s low watermark
value once per second. In Figure 15, we can see that the first stage’s watermark
lagged real time by 1.8 seconds, however, for subsequent stages, the lag
increased per stage by less than 200ms. Reducing watermark lag is an active area
of development.

These latencies are ok for dip detection like Zeitgeist but they certainly
cannot be called real-time streaming.