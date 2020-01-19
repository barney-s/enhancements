---
title: graduate-cronjob-to-stable  
authors:  
  - "@barney-s"  

owning-sig: sig-apps  
participating-sigs:
  - sig-scaling

reviewers:
  - "@liggitt"
  - "@kow3ns"
  - "@janetkuo"
  - “@mortent”

approvers:
  - "@kow3ns"
  - "@liggitt"
  - "@soltysh"

editor: TBD  
creation-date: 2019-04-18  
last-updated: 2019-04-18  
status: provisional  
see-also:  
replaces:  
superseded-by:  

---

# Graduate CronJob to stable

## Table of Contents

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
- [Goals](#goals)
- [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Promote CronJob API to v1](#promote-cronjob-api-to-v1)
  - [Rearchitect CronJob controller](#rearchitect-cronjob-controller)
    - [Existing controller](#existing-controller)
    - [Proposed rearchitect](#proposed-rearchitect)
      - [Informers and Caches](#informers-and-caches)
      - [Multiple workers](#multiple-workers)
      - [Handling Cron aspect](#handling-cron-aspect)
      - [Metrics](#metrics)
  - [Add .status.lastSuccessfulTime](#add-statuslastsuccessfultime)
  - [Add .status.nextScheduleTime](#add-statusnextscheduletime)
  - [Add Counters](#add-counters)
  - [Add .status.conditions](#add-statusconditions)
  - [CatchUp concurrency support](#catchup-concurrency-support)
  - [Use master timezone instead of UTC](#use-master-timezone-instead-of-utc)
  - [Fix applicable open issues](#fix-applicable-open-issues)
  - [Scale Targets for GA](#scale-targets-for-ga)
    - [CronJob Limits](#cronjob-limits)
    - [Frequency of launched jobs](#frequency-of-launched-jobs)
- [V1 API](#v1-api)
- [v1beta1 changes](#v1beta1-changes)
- [Validations](#validations)
- [Tests](#tests)
  - [E2E test](#e2e-test)
    - [Existing test cases](#existing-test-cases)
    - [New test cases](#new-test-cases)
  - [Conformance Tests](#conformance-tests)
  - [Unit Tests](#unit-tests)
- [Implementation plan](#implementation-plan)
- [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Summary

[CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) is a Kubernetes API that creates Job object on a schedule specified by a cron spec. It is in beta status since v1.8. This document lays out the plan to promote it to stable.

## Motivation

CronJob definition has been stable for the last few releases and is useful to run periodic tasks in kubernetes cluster. This API adds the ability to add cron facility to a cluster. We feel the API is ready to be promoted to Stable and be supported longterm by the community.

## Goals

- Rearchitect CronJob controller
- .status.lastSuccessfulTime
- .status.nextRunTime
- Use master timezone instead of UTC
- Fix open issues and add tests

## Non-Goals
- Changing API field or meaning
- [When creating a cronjob, Reference an existing job](https://github.com/kubernetes/kubernetes/issues/81329)
- Add `CatchUp` concurrency policy

## Proposal

### Rearchitect CronJob controller

#### Existing controller

The current implementation of the CronJob controller is different that the other workload controllers. 
Other controller use informers and caches to reduce the load on API server. The conjob controller does a periodic poll and sweep of all the objects and acts on them. Also CronJob controller has only one worker doing this. 

1. syncs all CronJob objects [every 10 seconds](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/cronjob/cronjob_controller.go#L98). 
2. Using pager library, gets all Pods and all CronJobs and [processes them one by one](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/cronjob/cronjob_controller.go#L136)

This is not a scalable design and ends up loading the API server.

#### Proposed rearchitect

With proposed rearchitecture we aim to:
1. Reduce the potential scale issues when using lots of CronJob objects
2. Reduce load on API server in such cases.
3. Reduce memory usage

##### Informers and Caches
To reduce the need to list all Jobs and CronJobs frequently to reconcile, we propose to replace it with an Informers and WorkQueue based architecture. We would be sharing the [same informer cache](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-controller-manager/app/controllermanager.go#L450) as the Job controller uses. 


```golang

// 
type CronJobController struct {
	kubeClient 			clientset.Interface
	queue	   			workqueue.RateLimitingInterface
	recorder   			record.EventRecorder
	
	jobControl 			jobControlInterface
	cronJobControl  		cronJobControlInterface
	
	jobLister  			batchv1listers.JobLister
	cronJobLister 			CronJobLister
	
	jobListerSynced 		cache.InformerSynced
	cronJobListerSynced		cache.InformerSynced
}

// CronJobControllerConfiguration contains controller config
type CronJobControllerConfiguration struct {
	// concurrentCronJobSyncs is the number of cronjob objects that are
	// allowed to sync concurrently. Larger number = more responsive
	// but more CPU (and network) load.
	ConcurrentCronJobSyncs int32
}


// Controller code

func (cjc *CronJobController)  Run() {
	...
	if !cache.WaitForNamedCacheSync("cronjob", stopCh, cjc.jobListerSynced, cjc.cronJobListerSynced) {
		return
	}
    for i := 0; i < workers; i++ {
        go wait.Until(cjc.worker, time.Second, stopCh)
    }
...
}

func (cjc *CronJobController) func worker() {
    for cjc.processNextWorkItem() {}
}

func (cjc *CronJobController) func processNextWorkItem() {
	key, quit := cjc.queue.Get()
	if quit {
		return false
	}
	defer cjc.queue.Done(key)
	if err := cjc.sync(key.(string)); err != nil {
		utilruntime.HandleError(fmt.Errorf("Error syncing CronJobController %v, requeuing: %v", key.(string), err))
		cjc.queue.AddRateLimited(key)
	} else {
		cjc.queue.Forget(key)
	}
	return true
}

func (cjc *CronJobController)  sync(cronJobKey) {
	childrenJobs := cjc.jobLister.GetJobsForCronJob(cronJobKey)
	// reuse/refactor the existing syncOne function
}


```


##### Multiple workers
We also propose to have multiple workers controller by a flag similar to [statefulset controller](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-controller-manager/app/apps.go#L65). The default would be set to 5 similar to [statefulset](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/statefulset/config/v1alpha1/defaults.go#L34)

```
// Controller code

func (cjc *CronJobController)  Run() {
	...
    for i := 0; i < workers; i++ {
        go wait.Until(cjc.worker, time.Second, stopCh)
    }
...
}

```

##### Handling Cron aspect
To detect which CronJob has met its schedule and need to create Jobs we need to implement a timer component. These are the possible options for implementing the timer:

| algorithm  | how it works | notes |
|:----------|:----------|:-------|
| Unordered timer list | Periodic Sweep from cache | Slower and similar to existing implementation. But improved because we sweep fom the cache instead of API server | 
|Ordered timer list| Maintain ordered list of Cronjob keys and next time of expiry. Keep starting a timer with the earliest expiry. | Efficient. Reinsertion to list takes O(n) |
|Timer trees| Instead of ordered list use a sorted binary tree. | More efficient. Insertion is O(log n) |
|Heap based timer|A variant of ordered timer list where heap is used to store the next expiry time | Efficient compared to ordered list. Bookkeeping and insertion is O(log n). |
|Simple Timing wheels| circular buffer of MaxTimeOut slots. List of expiring timers at each slot. | Works for small bounded  MaxTimeOut which is not our case. Insertion and removal is O(1) via indexing |
|Hashed Wheel| Hash expiring time and insert in a hash table with linked list at each index | Bookkeeping is O(1) and worst case insertion is O(n) |
|Hierarchical Wheel| multiple timer wheels for different resolutions (Seconds, minutes, hours, days). When seconds rolls over we grab the next minutes timers and recreate the seconds wheel. similarly for minutes and hours. | Sharding at different hierarchy levels improves insertion and bookkeeping performance. |


We will introduce a separate queue with the [`DelayingInterface`](https://github.com/kubernetes/client-go/blob/master/util/workqueue/delaying_queue.go#L37) that implements heap based single shot api [`AddAfter`](https://github.com/kubernetes/client-go/blob/master/util/workqueue/delaying_queue.go#L150). Every time we process an entry from this queue, we will add it back to the queue to simulate a periodic timer.


For further reading:
1. [Reinventing timer wheel](https://lwn.net/Articles/646950/)
2. [Hashed and hierarchical timer wheel](http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf)
3. [Golang timers in multi-cpu systems](https://github.com/golang/go/commit/76f4fd8a5251b4f63ea14a3c1e2fe2e78eb74f81)
4. [Go timerwheel](https://github.com/RussellLuo/timingwheel)

##### Metrics
We propose to add metrics that could expose the performance and health of the controller including and not limited to: 
- Skew (actualJobCreationTime-expectedJobCreationTime)
- Queue depth (pending CronJob and Job entries in the Queue)
- Job failures (counter)
- Job successes (counter)
- API server errors (counter)
- Job scheduling latency (podCreationTime - jobCreationTime)


### Add .status.lastSuccessfulTime
[#issue/75674](https://github.com/kubernetes/kubernetes/issues/75674)  
Add `lastSuccessfulTime` to `.status` that tracks the last time the job completed successfully. This will augment the `lastScheduledTime` available in the `.status` in the v1beta1 api. Potential use is in monitoring (e.g. fire an alert if lastSuccessfulTime is more than X ago).

Code for reference, subject to change:

```golang
func (cjc *CronJobController)  syncOne(sj *batchv1beta1.CronJob, js []batchv1.Job, now time.Time, jc jobControlInterface, sjc sjControlInterface, recorder record.EventRecorder) {
   ...
   // Get most recent successfully finished job from current job list.
   sj.Status.LastSuccessfulTime = &metav1.Time{Time: scheduledTime}
	if _, err := sjc.UpdateStatus(sj); err != nil {
		klog.Infof("Unable to update status for %s (rv = %s): %v", nameForLog, sj.ResourceVersion, err)
	}
   ...
}
```

### Add .status.nextScheduleTime
[#issue/78564](https://github.com/kubernetes/kubernetes/issues/78564)
Sdd `nextScheduleTime` to `.status` that tracks the next time the job will be scheduled. This may not be accurate with `Forbid` concurrency policy. This only tracks the `Job` creation time and not the actual `Pod` creation time.

| Concurrency policy | notes |
|:----------|:----------|
| Allow   | `nextScheduleTime` would be accurate within a margin of controller scheduling jitter |
| Forbid  |  `nextScheduleTime` would not be accurate if the previous `Job` takes longer than the cron interval time |
| Replace | `nextScheduleTime` would be accurate within a margin of controller scheduling jitter along with older concurrent `Job` cleanup if applicable. |


```golang
func (cjc *CronJobController)  syncOne(sj *batchv1beta1.CronJob, js []batchv1.Job, now time.Time, jc jobControlInterface, sjc sjControlInterface, recorder record.EventRecorder) {
   ...
   // Find next schedule time
   sj.Status.NextScheduleTime = &metav1.Time{Time: scheduledTime}
	if _, err := sjc.UpdateStatus(sj); err != nil {
		klog.Infof("Unable to update status for %s (rv = %s): %v", nameForLog, sj.ResourceVersion, err)
	}
   ...
}
```

### Add Counters
Add a set of counters that helps users understand a summary of runs.  
- `SuccessfulRuns` Count of all successful runs
- `FailedlRuns` Count of all failed runs
- `FailuresSinceSuccess` Count of all successful runs since last failure

### Add .status.conditions
Add a condition array with `Settled` condition type. This would help with the effort of standarizng conditions across all core types. `Settled` is set at the end of every successful reconcile run. The key thing to note here is the notions of `Settled` does not imply the `Job`s are running correctly. It just means that the controller is done processing this object successfully.

### CatchUp concurrency support
[#issue/79995](https://github.com/kubernetes/kubernetes/issues/79995)
Support for Catch up semantics is being discussed and no consensus has been reached. Adding it to the list for further discussion. Cachup aims to allow for missed runs to be backfilled. We would need to have additional config to bound the backfill and ensure that it doesnt cause the controller to be in backfill mode, never catching up with the current run.

### Use master timezone instead of UTC
We propose to address this request as well: [CronJob Schedule does not use master's timezone - instead it uses UTC](https://github.com/kubernetes/kubernetes/issues/80577)

### Support Jitter for cronjobs
We propose to introduce `.spec.jitter` which is a percentage of the time delta to the next schedule. We propose to cap it to 50%. There is also a [community request](https://github.com/kubernetes/community/issues/2440) for this.

```golang
delta = nextScheduleTime - currentTime
jitter = delta*cronjob.Spec.Jitter/100
nextScheduleTime += jitter
```

### Fix applicable open issues
These are the [current](https://github.com/kubernetes/kubernetes/issues/82659) list of issues that are being targeted for GA. 

- [Updating a cronjob causes jobs to be scheduled retroactively](https://github.com/kubernetes/kubernetes/issues/63371)
- [CLI: Updated CronJob Schedule Missing from Dry Run](https://github.com/kubernetes/kubernetes/issues/73613)
- [Kubernetes CronJob pods is not getting clean-up when Job is completed](https://github.com/kubernetes/kubernetes/issues/74741)
- [Infinite ImagePullBackOff CronJob results in resource leak](https://github.com/kubernetes/kubernetes/issues/76570)
- [Cronjob `spec.schedule` cannot be change when `spec.schedule` value not `"` or `’`](https://github.com/kubernetes/kubernetes/issues/78646)
- [Kubelet CPU/Memory Usage linearly increases using CronJob](https://github.com/kubernetes/kubernetes/issues/64137)
- [Stopping cluster overnight prevents scheduled jobs from running after cluster startup](https://github.com/kubernetes/kubernetes/issues/42649)

### Scale Targets for GA

The scale targets for GA of CronJob are defined by the same [API call latency
SLIs/SLOs as the Kuberetes native types](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/api_call_latency.md#api-call-latency-slisslos-details).

The targets are defined by the below suggested maximum limits, which are organized the same way as the [Kubernetes native type thresholds](https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/thresholds.md#kubernetes-thresholds).

#### CronJob Limits
There should be nothing in the implementation that limits CronJobs per namespace. Overall clusterwide limits of CronJob are important. Cluster wide limits for CronJob should be storage bound since it shares the storage space with all other objects. Determining the appropriate storage limit for a cluster is out-of-scope for this document. We would recommend having CronJob use not more than 15% of the etcd storage. This allows space for Jobs created by CronJobs with the default successfulJobsHistoryLimit of 3 and for other objects.

#### Frequency of launched jobs
The number of CronJobs is also sensitive to the API server QPS and the schedule of the individual CronJobs. This translated to the frequency of launched jobs. We could have large number of CronJobs with a spread of schedule that doenst stress the Job API. At the same time we could have a small number of CronJobs that schedule synchronously stressing the Jobs API. The design must be able to easily saturate the API server QPS.



## V1 API
Most of the changes are adding additional status fields.

```golang
// CronJob represents the configuration of a single cron job.
type CronJob struct {
	metav1.TypeMeta
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta

	// Specification of the desired behavior of a cron job, including the schedule.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
	// +optional
	Spec CronJobSpec

	// Current status of a cron job.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
	// +optional
	Status CronJobStatus
}


// CronJobList is a collection of cron jobs.
type CronJobList struct {
	metav1.TypeMeta
	// Standard list metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ListMeta

	// items is the list of CronJobs.
	Items []CronJob
}

// CronJobSpec describes how the job execution will look like and when it will actually run.
type CronJobSpec struct {

	// The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
	Schedule string

	// Optional deadline in seconds for starting the job if it misses scheduled
	// time for any reason.  Missed jobs executions will be counted as failed ones.
	// +optional
	StartingDeadlineSeconds *int64

	// Specifies how to treat concurrent executions of a Job.
	// Valid values are:
	// - "Allow" (default): allows CronJobs to run concurrently;
	// - "Forbid": forbids concurrent runs, skipping next run if previous run hasn't finished yet;
	// - "Replace": cancels currently running job and replaces it with a new one
	// +optional
	ConcurrencyPolicy ConcurrencyPolicy

	// This flag tells the controller to suspend subsequent executions, it does
	// not apply to already started executions.  Defaults to false.
	// +optional
	Suspend *bool

	// Specifies the job that will be created when executing a CronJob.
	JobTemplate JobTemplateSpec

	// The number of successful finished jobs to retain.
	// This is a pointer to distinguish between explicit zero and not specified.
	// +optional
	SuccessfulJobsHistoryLimit *int32

	// The number of failed finished jobs to retain.
	// This is a pointer to distinguish between explicit zero and not specified.
	// +optional
	FailedJobsHistoryLimit *int32
}

// ConcurrencyPolicy describes how the job will be handled.
// Only one of the following concurrent policies may be specified.
// If none of the following policies is specified, the default one
// is AllowConcurrent.
type ConcurrencyPolicy string

const (
	// AllowConcurrent allows CronJobs to run concurrently.
	AllowConcurrent ConcurrencyPolicy = "Allow"

	// ForbidConcurrent forbids concurrent runs, skipping next run if previous
	// hasn't finished yet.
	ForbidConcurrent ConcurrencyPolicy = "Forbid"

	// ReplaceConcurrent cancels currently running job and replaces it with a new one.
	ReplaceConcurrent ConcurrencyPolicy = "Replace"
)

// CronJobConditionType for CronJob conditions
type CronJobConditionType string

// These are valid conditions of a job.
const (
	// CronJobSettled means the cron job controller has
	CronJobSettled JobConditionType = "Settled"
)

// CronJobCondition describes a condition state
type CronJobCondition struct {
	// Type of job condition, Settled
	Type CronJobConditionType
	// Status of the condition, one of True, False, Unknown.
	Status api.ConditionStatus
	// Last time the condition was checked.
	// +optional
	LastProbeTime metav1.Time
	// Last time the condition transit from one status to another.
	// +optional
	LastTransitionTime metav1.Time
	// (brief) reason for the condition's last transition.
	// +optional
	Reason string
	// Human readable message indicating details about last transition.
	// +optional
	Message string
}

// CronJobStatus represents the current state of a cron job.
type CronJobStatus struct {
	// A list of pointers to currently running jobs.
	// +optional
	Active []api.ObjectReference

	// Information when was the last time the job was successfully scheduled.
	// +optional
	LastScheduleTime *metav1.Time

	// Information for when was the next time the job will be scheduled.
    // May not be accurate in some cases for Forbid concurrency policy
	// +optional
	NextScheduleTime *metav1.Time

	// Information when was the last time the job successfully completed.
	// +optional
	LastSuccessfulTime *metav1.Time

	// Counter for tracking successful Job runs
	// +optional
	SuccessfulRuns int64

	// Counter for tracking failed Job runs
	// +optional
    FailedlRuns int64

	// Counter for tracking failed Job runs since last successful run
	// +optional
    FailuresSinceSuccess int64

	// The set of conditions for this object
	// +optional
	Conditions []CronJobCondition

}
```

## v1beta1 changes
None

## Validations
Nothing additional from v1beta1

## Plan for promoting CronJob to GA

Dual controller (old and new coexist). Feature flag for new controller. alpha (disabled), beta(enabled, can be disabled), ga(enabled)
API will track the controller since GAing the API implies GAing scalable implementation

We will promote to  GA and create `batch/v1/CronJob` in the [batch/v1 API](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/api/batch/v1/types.go). The older versions of CronJob API (batch/v1beta1, batch/v2alpha1) will be deprecated following the [deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/) 


## Tests

### E2E test
CronJob E2E test code is [located here](`/test/e2e/apps/cronjob.go`)

#### Existing test cases

- ConcurrencyPolicy
  - should schedule multiple jobs concurrently when AllowConcurrent
  - should not schedule new jobs when ForbidConcurrent
  - should replace jobs when ReplaceConcurrent
- Suspend
  - should not schedule jobs when suspended
- SuccessfulJobsHistoryLimit
  - should delete successful finished jobs when above successfulJobsHistoryLimit
- FailedJobsHistoryLimit
  - should delete failed finished jobs when above failedJobsHistoryLimit
- Events Recorder
  - should not emit unexpected warnings
  - should remove from active list jobs that have been deleted

#### New test cases
- Schedule
  - Should not create a cronjob with invalid schedule format
- StartingDeadlineSeconds
  - Should not schedule a job within two minutes when missed the current window if 
  - StartingDeadlineSeconds is 0
  - Should schedule a job soon when missed the current window if StartingDeadlineSeconds is long 
- JobTemplate
  - Should not schedule a job with invalid job template
- [Endpoints coverage](https://apisnoop.cncf.io/?zoomed=category-beta-batch)
  - Should list cronjobs for all namespaces
  - Should update a cronjob
  - Should patch a cronjob
  - Should delete all cronjobs in a namespace
  - Should get a cronjob status
  - Should update a cronjob status
  - Should patch a cronjob status
- Tests covering Bug fixes
  - [issue/63371 - Should not start “missed” jobs from old cronjob after updating time](https://github.com/kubernetes/kubernetes/issues/63371) 
  - [issue/74741 - Should cleanup finished pods when job is completed](https://github.com/kubernetes/kubernetes/issues/74741)
  - [issue/76570 - Should not keep creating new pods when job image has ImagePullBackOff error](https://github.com/kubernetes/kubernetes/issues/76570)
  - [issue/78646 - Should change shedule when schedule value is not wrapped with quotes](https://github.com/kubernetes/kubernetes/issues/78646)
- Scaling
  - Should be able to create and schedule atleast 5000 cronjobs
  - Measure scheduling skew, podCreationLatency and check if it meets epxectation (To Be Defined)
- Start Stop Tests
  - Schedule cronjobs and randomly stop the controller and start it.
  - Schedule cronjobs and stop the controller and start it after the deadline.



### Conformance Tests
The conformance tests are a subset of e2e tests. We will select test scenarios that we believe are expected from all conforming clusters. Then modify the test case to use the `framework.ConformanceIt()` function rather than the `framework.It()` function. 


### Unit Tests
This is subject to the new rearchitected controller implementation. Overall these scenarios would be tested.

- [Run or Not](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/cronjob/cronjob_controller_test.go#L167) tests the controller under different scenarios to check if the Job is created or not
- [Validates Job cleanup](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/cronjob/cronjob_controller_test.go#L371) path of the controller under different conditions.
- [Validates Status](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/cronjob/cronjob_controller_test.go#L593) of the CronJob after sync under different conditions.


## Implementation plan
#### Release X
- Alpha: Feature flag for new controller is disabled by default
- Dual controller. Both old and new implementation co-exist
- API will track the controller since GAing the API implies GAing scalable implementation

#### Release X+1
- Beta: Feature flag for new controller is enabled by default. If the distribution chooses it can be disabled

#### Release X+2
- GA: The feature flag is deprecated and the old controller code is cleaned up

## Graduation Criteria

- [ ] Implement shared informers to reduce pressure on API Server
- [ ] Add conformance tests
- [ ] Update documents to reflect the changes
- [ ] Pass CronJob e2e tests
- [ ] Pass CronJob unit-tests
- [ ] Pass scale tests

## Implementation History

- CronJob was introduced in Kubernetes 1.3 as ScheduledJobs
- In Kuberenetes 1.8 it was renamed to CronJob and promoted to Beta