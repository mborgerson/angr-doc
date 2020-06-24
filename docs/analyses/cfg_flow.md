Control-Flow Graph (CFG) Recovery Flow
======================================

This document is intended to be a walkthrough of what happens when CFG
recovery is performed in angr; specifically, the CFGFast static analysis. That is, this document explores what happens "under the hood" when you run:

```python
>>> cfg = p.analyses.CFGFast()
```

This document is not going to be a line-by-line explanation of the cdoe, but
rather a slightly more descriptive reference, with pointers on where to look
in the code base for specific things. For higher level explanations of CFG
analysis, see [cfg.md](cfg.md).

It's my hope that this document will save developers time, and hopefully give
insight into the general flow inside and outside of angr.

**Note:** This document is currently a work in progress and is not considered
complete.

AnalysesHub, AnalysisFactory
----------------------------
In the line above, note that `p.analyses.CFGFast` does not rerer to the
`CFGFast` class directly. `p.analyses` is an `AnalysesHub` object, and
`CFGFast` is property on that object is actually an `AnalysisFactory` object.
When the `CFGFast` instance of `AnalysisFactory` is called, it will
instantiate the real `CFGFast` class and return it.

Base Classes and Constructor
----------------------------
The `CFGFast` constructor is where the analysis begins. The `CFGFast` class
inherits two other classes which provide important functionality:

- `CFGBase`
- `ForwardAnalysis`

In the `CFGFast` constructor, a call is made to `self._analyze`, which is
inherited from `ForwardAnalysis`.

ForwardAnalysis
---------------
The CFG analysis process is structured as a series of `CFGJob` jobs. A single
job completes a piece of the recovery (e.g. adding a node to the graph), and
during execution of this job additional jobs can be created for further
exploration.

`ForwardAnalysis` (`analyses/forward_analysis/forward_analysis.py`) is an
abstract, queue-based job execution framework. `ForwardAnalysis` handles the
main-loop processing of these jobs until the there are no more jobs to process
or the analysis is interrupted (via the `should_abort` flag).

This class is intended to be inherited, and the exact definition of a "job"
and how to process it is entirely up to the inheriting class. This class is
not restricted to only CFG recovery, it can be used to drive other types of
analyses as well.

### `ForwardAnalysis::_analyze` is the starting point for execution

`_analyze` will first call the `_pre_analysis` abstract method, which for
`CFGFast` will create the initial jobs which seed exploration, as well as
additional initialization.

Next `_analyze` will call `ForwardAnalysis::_analysis_core_baremetal` to run
the job processing loop. This loop will pull jobs from `_job_info_queue`.
These jobs are `JobInfo` objects, which is a wrapper around the abstract job.
A job is processed through `ForwardAnalysis::_process_job_and_get_successors`.

Finally, `_analyze` will call the `_post_analysis` abstract method, which for
`CFGFast` will perform final cleanup on the CFG (e.g. normalization, removal
of rundandant blocks, final construction of functions).

### `ForwardAnalysis::_process_job_and_get_successors`

The main flow of this method is:

```
for successor in self[CFGFast::]._get_successors(job):
	for job in self[CFGFast::]._handle_successor(successor):
		self[ForwardAnalysys::]._insert_job(job)
self[CFGFast::]._post_job_handling(job)
```

### `ForwardAnalysis::_insert_job`

Handles creating the new `JobInfo` object and adding it to the queue. If two
jobs have the same key value, this case can be handled by one of:

A. The original job can be "widenend" to include the new job
   _should_widen_jobs, _widen_jobs -- Both defined by CFGFast.py
B. The original job and the new job can be "merged" into a new job
   _merge_jobs, if merging fails defaults to (C).
C. The new job can replace the existing job in the map.

If job merging is not supported by the concrete class, (C) is performed; that
is, new jobs which have the same job key will replace any existing job with
that key.

Each `JobInfo` object contains a list of jobs. The `job` property returns the
last job in the list. This is used to track history of merged and widened
jobs--as the `JobInfo` structure is retained, and the widened/merged job is
added. Only the last job is processed but jobs that were part of this JobInfo
prior to merge can be queried.
