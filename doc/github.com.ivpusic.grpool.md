
# 原理

* Pool 中有两个 channel: worker 和 job，初始化 Pool 时会创建一个 dispatcher，dispatcher 会创建指定数目的 worker，每个 worker 独占一个 goroutine。
* worker 有自己的 job channel，它先将自己放入 Pool.workerChannel 中，当自己的 job channel 有 Job 时就会执行它，执行完再次将自己放入 Pool.workerChannel 中。
* dispatcher 每次从 Pool.jobChannel 中拿一个 Job， 再从 Pool.workerChannel 中拿一个空闲的 worker，交给它执行（放入worker的jobChannel）