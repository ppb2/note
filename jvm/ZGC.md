https://zhuanlan.zhihu.com/p/170572432

| General GC Options                                           | ZGC Options                                                  | ZGC Diagnostic Options (-XX:+UnlockDiagnosticVMOptions)      |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `-XX:MinHeapSize, -Xms````-XX:InitialHeapSize, -Xms``-XX:MaxHeapSize, -Xmx``-XX:SoftMaxHeapSize``-XX:ConcGCThreads``-XX:ParallelGCThreads``-XX:UseDynamicNumberOfGCThreads``-XX:UseLargePages``-XX:UseTransparentHugePages``-XX:UseNUMA``-XX:SoftRefLRUPolicyMSPerMB````-XX:AllocateHeapAt` | `-XX:ZAllocationSpikeTolerance``-XX:ZCollectionInterval``-XX:ZFragmentationLimit``-XX:ZMarkStackSpaceLimit``-XX:ZProactive``-XX:ZUncommit``-XX:ZUncommitDelay` | `-XX:ZStatisticsInterval``-XX:ZVerifyForwarding``-XX:ZVerifyMarking``-XX:ZVerifyObjects``-XX:ZVerifyRoots``-XX:ZVerifyViews` |