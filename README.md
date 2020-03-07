<p>Resources: <a href="https://kubernetes.io/docs/concepts/configuration/assign-pod-node/">https://kubernetes.io/docs/concepts/configuration/assign-pod-node/</a>
</p>
<h1>Introduction</h1>
<p>This document is intented to review the batch and stream affinity and anti-affinity rules. For this we will consider the cases of Flink and Spark separately, because their characteristics are slightly different. Their characteristics are, for the most part, quite different from 'services'.</p>
<p>For services, the main motivation for anti-affinity rules is high availability. In case of a Node or Zone failure we want to avoid any interruption of service. For Batch and Streaming pipelines this may however be unavoidable.</p>
<h1>Characteristics</h1>
<h2>Spark</h2>
<p>Batch pipelines are relatively short lived. Often less than an hour, definitely not more than a day.</p>
<ul>
  <li>The less time it takes a batch pipeline to run, the less vulnerable it is to outages.</li>
  <li>The Spark Master pod is configured to recover from a persistent volume. After a failure it will be able to retrieve the job information.</li>
  <li>Spark is resilient enough to handle losing one or more of its worker nodes. Spark will detect the failure, and attempt to re-compute the data that was lost.</li>
  <li>The Spark driver process is not restart-able itself. If the driver is lost the whole pipeline job needs to be resubmitted.</li>
  <li>Data exchange between worker nodes ("executor processes") is one of the main limiting factors in processing speed. Each time Spark reshuffles, large amounts of data is exchanged across the network.</li>
</ul>
<h2>Flink</h2>
<p>Stream pipelines are intended to run 24/7.</p>
<ul>
  <li>Flink's jobmanager comes with a High availability option. If used, then multiple jobmanager nodes will be provisioned, only one of which is considered the leader.</li>
  <li>A high available jobmanager relies on a zookeeper cluster for storing job metadata and leader election.</li>
  <li>Flink provides checkpoints capability to save snapshots of state. If enabled, restart of the Flink job inside the pipeline namespace should be able to resume/recover from the latest checkpoint.</li>
  <li>When a taskmanager is lost, the Flink job will go through an automatic restart (if configured)</li>
  <li>Data exchange between taskmanager nodes is one of the main limiting factors in processing speed. Depending on the Flink Job graph and parallelism of operators, large data will be exchanged across the network.</li>
</ul>
<h1>Consideration</h1>
<p>We can consider the following rules for pipelines:</p>
<h2>Batch</h2>
<h3>Master</h3>
<p>
  <u>Affinity:</u> The master pod shall be scheduled in the same availability zone as the workers and driver pods for the same pipeline job.</p>
<h3>Worker</h3>
<p>
  <u>Affinity:</u> The worker pods shall be scheduled in the same availability zone as the master and driver pods for the same pipeline job.</p>
<p>
  <u>Affinity:</u> The worker pods shall be scheduled where possible on the same node for the same pipeline job. The rationale for this is to further reduce network IO, and minimize fragmentation. The flip-side is that more workers may be impacted by a node failure.</p>
<h3>Driver</h3>
<p>
  <u>Affinity:</u> The driver pod shall be scheduled in the same availability zone as the workers and master pods for the same pipeline job.</p>
<h3>Monitor</h3>
<p>Although it could be useful to schedule the monitor in the same zone, the process is resilient enough to be scheduled anywhere.</p>
<h2>Stream</h2>
<h3>Jobmanager</h3>
<p>
  <u>AntiAffinity:</u> the redundant jobmanagers shall be scheduled across different zones as much as possible for the same pipeline job.</p>
<p>
  <u>AntiAffinity:</u> the redundant jobmanagers shall be scheduled across different nodes where possible for the same pipeline job, to avoid pods scheduled on the same node after a zone failure.</p>
<h3>Zookeeper</h3>
<p>
  <u>AntiAffinity:</u> the redundant zookeepers shall be scheduled across different zones as much as possible for the same pipeline job.</p>
<p>
  <u>AntiAffinity:</u> the redundant zookeepers shall be scheduled across different nodes where possible for the same pipeline job, to avoid pods scheduled on the same node after a zone failure.</p>
<h3>Taskmanager</h3>
<p>
  <u>Affinity:</u> The taskmanager pods shall be scheduled in the same availability zone for the same pipeline job.</p>
<p>
  <u>Affinity:</u> The taskmanager pods shall be scheduled where possible on the same node for the same pipeline job. The rationale for this is to further reduce network IO, and minimize fragmentation. The flip-side is that more taskmanagers may be impacted by a node failure.</p>
<h3>Job Launcher</h3>
<p>
  <span style="color: rgb(23,43,77);">Since the joblauncher is a short lived, run-once pod, it does not matter where it is scheduled.</span>
</p>
<h3>Monitor</h3>
<p>Although it could be useful to schedule the monitor in the same zone, the process is resilient enough to be scheduled anywhere.</p>
<p>
  <br/>
</p>
