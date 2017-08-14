How to Run Scalability Tests on a CE
====================================

Introduction
------------

This document is intended as a general overview of the process for scalability testing of an OSG CE (Compute Element). All examples are for testing an [HTCondor-CE](https://twiki.grid.iu.edu/bin/view/Trash/Trash/Sandbox/HTCondorCEScaleTests), but they should be applicable for other CE software.

The focus of testing a CE is on the number of concurrent running jobs the CE can sustain as well as the ramp-up rate when many jobs are queued.

Sleeper Pool
------------

With the focus on the CE, actual job payloads can be minimal – simple long sleep jobs are fine. Thus, then can run on nearly any resources, and it is even possible to allow far more of these jobs to run on a single resource than would be sensible for real jobs.

When large-scale testing a CE, one of the objectives is to see if the CE can fully utilize all resources (cores) available to it or if there are bottlenecks preventing that outcome. However to do this would normally require using up production slots, and it is hard to find a site willing to give up so many production slots for so long. Thus, running resourceless jobs in parallel with production jobs allows the testing to proceed without interfering with real work.

### Setting Up a Sleeper Pool

A sleeper pool is created by “tricking” a worker node into thinking it has more cores than physically available. Then, the host is configured so that jobs marked for the sleeper pool are routed to the extra slots. In HTCondor, this is done by changing the START expression on each startd. For example, on a 32-core machine:

``` file
START = ( \
          (SlotID >= 1) && \
          (SlotID < 33) && \
          (RequiresWholeMachine =!= TRUE ) && \
          (SleepSlot =!= TRUE) && \
          (distro =?= "RHEL6" ) && \
          (CPU_Only == TRUE ) \
          ) || \
          ( (SlotID >= 33) && (distro =?= "RHEL6" ) && (SleepSlot == TRUE) )
```

Usual Topology of the Tests
---------------------------

A brief introduction to the topology involved in the tests.

### Batch System and Sleeper Pool

This is normally the batch system of the resources which will be behind the CE to be tested. It is normally set up by a site administrator.

### CE

This is the physical hardware where the CE software runs, hopefully mimicking real production hardware specifications.

### Submitter

An HTCondor submit host. It can be a virtual machine for most test submissions.

Monitoring tools
----------------

To monitor tests, two software components are needed (which can be installed on the same node): ganglia-gmond and ganglia-gmetad. Once they are installed, then some ad-hoc metrics can be created to monitor the CE; for example:

``` console
condor_q -pool red.unl.edu:9619 -name sleeper@red.unl.edu -const 'JobStatus=?=2' | wc -l
gmetric --name RunningJobsCE 
```

Generating Load
---------------

### Location

The load\_generators are found in the  [OSgscal github repo](https://github.com/efajardo/osgscal). The binary of interest here is `loadtest_condor`

### Use

Just untar it or check it out from the repo into the HTCondor submit node (see above):

``` console
git checkout https://github.com/efajardo/osgscal
cd load_generators/loadtest_condor/trunk/bin
```

Keep in mind that you also need a valid proxy for grid submissions.

For example, if the goal is to keep 1,000 jobs in the queue and run 6-hour sleep jobs (on average), you can run this command:

``` console
./loadtest_condor.sh -type grid condor sleeper@red.unl.edu red.unl.edu:9619 -jobs 40000 -cluster 10 -proxy /home/submituser/.globus/cmspilot01.proxy -end random 21600 -maxidle 1000 -in sandbox 50
```
