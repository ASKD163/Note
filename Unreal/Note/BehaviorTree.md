#  行为树
## 行为树加载过程

两种方式运行行为树

- `AAIController::RunBehaviorTree(UBehaviorTree* BTAsset)`

- `Run Behavior` 任务节点，在一颗行为树；里加载运行另一棵；`UBTTask_RunBehaviorDynamic` 允许动态在运行时添加子树 (`SetDynamicSubtree`)

![](https://raw.githubusercontent.com/ASKD163/Note/main/Unreal/Pic/BehaviorTree/20230510105655.png?token=AHZV7HLE3YCA7UKPQBDUGDDELMEDI)

### 大致流程

1. 执行`Run Behavior`  Task

   ```C++
   	const bool bPushed = BehaviorAsset != nullptr && OwnerComp.PushInstance(*BehaviorAsset);
   	if (bPushed && OwnerComp.InstanceStack.Num() > 0)
   	{
   		FBehaviorTreeInstance& MyInstance = OwnerComp.InstanceStack[OwnerComp.InstanceStack.Num() - 1];
   		MyInstance.DeactivationNotify.BindUObject(this, &UBTTask_RunBehavior::OnSubtreeDeactivated);
   		return EBTNodeResult::InProgress;
   	}
   ```

   > 将自己的 `UBTTask_RunBehavior::OnSubtreeDeactivated` 函数绑定到子树的 `DeactivationNotify` 代理上

   `UBehaviorTreeComponent::PushInstance(UBehaviorTree& TreeAsset)` 将行为树推入执行栈 (execution Stack)

2. 准备执行树前的检查

   检查黑板类是否匹配，子树黑板需跟父树兼容 (Compitable)，同一资产或键值相同 (子树黑板可以是父树黑板的parent，但父树黑板不能是子树的Parent)

   BehaviorTreeManager能否获取，

   父节点是否允许子树运行，`UBTCompositeNode::CanPushSubtree` 默认返回True，唯一例外 SimpleParallel `return (ChildIdx != EBTParallelChild::MainTask);`

    

3. 加载树 `UBehaviorTreeManager::LoadTree`


   已加载的树会存储在 `LoadedTemplates` (`TArray<FBehaviorTreeTemplateInfo>`)

   

   BehaviorTreeManager 用 `LoadedTemplates` 缓存已经加载的树，加载树时先检查该树是否已经缓存，已加载时直接更改 Root 指向，修改树内存大小

   加载新树，初始化树节点，获取每个节点的父节点、执行顺序、内存偏移量、树深度 `UBTNode::InitializeNode(UBTCompositeNode* InParentNode, uint16 InExecutionIndex, uint16 InMemoryOffset, uint8 InTreeDepth)`

   创建 `FBehaviorTreeTemplateInfo` 结构体

​    

   `FNodeInitializationData` 

   ```C++
   	UBTNode* Node;  // 节点
   	UBTCompositeNode* ParentNode; // 父节点 (sequence, selector)
   	uint16 ExecutionIndex; // 执行顺序
   	uint16 DataSize; // 内存大小
   	uint16 SpecialDataSize; // Special 内存大小
   	uint8 TreeDepth; // 数深度
   ```

   `SpecialDataSize` 节点Meory的内存大小，若节点创建NodeInstance则为0，**默认所有相同节点共用一个Instance，各自需要的参数存储在 NodeMemory里** ， 

   通过 `InitializeNodeHelper` 初始化输入 树的每个节点，深度优先 按执行顺序将每个节点添加到 `InitList`

   从父节点 (CompositeNode)开始，自身的Service，子节点的Decorator，单独处理RunBehavior Task 的Decorator 标记为 InjectedNode；main difference `Decorator->MarkInjectedNode();`

   子节点如果是CompositeNode 继续递归处理子节点

   ```C++
   			UBTNode* ChildNode = NULL;
   			
   			if (ChildInfo.ChildComposite)
   			{
   				ChildInfo.ChildComposite = Cast<UBTCompositeNode>(StaticDuplicateObject(ChildInfo.ChildComposite, NodeOuter));
   				ChildNode = ChildInfo.ChildComposite;
   			}
   			...
               if (ChildNode)
               {
                   InitializeNodeHelper(CompositeOb, ChildNode, TreeDepth + 1, ExecutionIndex, InitList, TreeAsset, NodeOuter);
               }
   ```

   子节点 Server，子节点Task

   > 父节点 (Composite) 本身
   >
   > 父节点 (Composite) 自身Service
   >
   > 子节点 Decorator
   >
   > if (子节点 == Composite)
   >
   > ​	`InitializeNodeHelper(子节点)` 
   >
   > else
   >
   > ​	子节点 Server
   >
   > ​	子节点Task



根节点Decorator的加载，根节点Decorator执行顺序为 -1 ，该Decorator只在Run Behavior Task执行时有效

![](https://raw.githubusercontent.com/ASKD163/Note/main/Unreal/Pic/BehaviorTree/20230511114018.png)



![](https://raw.githubusercontent.com/ASKD163/Note/main/Unreal/Pic/BehaviorTree/20230511114227.png)


4.  **计算内存**  `LoadTree`

​		得到节点的内存偏移量，方便之后读取节点内存 FNodeMemory

​	 为什么排序？()

```C++
		InitList.Sort(FNodeInitializationData::FMemorySort());
		uint16 MemoryOffset = 0;
		for (int32 Index = 0; Index < InitList.Num(); Index++)
		{
			InitList[Index].Node->InitializeNode(InitList[Index].ParentNode, InitList[Index].ExecutionIndex, InitList[Index].SpecialDataSize + MemoryOffset, InitList[Index].TreeDepth);
			MemoryOffset += InitList[Index].DataSize;
		}
		
		TemplateInfo.InstanceMemorySize = MemoryOffset;
```



5. `UBehaviorTreeComponent::PushInstance`

   初始化内存以及node instance

   完成 FBehaviorTreeInstance 的设置，入栈

   执行RootNode的server，开始新树的执行 

   `FBehaviorTreeDelegates::OnTreeStarted.Broadcast(*this, TreeAsset);`  树启动事件

   

`FBehaviorTreeInstanceId` 树实例的ID标识，树资产与被执行路径(GetExecutionIndex)相同表示树实例相同
