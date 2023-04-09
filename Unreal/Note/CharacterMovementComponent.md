# Character Movement Component

>  Ref:  https://zhuanlan.zhihu.com/p/368613242
>
>  ​	     https://zhuanlan.zhihu.com/p/34257208



计算位置更新 *UpdatedComponent的位置



网络同步， 移动状态更新（flying, walking, ...），ROV避障

更改角色移动的两种：1. 给予初速度，2. 按 rootmotion 移动

```mermaid
classDiagram 
class UMovementComponent{
	+ UpdatedComponent
	+ MoveUpdatedComponent():bool
	# MoveUpdatedComponentImpl()
	+SafeMoveUpdatedComponent()
}
class USceneComponent{
	+MoveComponent()
	#InternalSetworldLocationAndRotation()
	- UpdateComponentToWorldWithParent()
}
class UNaveMovementComopnent{
	- NavAgentProps
}
UObject <|-- UActorComponent
UActorComponent <|-- UMovementComponent
UPawnMovementComponent <|-- UCharacterMovementComponent
UNaveMovementComopnent <|-- UPawnMovementComponent
UNaveMovementComopnent --|> UMovementComponent
USceneComponent --o UMovementComponent

```

## MovementComponet

作为移动组件的基类实现了基本的移动接口`SafeMoveUpdatedComponent`，更新UpdatedComponent 更新Actor的位置，

`MoveUpdatedCOmponent` 移动UpdatedComponent通过传入的Delta以及变更Rotation，调用`MoveUpdatedComponentImpl`

UpdatedComponent 类型为SceneComponent，该组件提供了最基本的位置信息，同时提供了改变自身以及其子组件的位置的接口InternalSetWorldLocationAndRotation()



## UNaveMovementComopnent

提供AI相关移动

··
