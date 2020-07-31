# RPGGameplayAblity 剩下方法的实现

## 1 MakeEffectContainerSpecFromContainer

```c++
/** 
	* Make gameplay effect container spec to be applied later, using the passed in container
	* 产生一个[游戏效果规格]通过参数中的Container，这个规格将稍后被应用
	*/
	UFUNCTION(BlueprintCallable, Category = Ability, meta=(AutoCreateRefTerm = "EventData"))
	virtual FRPGGameplayEffectContainerSpec MakeEffectContainerSpecFromContainer(
		const FRPGGameplayEffectContainer& Container, 
		const FGameplayEventData& EventData, 
		int32 OverrideGameplayLevel = -1
	);
```

```c++
FRPGGameplayEffectContainerSpec URPGGameplayAbility::MakeEffectContainerSpecFromContainer(
	const FRPGGameplayEffectContainer& Container, 
	const FGameplayEventData& EventData, int32 OverrideGameplayLevel
)
{
	FRPGGameplayEffectContainerSpec ReturnSpec;
	// 获取 ARPGCharacterBase 并继续获取 URPGAbilitySystemComponent
	AActor* OwningActor = GetOwningActorFromActorInfo();
	ARPGCharacterBase* OwningCharacter = Cast<ARPGCharacterBase>(OwningActor);
	URPGAbilitySystemComponent* OwningASC = URPGAbilitySystemComponent::
		GetAbilitySystemComponentFromActor(OwningActor);

	if (OwningASC)
	{
		// If we have a target type, run the targeting logic. This is optional, targets can be added later
		// 如果已经有目标了，就立刻执行。这里的执行是可选的，目标可以稍后再添加上
		if (Container.TargetType.Get())
		{
			TArray<FHitResult> HitResults;
			TArray<AActor*> TargetActors;
			const URPGTargetType* TargetTypeCDO = Container.TargetType.GetDefaultObject();
			AActor* AvatarActor = GetAvatarActorFromActorInfo();
			TargetTypeCDO->GetTargets(OwningCharacter, AvatarActor, EventData, HitResults, TargetActors);
			ReturnSpec.AddTargets(HitResults, TargetActors);
		}

		// If we don't have an override level, use the default on the ability itself
		// 如果没有重写 OverrideGameplayLevel，使用默认的 OverrideGameplayLevel
		if (OverrideGameplayLevel == INDEX_NONE)
		{
			OverrideGameplayLevel = OverrideGameplayLevel = this->GetAbilityLevel(); //OwningASC->GetDefaultAbilityLevel();
		}

		// Build GameplayEffectSpecs for each applied effect
		// 从每一个需要应用的效果中构建 GameplayEffectSpecs
		for (const TSubclassOf<UGameplayEffect>& EffectClass : Container.TargetGameplayEffectClasses)
		{
			ReturnSpec.TargetGameplayEffectSpecs.Add(MakeOutgoingGameplayEffectSpec(EffectClass, OverrideGameplayLevel));
		}
	}
	return ReturnSpec;
}
```

## 2 MakeEffectContainerSpec 

```c++
/** 
	* Search for and make a gameplay effect container spec to be applied later, 
	* from the EffectContainerMap 
	* 从 EffectContainerMap 查找 FRPGGameplayEffectContainer，如果合法，上面的方法来产生效果
	*/
	UFUNCTION(BlueprintCallable, Category = Ability, meta = (AutoCreateRefTerm = "EventData"))
	virtual FRPGGameplayEffectContainerSpec MakeEffectContainerSpec(
		FGameplayTag ContainerTag, 
		const FGameplayEventData& EventData, 
		int32 OverrideGameplayLevel = -1);
```

```c++
FRPGGameplayEffectContainerSpec URPGGameplayAbility::MakeEffectContainerSpec(
	FGameplayTag ContainerTag, 
	const FGameplayEventData& EventData, 
	int32 OverrideGameplayLevel)
{
	// 通过 FGameplayTag 查找 FRPGGameplayEffectContainer，然后产生效果
	FRPGGameplayEffectContainer* FoundContainer = EffectContainerMap.Find(ContainerTag);

	if (FoundContainer)
	{
		return MakeEffectContainerSpecFromContainer(*FoundContainer, EventData, OverrideGameplayLevel);
	}
	return FRPGGameplayEffectContainerSpec();
}
```

## 3 ApplyEffectContainerSpec

```c++
/** 
	* Applies a gameplay effect container spec that was previously created
	* 将之前被创建的效果应用到游戏效果中
	*/
	UFUNCTION(BlueprintCallable, Category = Ability)
	virtual TArray<FActiveGameplayEffectHandle> ApplyEffectContainerSpec(
		const FRPGGameplayEffectContainerSpec& ContainerSpec);
```

```c++
TArray<FActiveGameplayEffectHandle> URPGGameplayAbility::ApplyEffectContainerSpec(const FRPGGameplayEffectContainerSpec& ContainerSpec)
{
	TArray<FActiveGameplayEffectHandle> AllEffects;

	// Iterate list of effect specs and apply them to their target data
	// 迭代所有的特效并且应用他们到目标数据
	for (const FGameplayEffectSpecHandle& SpecHandle : ContainerSpec.TargetGameplayEffectSpecs)
	{
		AllEffects.Append(K2_ApplyGameplayEffectSpecToTarget(SpecHandle, ContainerSpec.TargetData));
	}
	return AllEffects;
}
```

## 4 ApplyEffectContainer

```c++
/**
	* Applies a gameplay effect container, by creating and then applying the spec 
	* 通过 FGameTag 查找 FRPGGameplayEffectContainerSpec，然后应用它
	*/
	UFUNCTION(BlueprintCallable, Category = Ability, meta = (AutoCreateRefTerm = "EventData"))
	virtual TArray<FActiveGameplayEffectHandle> ApplyEffectContainer(
		FGameplayTag ContainerTag, 
		const FGameplayEventData& EventData, 
		int32 OverrideGameplayLevel = -1);
```

```c++
TArray<FActiveGameplayEffectHandle> URPGGameplayAbility::ApplyEffectContainer(FGameplayTag ContainerTag, const FGameplayEventData& EventData, int32 OverrideGameplayLevel)
{
	// 通过 GameTag 查找 FRPGGameplayEffectContainerSpec，然后应用它
	FRPGGameplayEffectContainerSpec Spec = MakeEffectContainerSpec(ContainerTag, EventData, OverrideGameplayLevel);
	return ApplyEffectContainerSpec(Spec);
}
```

