# URPGAbilityTask_PlayMontageAndWaitForEvent-cpp件注释

```C++
// Copyright 1998-2019 Epic Games, Inc. All Rights Reserved.

#include "Abilities/RPGAbilityTask_PlayMontageAndWaitForEvent.h"
#include "Abilities/RPGAbilitySystemComponent.h"
#include "GameFramework/Character.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"
#include "Animation/AnimInstance.h"

URPGAbilityTask_PlayMontageAndWaitForEvent::URPGAbilityTask_PlayMontageAndWaitForEvent(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	// 设置播放比率为1，并且当能力结束时，自动结束蒙太奇
	Rate = 1.f;
	bStopWhenAbilityEnds = true;
}

URPGAbilitySystemComponent* URPGAbilityTask_PlayMontageAndWaitForEvent::GetTargetASC()
{
	// 将内置的变量转换为RPG 能力系统
	return Cast<URPGAbilitySystemComponent>(AbilitySystemComponent);
}

//蒙太奇混合
void URPGAbilityTask_PlayMontageAndWaitForEvent::OnMontageBlendingOut(
	UAnimMontage* Montage, 
	bool bInterrupted)
{
	// 获取能力和能力的蒙太奇，如果相同并且和参数中 Montage 相同，就设置根骨骼缩放
	if (Ability && Ability->GetCurrentMontage() == MontageToPlay)
	{
		if (Montage == MontageToPlay)
		{
			AbilitySystemComponent->ClearAnimatingAbility(Ability);

			// Reset AnimRootMotionTranslationScale
			ACharacter* Character = Cast<ACharacter>(GetAvatarActor());
			if (Character && (Character->GetLocalRole() == ROLE_Authority ||
							  (Character->GetLocalRole() == ROLE_AutonomousProxy && Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted)))
			{
				Character->SetAnimRootMotionTranslationScale(1.f);
			}

		}
	}
	// 对 bInterrupted 为 true 时，广播中断，否则广播混合
	if (bInterrupted)
	{
		if (ShouldBroadcastAbilityTaskDelegates())
		{
			OnInterrupted.Broadcast(FGameplayTag(), FGameplayEventData());
		}
	}
	else
	{
		if (ShouldBroadcastAbilityTaskDelegates())
		{
			OnBlendOut.Broadcast(FGameplayTag(), FGameplayEventData());
		}
	}
}

void URPGAbilityTask_PlayMontageAndWaitForEvent::OnAbilityCancelled()
{
	// TODO: Merge this fix back to engine, it was calling the wrong callback

	// 首先执行停止播放，如果停止成功，广播取消成功
	if (StopPlayingMontage())
	{
		// Let the BP handle the interrupt as well
		if (ShouldBroadcastAbilityTaskDelegates())
		{
			OnCancelled.Broadcast(FGameplayTag(), FGameplayEventData());
		}
	}
}

void URPGAbilityTask_PlayMontageAndWaitForEvent::OnMontageEnded(UAnimMontage* Montage, bool bInterrupted)
{
	// 如果没有被中断，广播完成事件，并且结束任务
	if (!bInterrupted)
	{
		if (ShouldBroadcastAbilityTaskDelegates())
		{
			OnCompleted.Broadcast(FGameplayTag(), FGameplayEventData());
		}
	}

	EndTask();
}

void URPGAbilityTask_PlayMontageAndWaitForEvent::OnGameplayEvent(FGameplayTag EventTag, const FGameplayEventData* Payload)
{
	// 广播事件接收到消息
	if (ShouldBroadcastAbilityTaskDelegates())
	{
		FGameplayEventData TempData = *Payload;
		TempData.EventTag = EventTag;

		EventReceived.Broadcast(EventTag, TempData);
	}
}

URPGAbilityTask_PlayMontageAndWaitForEvent* URPGAbilityTask_PlayMontageAndWaitForEvent::
	PlayMontageAndWaitForEvent(
		UGameplayAbility* OwningAbility,
		FName TaskInstanceName,
		UAnimMontage* MontageToPlay, 
		FGameplayTagContainer EventTags, 
		float Rate, 
		FName StartSection, 
		bool bStopWhenAbilityEnds, 
		float AnimRootMotionTranslationScale)
{
	// 应用播放比率，并且构建 URPGAbilityTask_PlayMontageAndWaitForEvent
	UAbilitySystemGlobals::NonShipping_ApplyGlobalAbilityScaler_Rate(Rate);

	URPGAbilityTask_PlayMontageAndWaitForEvent* MyObj = NewAbilityTask<URPGAbilityTask_PlayMontageAndWaitForEvent>(OwningAbility, TaskInstanceName);
	MyObj->MontageToPlay = MontageToPlay;
	MyObj->EventTags = EventTags;
	MyObj->Rate = Rate;
	MyObj->StartSection = StartSection;
	MyObj->AnimRootMotionTranslationScale = AnimRootMotionTranslationScale;
	MyObj->bStopWhenAbilityEnds = bStopWhenAbilityEnds;

	return MyObj;
}

void URPGAbilityTask_PlayMontageAndWaitForEvent::Activate()
{
	if (Ability == nullptr)
	{
		return;
	}

	bool bPlayedMontage = false;
	// 获取目标的能力系统组件
	URPGAbilitySystemComponent* RPGAbilitySystemComponent = GetTargetASC();

	if (RPGAbilitySystemComponent)
	{
		// 获取绑定当前能力系统的 Actor
		const FGameplayAbilityActorInfo* ActorInfo = Ability->GetCurrentActorInfo();
		// 获取 Actor 绑定的动画实例
		UAnimInstance* AnimInstance = ActorInfo->GetAnimInstance();
		if (AnimInstance != nullptr)
		{
			// Bind to event callback
			// 绑定回调事件
			EventHandle = RPGAbilitySystemComponent->AddGameplayEventTagContainerDelegate(
				EventTags, 
				FGameplayEventTagMulticastDelegate::FDelegate::CreateUObject(
					this, 
					&URPGAbilityTask_PlayMontageAndWaitForEvent::OnGameplayEvent
				)
			);

			// 播放蒙太奇动画成功
			if (RPGAbilitySystemComponent->PlayMontage(
					Ability,
					Ability->GetCurrentActivationInfo(), 
					MontageToPlay, 
					Rate, 
					StartSection)
				> 0.f
			)
			{
				// Playing a montage could potentially fire off a callback into game code which could kill this ability! 
				// Early out if we are  pending kill.
				// 播放一个蒙太奇动画可能会潜在的让回调事件哑火，因为游戏代码可能会杀死能力
				// 如果该任务正等待被结束，尽量尽早的返回
				if (ShouldBroadcastAbilityTaskDelegates() == false)
				{
					return;
				}

				// 绑定取消事件
				CancelledHandle = Ability->OnGameplayAbilityCancelled.AddUObject(
						this, 
						&URPGAbilityTask_PlayMontageAndWaitForEvent::OnAbilityCancelled
				);

				// 绑定混合事件
				BlendingOutDelegate.BindUObject(
					this, 
					&URPGAbilityTask_PlayMontageAndWaitForEvent::OnMontageBlendingOut
				);
				// 设置混合事件
				AnimInstance->Montage_SetBlendingOutDelegate(
					BlendingOutDelegate, MontageToPlay);

				// 绑定并且设置播放结束事件
				MontageEndedDelegate.BindUObject(
					this, 
					&URPGAbilityTask_PlayMontageAndWaitForEvent::OnMontageEnded
				);
				AnimInstance->Montage_SetEndDelegate(MontageEndedDelegate, MontageToPlay);

				// 设置角色蒙太奇动画播放比率
				ACharacter* Character = Cast<ACharacter>(GetAvatarActor());
				if (Character && (Character->GetLocalRole() == ROLE_Authority ||
								  (Character->GetLocalRole() == ROLE_AutonomousProxy && 
									  Ability->GetNetExecutionPolicy() == 
										EGameplayAbilityNetExecutionPolicy::LocalPredicted)))
				{
					Character->SetAnimRootMotionTranslationScale(AnimRootMotionTranslationScale);
				}
				// 已经播放过蒙太奇了
				bPlayedMontage = true;
			}
		}
		else
		{
			ABILITY_LOG(Warning, TEXT("URPGAbilityTask_PlayMontageAndWaitForEvent call to PlayMontage failed!"));
		}
	}
	else
	{
		// 打印错误日志
		ABILITY_LOG(Warning, TEXT("URPGAbilityTask_PlayMontageAndWaitForEvent called on invalid AbilitySystemComponent"));
	}

	if (!bPlayedMontage)
	{
		// 打印日志并广播取消
		ABILITY_LOG(Warning, TEXT("URPGAbilityTask_PlayMontageAndWaitForEvent called in Ability %s failed to play montage %s; Task Instance Name %s."), *Ability->GetName(), *GetNameSafe(MontageToPlay),*InstanceName.ToString());
		if (ShouldBroadcastAbilityTaskDelegates())
		{
			OnCancelled.Broadcast(FGameplayTag(), FGameplayEventData());
		}
	}

	SetWaitingOnAvatar();
}

void URPGAbilityTask_PlayMontageAndWaitForEvent::ExternalCancel()
{
	// 外部取消
	check(AbilitySystemComponent);

	OnAbilityCancelled();

	Super::ExternalCancel();
}

void URPGAbilityTask_PlayMontageAndWaitForEvent::OnDestroy(bool AbilityEnded)
{
	// Note: Clearing montage end delegate isn't necessary since its not a multicast 
	// and will be cleared when the next montage plays.
	// (If we are destroyed, it will detect this and not do anything)
	// 记住：清除蒙太奇的结束会话不是很有必要的因为他是不是一个多播代练，
	// 并且这个多播代理也会被清除，当下个蒙太奇播放的时候。
	// 如果我们执行销毁，他将会侦测此任务，并且不会做任何事情

	// This delegate, however, should be cleared as it is a multicast
	// 当这个会话是多播的时候，应该被清理
	if (Ability)
	{
		// 移除取消的句柄
		Ability->OnGameplayAbilityCancelled.Remove(CancelledHandle);
		// 如果需要结束，停止蒙太奇播放
		if (AbilityEnded && bStopWhenAbilityEnds)
		{
			StopPlayingMontage();
		}
	}

	// 获取目标能力系统，然后移除
	URPGAbilitySystemComponent* RPGAbilitySystemComponent = GetTargetASC();
	if (RPGAbilitySystemComponent)
	{
		RPGAbilitySystemComponent->RemoveGameplayEventTagContainerDelegate(EventTags, EventHandle);
	}
	// 根据说明，需要调用父类方法
	Super::OnDestroy(AbilityEnded);

}

// 停止正在播放的蒙太奇
bool URPGAbilityTask_PlayMontageAndWaitForEvent::StopPlayingMontage()
{
	// 获取当前的 Actor
	const FGameplayAbilityActorInfo* ActorInfo = Ability->GetCurrentActorInfo();
	if (!ActorInfo)
	{
		return false;
	}

	// 获取当前的动画实例
	UAnimInstance* AnimInstance = ActorInfo->GetAnimInstance();
	if (AnimInstance == nullptr)
	{
		return false;
	}

	// Check if the montage is still playing
	// 检查当前的蒙太奇是否在播放
	// The ability would have been interrupted, in which case we should automatically 
	// stop the montage
	// 这个能力系统将会被中断，这里我们相应的要自动停止蒙太奇的播放
	if (AbilitySystemComponent && Ability)
	{
		// 对当前Actor的能力系统和蒙太奇做判断
		if (AbilitySystemComponent->GetAnimatingAbility() == Ability
			&& AbilitySystemComponent->GetCurrentMontage() == MontageToPlay)
		{
			// Unbind delegates so they don't get called as well
			// 获取蒙太奇动画实例，并且解绑 BlendingOut 和 Ended
			FAnimMontageInstance* MontageInstance = AnimInstance->GetActiveInstanceForMontage(MontageToPlay);
			if (MontageInstance)
			{
				MontageInstance->OnMontageBlendingOutStarted.Unbind();
				MontageInstance->OnMontageEnded.Unbind();
			}

			// 停止当前的蒙太奇
			AbilitySystemComponent->CurrentMontageStop();
			return true;
		}
	}

	return false;
}

FString URPGAbilityTask_PlayMontageAndWaitForEvent::GetDebugString() const
{
	UAnimMontage* PlayingMontage = nullptr;
	if (Ability)
	{
		// 获取当前的Actor信息和动画实例
		const FGameplayAbilityActorInfo* ActorInfo = Ability->GetCurrentActorInfo();
		UAnimInstance* AnimInstance = ActorInfo->GetAnimInstance();

		// 如果当前的 MontageToPlay 已经激活，使用MontageToPlay，否则使用 AnimInstance->GetCurrentActiveMontage()
		if (AnimInstance != nullptr)
		{
			PlayingMontage = AnimInstance->Montage_IsActive(MontageToPlay) ? MontageToPlay : AnimInstance->GetCurrentActiveMontage();
		}
	}
	// 根据 PlayingMontage 和 MontageToPlay 来拼接任务描述字符串
	return FString::Printf(TEXT("PlayMontageAndWaitForEvent. MontageToPlay: %s  (Currently Playing): %s"), *GetNameSafe(MontageToPlay), *GetNameSafe(PlayingMontage));
}

```

