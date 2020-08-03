# URPGGameplayAbility

声明：

```c++
// Copyright 1998-2019 Epic Games, Inc. All Rights Reserved.

#pragma once

#include "ActionRPG.h"
#include "AbilitySystemComponent.h"
#include "Abilities/RPGAbilityTypes.h"
#include "RPGAbilitySystemComponent.generated.h"

class URPGGameplayAbility;

/**
 * Subclass of ability system component with game-specific data
 * Most games will need to make a game-specific subclass to provide utility functions
 * 
 * 一个能力系统子类用来提供游戏数据，大多数游戏都需要这样的一个子类来提供功能函数
 * 
 */
UCLASS()
class ACTIONRPG_API URPGAbilitySystemComponent : public UAbilitySystemComponent
{
	GENERATED_BODY()

public:
	// Constructors and overrides
	// 构造函数重载
	URPGAbilitySystemComponent();

	/** 
	* Returns a list of currently active ability instances that match the tags 
	* 获取当前标签匹配的能力列表
	*/
	void GetActiveAbilitiesWithTags(const FGameplayTagContainer& GameplayTagContainer, TArray<URPGGameplayAbility*>& ActiveAbilities);

	/** 
	* Returns the default level used for ability activations, derived from the character 
	* 获取默认的能力等级，这些等级基于角色的
	*/
	int32 GetDefaultAbilityLevel() const;

	/** 
	* Version of function in AbilitySystemGlobals that returns correct type 
	*  从 Actor 中获取能力系统组件
	*/
	static URPGAbilitySystemComponent* GetAbilitySystemComponentFromActor(const AActor* Actor, bool LookForComponent = false);

};
```



实现：

```c++
// Copyright 1998-2019 Epic Games, Inc. All Rights Reserved.

#include "Abilities/RPGAbilitySystemComponent.h"
#include "RPGCharacterBase.h"
#include "Abilities/RPGGameplayAbility.h"
#include "AbilitySystemGlobals.h"

URPGAbilitySystemComponent::URPGAbilitySystemComponent() {}

void URPGAbilitySystemComponent::GetActiveAbilitiesWithTags(
	const FGameplayTagContainer& GameplayTagContainer, 
	TArray<URPGGameplayAbility*>& ActiveAbilities
){
	// 获取所有可激活的能力系统组件
	TArray<FGameplayAbilitySpec*> AbilitiesToActivate;
	// 此方法中的 ActivatableAbilities 实在 CharacterBase 中调用 AbilitySystemComponent->GiveAbility 设置的
	GetActivatableGameplayAbilitySpecsByAllMatchingTags(GameplayTagContainer, AbilitiesToActivate, false);

	// Iterate the list of all ability specs
	// 迭代所有的能力系统
	for (FGameplayAbilitySpec* Spec : AbilitiesToActivate)
	{
		// Iterate all instances on this ability spec
		// 迭代所有能力系统规格实例
		TArray<UGameplayAbility*> AbilityInstances = Spec->GetAbilityInstances();

		for (UGameplayAbility* ActiveAbility : AbilityInstances)
		{
			ActiveAbilities.Add(Cast<URPGGameplayAbility>(ActiveAbility));
		}
	}
}

int32 URPGAbilitySystemComponent::GetDefaultAbilityLevel() const
{
	// 默认返回角色等级
	ARPGCharacterBase* OwningCharacter = Cast<ARPGCharacterBase>(OwnerActor);

	if (OwningCharacter)
	{
		return OwningCharacter->GetCharacterLevel();
	}
	return 1;
}

URPGAbilitySystemComponent* URPGAbilitySystemComponent::GetAbilitySystemComponentFromActor(const AActor* Actor, bool LookForComponent)
{
	// 调用底层方法来获取能力系统
	return Cast<URPGAbilitySystemComponent>(UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Actor, LookForComponent));
}

```

