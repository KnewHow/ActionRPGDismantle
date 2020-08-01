# 伤害计算

## 1 RPGDamageStatics

```c++
struct RPGDamageStatics
{
	// 定义属性追捕，下面分别是 防御力，攻击力和伤害
	DECLARE_ATTRIBUTE_CAPTUREDEF(DefensePower);
	DECLARE_ATTRIBUTE_CAPTUREDEF(AttackPower);
	DECLARE_ATTRIBUTE_CAPTUREDEF(Damage);
	
	// 构造函数
	RPGDamageStatics()
	{
		// Capture the Target's DefensePower attribute. Do not snapshot it, 
		// because we want to use the health value at the moment we apply the execution.
		// 追踪目标防御力，不会存储他的快照，因为我们想使用实时的防御力来计算血量
		DEFINE_ATTRIBUTE_CAPTUREDEF(URPGAttributeSet, DefensePower, Target, false);

		// Capture the Source's AttackPower. We do want to snapshot this at the moment 
		// we create the GameplayEffectSpec that will execute the damage.
		// (imagine we fire a projectile: we create the GE Spec when the projectile is fired. 
		// When it hits the target, we want to use the AttackPower at the moment
		// the projectile was launched, not when it hits).
		// 追踪施法者的攻击力，我们将会使用施法者释放技能那一刻的攻击力作为计算。举个例子，我们发射
		// 一颗火球，我们会用发射时角色的攻击力来计算，而不是命中目标时角色的攻击力。
		DEFINE_ATTRIBUTE_CAPTUREDEF(URPGAttributeSet, AttackPower, Source, true);

		// Also capture the source's raw Damage, which is normally passed 
		// in directly via the execution
		// 也需要去追踪角色是原生伤害，这个伤害是被正常的传递到执行器中
		DEFINE_ATTRIBUTE_CAPTUREDEF(URPGAttributeSet, Damage, Source, true);
	}
};

// 获取静态伤害计算
static const RPGDamageStatics& DamageStatics()
{
	static RPGDamageStatics DmgStatics;
	return DmgStatics;
}

// 伤害执行的构造函数，将和伤害相关的属性添加进去
URPGDamageExecution::URPGDamageExecution()
{
	RelevantAttributesToCapture.Add(DamageStatics().DefensePowerDef);
	RelevantAttributesToCapture.Add(DamageStatics().AttackPowerDef);
	RelevantAttributesToCapture.Add(DamageStatics().DamageDef);
}
```

## 2 URPGDamageExecution

```c++
// Copyright 1998-2019 Epic Games, Inc. All Rights Reserved.

#pragma once

#include "ActionRPG.h"
#include "GameplayEffectExecutionCalculation.h"
#include "RPGDamageExecution.generated.h"

/**
 * A damage execution, which allows doing damage by combining a raw Damage number 
 * with AttackPower and DefensePower
 * Most games will want to implement multiple game-specific executions
 * 一个执行伤害计算的类，这将会对原生的伤害做一些关于攻击力和防御力的处理。
 * 对于大多数游戏来说，伤害的计算要更加复杂。
 */
UCLASS()
class ACTIONRPG_API URPGDamageExecution : public UGameplayEffectExecutionCalculation
{
	GENERATED_BODY()

public:
	// Constructor and overrides
	URPGDamageExecution();
	virtual void Execute_Implementation(
		const FGameplayEffectCustomExecutionParameters& ExecutionParams, 
		OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;

};
```



```c++
// 伤害执行的构造函数，将和伤害相关的属性添加进去
URPGDamageExecution::URPGDamageExecution()
{
	RelevantAttributesToCapture.Add(DamageStatics().DefensePowerDef);
	RelevantAttributesToCapture.Add(DamageStatics().AttackPowerDef);
	RelevantAttributesToCapture.Add(DamageStatics().DamageDef);
}

void URPGDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
	// 目标能力系统
	UAbilitySystemComponent* TargetAbilitySystemComponent = ExecutionParams.GetTargetAbilitySystemComponent();
	// 施法者能力系统
	UAbilitySystemComponent* SourceAbilitySystemComponent = ExecutionParams.GetSourceAbilitySystemComponent();

	// 施法者的 Actor
	AActor* SourceActor = SourceAbilitySystemComponent ? SourceAbilitySystemComponent->AvatarActor : nullptr;
	// 目标者的 Actor
	AActor* TargetActor = TargetAbilitySystemComponent ? TargetAbilitySystemComponent->AvatarActor : nullptr;

	// 游戏效果
	const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

	// Gather the tags from the source and target as that can affect which buffs should be used
	// 收集施法者和目标者的标签，这些标签可以影响某些增益
	const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
	const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

	FAggregatorEvaluateParameters EvaluationParameters;
	EvaluationParameters.SourceTags = SourceTags;
	EvaluationParameters.TargetTags = TargetTags;

	// --------------------------------------
	//	Damage Done = Damage * AttackPower / DefensePower
	//	If DefensePower is 0, it is treated as 1.0
	// --------------------------------------

	// 伤害计算公式：真实伤害 = 基本伤害 * 施法者攻击力 / 目标防御力
	// 如果防御力为0，就视为1

	// 获取防御力
	float DefensePower = 0.f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().DefensePowerDef, EvaluationParameters, DefensePower);
	if (DefensePower == 0.0f)
	{
		DefensePower = 1.0f;
	}

	// 获取攻击力
	float AttackPower = 0.f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().AttackPowerDef, EvaluationParameters, AttackPower);

	// 获取基本伤害
	float Damage = 0.f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().DamageDef, EvaluationParameters, Damage);
	
	// 计算真实伤害并输出
	float DamageDone = Damage * AttackPower / DefensePower;
	if (DamageDone > 0.f)
	{
		OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(DamageStatics().DamageProperty, EGameplayModOp::Additive, DamageDone));
	}
}
```

