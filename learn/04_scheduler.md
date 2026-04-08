# Action 和 Plugin 接口

- 代码位置: ./pkg/scheduler/framework/interface.go

```go
type Action interface {
	// The unique name of Action.
	Name() string

	// Initialize initializes the allocator plugins.
	Initialize()

	// Execute allocates the cluster's resources into each queue.
	Execute(ssn *Session)

	// UnInitialize un-initializes the allocator plugins.
	UnInitialize()
}

// Plugin is the interface of scheduler plugin
type Plugin interface {
	// The unique name of Plugin.
	Name() string

	OnSessionOpen(ssn *Session)
	OnSessionClose(ssn *Session)
}
```

# Action 实现

- ./pkg/scheduler/actions

# Plugin 实现

- ./pkg/scheduler/plugins

# Session struct

- ./pkg/scheduler/framework/session.go

```go
// Session information for the current session
type Session struct {
	UID types.UID

	kubeClient      kubernetes.Interface
	vcClient        vcclient.Interface
	recorder        record.EventRecorder
	cache           cache.Cache
	restConfig      *rest.Config
	informerFactory informers.SharedInformerFactory

	TotalResource  *api.Resource
	TotalGuarantee *api.Resource
	TotalDeserved  *api.Resource
	// PodGroupOldState contains podgroup status and annotations during schedule
	// This should not be mutated after initiated
	PodGroupOldState *api.PodGroupOldState
	// DirtyJobs include the jobs that need to flush to SchedulerCache on session close
	DirtyJobs sets.Set[api.JobID]

	Jobs           map[api.JobID]*api.JobInfo
	Nodes          map[string]*api.NodeInfo
	CSINodesStatus map[string]*api.CSINodeStatusInfo
	RevocableNodes map[string]*api.NodeInfo
	Queues         map[api.QueueID]*api.QueueInfo
	NamespaceInfo  map[api.NamespaceName]*api.NamespaceInfo

	// NodeMap is like Nodes except that it uses k8s NodeInfo api and should only
	// be used in k8s compatible api scenarios such as in predicates and nodeorder plugins.
	NodeMap   map[string]fwk.NodeInfo
	PodLister *PodLister

	Tiers          []conf.Tier
	Configurations []conf.Configuration
	NodeList       []*api.NodeInfo
	// HyperNodes stores the HyperNodeInfo of each HyperNode
	HyperNodes           api.HyperNodeInfoMap
	HyperNodeTierNameMap api.HyperNodeTierNameMap
	// HyperNodesSetByTier contains a set of hyperNodes by tier from down to top, nodes under the same hyperNode
	// have the same topology domain, e.g., nodes under the same switch or tor, jobs allocated in the same
	// hyperNode can gain a better performance, the lower the tier of hyperNode, the better performance.
	HyperNodesSetByTier map[int]sets.Set[string]
	HyperNodesTiers     []int
	// RealNodesList maps hyperNode Name -> nodes under the hyperNode.
	RealNodesList             map[string][]*api.NodeInfo
	RealNodesSet              map[string]sets.Set[string]
	HyperNodesReadyToSchedule bool

	plugins             map[string]Plugin
	eventHandlers       []*EventHandler
	jobOrderFns         map[string]api.CompareFn
	queueOrderFns       map[string]api.CompareFn
	victimQueueOrderFns map[string]api.VictimCompareFn
	taskOrderFns        map[string]api.CompareFn
	clusterOrderFns     map[string]api.CompareFn
	predicateFns        map[string]api.PredicateFn
	prePredicateFns     map[string]api.PrePredicateFn
	bestNodeFns         map[string]api.BestNodeFn
	nodeOrderFns        map[string]api.NodeOrderFn
	batchNodeOrderFns   map[string]api.BatchNodeOrderFn
	nodeMapFns          map[string]api.NodeMapFn
	nodeReduceFns       map[string]api.NodeReduceFn
	hyperNodeOrderFns   map[string]api.HyperNodeOrderFn
	preemptableFns      map[string]api.EvictableFn
	reclaimableFns      map[string]api.EvictableFn
	overusedFns         map[string]api.ValidateFn
	// preemptiveFns means whether current queue can reclaim from other queue,
	// while reclaimableFns means whether current queue's resources can be reclaimed.
	preemptiveFns                 map[string]api.ValidateWithCandidateFn
	allocatableFns                map[string]api.AllocatableFn
	jobReadyFns                   map[string]api.ValidateFn
	jobPipelinedFns               map[string]api.VoteFn
	jobValidFns                   map[string]api.ValidateExFn
	jobEnqueueableFns             map[string]api.VoteFn
	jobEnqueuedFns                map[string]api.JobEnqueuedFn
	targetJobFns                  map[string]api.TargetJobFn
	reservedNodesFns              map[string]api.ReservedNodesFn
	victimTasksFns                map[string][]api.VictimTasksFn
	jobStarvingFns                map[string]api.ValidateFn
	simulateRemoveTaskFns         map[string]api.SimulateRemoveTaskFn
	simulateAddTaskFns            map[string]api.SimulateAddTaskFn
	simulatePredicateFns          map[string]api.SimulatePredicateFn
	simulateAllocatableFns        map[string]api.SimulateAllocatableFn
	subJobReadyFns                map[string]api.ValidateFn
	subJobPipelinedFns            map[string]api.VoteFn
	subJobOrderFns                map[string]api.CompareFn
	hyperNodeGradientForJobFns    map[string]api.HyperNodeGradientForJobFn
	hyperNodeGradientForSubJobFns map[string]api.HyperNodeGradientForSubJobFn

	// cycleStatesMap is used to temporarily store the scheduling status of each pod, its life cycle is same as Session.
	// Because state needs to be passed between different extension points (not only used in PreFilter and Filter),
	// in order to avoid different Pod scheduling states from being overwritten,
	// the state needs to be temporarily stored in cycleStatesMap when an extension point is executed.
	// The key is task's UID, value is the CycleState.
	cycleStatesMap sync.Map

	NodesInShard sets.Set[string]
}
```
