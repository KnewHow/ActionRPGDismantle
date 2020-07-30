# URPGTargetType子类和 FRPGGameplayEffectContainerSpec 的方法实现

## 1 URPGTargetType_UseOwner

```c++
/** 
* Trivial target type that uses the owner 
* 不重要的目标
* 
*/
UCLASS(NotBlueprintable)
class ACTIONRPG_API URPGTargetType_UseOwner : public URPGTargetType
{
	GENERATED_BODY()

public:
	// Constructor and overrides
	URPGTargetType_UseOwner() {}

	/** Uses the passed in event data */
	virtual void GetTargets_Implementation(ARPGCharacterBase* TargetingCharacter, AActor* TargetingActor, FGameplayEventData EventData, TArray<FHitResult>& OutHitResults, TArray<AActor*>& OutActors) const override;
};
```

```c++
// 将目标角色添加到外部Actors
void URPGTargetType_UseOwner::GetTargets_Implementation(
	ARPGCharacterBase* TargetingCharacter, 
	AActor* TargetingActor, 
	FGameplayEventData EventData, 
	TArray<FHitResult>& OutHitResults, 
	TArray<AActor*>& OutActors) const
{
	OutActors.Add(TargetingCharacter);
}

```



## 2 URPGTargetType_UseEventData

```c++
/** 
* Trivial target type that pulls the target out of the event data 
* 从事件数据中拉取的不重要的目标
*/
UCLASS(NotBlueprintable)
class ACTIONRPG_API URPGTargetType_UseEventData : public URPGTargetType
{
	GENERATED_BODY()

public:
	// Constructor and overrides
	URPGTargetType_UseEventData() {}

	/** Uses the passed in event data */
	virtual void GetTargets_Implementation(ARPGCharacterBase* TargetingCharacter, AActor* TargetingActor, FGameplayEventData EventData, TArray<FHitResult>& OutHitResults, TArray<AActor*>& OutActors) const override;
};

```

```c++
void URPGTargetType_UseEventData::GetTargets_Implementation(
	ARPGCharacterBase* TargetingCharacter, 
	AActor* TargetingActor, 
	FGameplayEventData EventData, 
	TArray<FHitResult>& OutHitResults, 
	TArray<AActor*>& OutActors) const
{
	// 如果有碰撞就添加碰撞信息，否则把 EventData 添加到外部Actors
	const FHitResult* FoundHitResult = EventData.ContextHandle.GetHitResult();
	if (FoundHitResult)
	{
		OutHitResults.Add(*FoundHitResult);
	}
	else if (EventData.Target)
	{
		OutActors.Add(const_cast<AActor*>(EventData.Target));
	}
}
```

## 3 FGameplayAbilityTargetDataHandle

```c++
/**
*	Handle for Targeting Data. This servers two main purposes:
*		-Avoid us having to copy around the full targeting data structure in Blueprints
*		-Allows us to leverage polymorphism in the target data structure
*		-Allows us to implement NetSerialize and replicate by value between clients/server
*
*		-Avoid using UObjects could be used to give us polymorphism and by reference passing in blueprints.
*		-However we would still be screwed when it came to replication
*
*		-Replication by value
*		-Pass by reference in blueprints
*		-Polymophism in TargetData structure
* 
* 处理目标数据的一个类，主要有两个目的：
*	- 避免在蓝图中复制一些和目标相关的数据
*	- 允许我们在目标数据结构体中使用多态
*	- 允许我们在客户端或者服务器实现一些值类型变量的序列化和复制操作
*
* 	- 避免使用 UObject类型，即使它可以作为多态的父类
*	- 因为当复制的时候，我们的对象可能仍然是螺旋的。
*	
* 	- 通过值类型复制
* 	- 在蓝图中通过引用传递
* 	- 在目标的数据中可以以多态的形式存在
*/
USTRUCT(BlueprintType)
struct GAMEPLAYABILITIES_API FGameplayAbilityTargetDataHandle
{
	GENERATED_USTRUCT_BODY()

	FGameplayAbilityTargetDataHandle() { }
	FGameplayAbilityTargetDataHandle(FGameplayAbilityTargetData* DataPtr)
	{
		Data.Add(TSharedPtr<FGameplayAbilityTargetData>(DataPtr));
	}

	FGameplayAbilityTargetDataHandle(FGameplayAbilityTargetDataHandle&& Other) : UniqueId(Other.UniqueId), Data(MoveTemp(Other.Data))	{ }
	FGameplayAbilityTargetDataHandle(const FGameplayAbilityTargetDataHandle& Other) : UniqueId(Other.UniqueId), Data(Other.Data) { }

	FGameplayAbilityTargetDataHandle& operator=(FGameplayAbilityTargetDataHandle&& Other) { UniqueId = Other.UniqueId; Data = MoveTemp(Other.Data); return *this; }
	FGameplayAbilityTargetDataHandle& operator=(const FGameplayAbilityTargetDataHandle& Other) { UniqueId = Other.UniqueId; Data = Other.Data; return *this; }

	uint8 UniqueId = 0;

	/** Raw storage of target data, do not modify this directly */
	TArray<TSharedPtr<FGameplayAbilityTargetData>, TInlineAllocator<1> >	Data;

	/** Resets handle to have no targets */
	void Clear()
	{
		Data.Reset();
	}

	/** Returns number of target data, not number of actors/targets as target data may contain multiple actors */
	int32 Num() const
	{
		return Data.Num();
	}

	/** Returns true if there are any valid targets */
	bool IsValid(int32 Index) const
	{
		return (Index < Data.Num() && Data[Index].IsValid());
	}

	/** Returns data at index, or nullptr if invalid */
	const FGameplayAbilityTargetData* Get(int32 Index) const
	{
		return IsValid(Index) ? Data[Index].Get() : nullptr;
	}

	/** Returns data at index, or nullptr if invalid */
	FGameplayAbilityTargetData* Get(int32 Index)
	{
		return IsValid(Index) ? Data[Index].Get() : nullptr;
	}

	/** Adds a new target data to handle, it must have been created with new */
	void Add(FGameplayAbilityTargetData* DataPtr)
	{
		Data.Add(TSharedPtr<FGameplayAbilityTargetData>(DataPtr));
	}

	/** Does a shallow copy of target data from one handle to another */
	void Append(const FGameplayAbilityTargetDataHandle& OtherHandle)
	{
		Data.Append(OtherHandle.Data);
	}

	/** Serialize for networking, handles polymorphism */
	bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);

	/** Comparison operator */
	bool operator==(const FGameplayAbilityTargetDataHandle& Other) const
	{
		// Both invalid structs or both valid and Pointer compare (???) // deep comparison equality
		if (Data.Num() != Other.Data.Num())
		{
			return false;
		}
		for (int32 i = 0; i < Data.Num(); ++i)
		{
			if (Data[i].IsValid() != Other.Data[i].IsValid())
			{
				return false;
			}
			if (Data[i].Get() != Other.Data[i].Get())
			{
				return false;
			}
		}
		return true;
	}

	/** Comparison operator */
	bool operator!=(const FGameplayAbilityTargetDataHandle& Other) const
	{
		return !(FGameplayAbilityTargetDataHandle::operator==(Other));
	}
};
```



而这里面最重要的是：

```c++
	/** Raw storage of target data, do not modify this directly */
	// 存储了一个原生的目标数据（数组类型），但是不推荐直接的修改它
	TArray<TSharedPtr<FGameplayAbilityTargetData>, TInlineAllocator<1> >	Data;
```

他的操作基本上和数组有些类似，其实就算再判断里面的`Data` 是否有效。

## 4 FRPGGameplayEffectContainerSpec 的一些实现

```c++
bool FRPGGameplayEffectContainerSpec::HasValidEffects() const
{
	// effects 大于0 表示有合法的效果
	return TargetGameplayEffectSpecs.Num() > 0;
}

bool FRPGGameplayEffectContainerSpec::HasValidTargets() const
{
	// 使用 TargetData 中的 Data 做判断
	return TargetData.Num() > 0;
}

void FRPGGameplayEffectContainerSpec::AddTargets(
	const TArray<FHitResult>& HitResults, 
	const TArray<AActor*>& TargetActors
){
	// 将所有的碰撞事件添加到目标数据中
	for (const FHitResult& HitResult : HitResults)
	{
		FGameplayAbilityTargetData_SingleTargetHit* NewData = new FGameplayAbilityTargetData_SingleTargetHit(HitResult);
		TargetData.Add(NewData);
	}

	// 如果目光 Actor 也有数据，也把对应的数据添加进去
	if (TargetActors.Num() > 0)
	{
		FGameplayAbilityTargetData_ActorArray* NewData = new FGameplayAbilityTargetData_ActorArray();
		NewData->TargetActorArray.Append(TargetActors);
		TargetData.Add(NewData);
	}
}
```



