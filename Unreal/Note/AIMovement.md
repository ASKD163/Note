# AIMovement

## Nav

### Hint

- 行为树中的MoveTo节点下AllowPartialPath选项若勾选如果无法通过导航到达目标位置回返回最接近目标的路径；`FPathFindingQuery` 中该值默认为true。
- MoveTo节点中的 Accept Radius；Move 判断为成功时 Pawn 的 ActorLocation 到 TargetLocation 的距离等于  Accept Radius + Pawn 的Extend 的半径