# Flink

> Flink 

## 需要关注的点

- 运维
	- 升级/重启
	- 数据核对
- 部署
	- Yarn
- 运行时
	- 失败重试
	- exactly once
	- 丢失数据后补救措施
	
## 待验证

- checkpoint

	```
		public JobGraph getJobGraph() {
		// temporarily forbid checkpointing for iterative jobs
		if (isIterative() && checkpointConfig.isCheckpointingEnabled() && !checkpointConfig.isForceCheckpointing()) {
			throw new UnsupportedOperationException(
					"Checkpointing is currently not supported by default for iterative jobs, as we cannot guarantee exactly once semantics. "
							+ "State checkpoints happen normally, but records in-transit during the snapshot will be lost upon failure. "
							+ "\nThe user can force enable state checkpoints with the reduced guarantees by calling: env.enableCheckpointing(interval,true)");
		}

		return StreamingJobGraphGenerator.createJobGraph(this);
	}
	```
	
- savepoint

- TwoPhaseCommitSinkFunction

- EXACTLY_ONCE

- StatefulFunction

