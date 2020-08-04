# URPGAbilityTask_PlayMontageAndWaitForEvent 头文件注释

```c++
// Copyright 1998-2019 Epic Games, Inc. All Rights Reserved.

#pragma once

#include "ActionRPG.h"
#include "Abilities/Tasks/AbilityTask.h"
#include "RPGAbilityTask_PlayMontageAndWaitForEvent.generated.h"

class URPGAbilitySystemComponent;

/** 
* Delegate type used, EventTag and Payload may be empty if it came from the montage callbacks 
* 一个播放蒙太奇并且等待的会话类型，EventTag 和 EventData 可能会为空如果该会话来自蒙太奇的回调
*/
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FRPGPlayMontageAndWaitForEventDelegate, FGameplayTag, EventTag, FGameplayEventData, EventData);

/**
 * This task combines PlayMontageAndWait and WaitForEvent into one task, so you can wait 
 * for multiple types of activations such as from a melee combo
 * Much of this code is copied from one of those two ability tasks
 * This is a good task to look at as an example when creating game-specific tasks
 * It is expected that each game will have a set of game-specific tasks to do what they want
 * 
 * 这个任务包含播放蒙太奇等待和事件等待。因此你可以等待多种类型的能力激活，例如近战连招系统
 * 大部分的代码都是从能力系统中复制而来的。当创建游戏特定数据的时候，这是一个非常好的任务去寻找一个实例。
 * 在每个游戏中，最好都使用这样的任务机制来做一些游戏中想做的事情。
 */
UCLASS()
class ACTIONRPG_API URPGAbilityTask_PlayMontageAndWaitForEvent : public UAbilityTask
{
	GENERATED_BODY()

public:
	// 构造函数
	URPGAbilityTask_PlayMontageAndWaitForEvent(const FObjectInitializer& ObjectInitializer);
	/** 
	* Called to trigger the actual task once the delegates have been set up
	* Note that the default implementation does nothing and you don't have to call it 
	* 一旦会话被创建，立刻调用此方法去触发实际要执行的任务，
	* 记住：默认的实现将不会去做任何事情，并且你也没必要去调用他
	*/
	virtual void Activate() override;
	/** 
	* Called when the task is asked to cancel from an outside node. 
	* What this means depends on the individual task. 
	* By default, this does nothing other than ending the task. 
	* 如果该任务需要在外部取消，调用此方法。这样就意味着依赖单独的任务。
	* 默认情况下，相比与结束任务，此方法将不会做任何事
	*/
	virtual void ExternalCancel() override;

	// 获取调试模式下的任务描述
	virtual FString GetDebugString() const override;
	
	/** 
	* End and CleanUp the task - may be called by the task itself or
	* by the task owner if the owner is ending.
	* 结束或者清理任务。此方法将被由任务自身调用或者任务的所有者在它自身结束时调用。
	*	IMPORTANT! Do NOT call directly! Call EndTask() or TaskOwnerEnded()
	*   重点：不要直接调用此方法，应该调用 EndTask 或者 TaskOwnerEnded
	* 
	*	IMPORTANT! When overriding this function make sure to call 
	*   Super::OnDestroy(bOwnerFinished) as the last thing,
	*   since the function internally marks the task as "Pending Kill", 
	*   and this may interfere with internal BP mechanics
	*   重点：当重载此方法时，务必要在最后一行调用Super::OnDestroy(bOwnerFinished)
	*   因为此方法将会在内部为该任务打上一个"等待杀死"的标记，并且可能会在内部干扰蓝图虚拟机的运行
	*/
	virtual void OnDestroy(bool AbilityEnded) override;

	/** 
	* The montage completely finished playing 
	* 蒙太奇播放和等待的会话
	*/
	UPROPERTY(BlueprintAssignable)
	FRPGPlayMontageAndWaitForEventDelegate OnCompleted;

	/** 
	* The montage started blending out 
	* 蒙太奇开始混合
	*/
	UPROPERTY(BlueprintAssignable)
	FRPGPlayMontageAndWaitForEventDelegate OnBlendOut;

	/** 
	* The montage was interrupted
	* 蒙太奇被中断
	*/
	UPROPERTY(BlueprintAssignable)
	FRPGPlayMontageAndWaitForEventDelegate OnInterrupted;

	/** 
	* The ability task was explicitly cancelled by another ability 
	* 蒙太奇事件被另一种能力所取消
	*/
	UPROPERTY(BlueprintAssignable)
	FRPGPlayMontageAndWaitForEventDelegate OnCancelled;

	/** 
	* One of the triggering gameplay events happened 
	* 蒙太奇中一个触发事件发生
	*/
	UPROPERTY(BlueprintAssignable)
	FRPGPlayMontageAndWaitForEventDelegate EventReceived;

	/**
	 * Play a montage and wait for it end. If a gameplay event happens that 
	 * matches EventTags (or EventTags is empty), the EventReceived delegate will 
	 * fire with a tag and event data.
	 * 播放蒙太奇动画并且等待它结束，如果一个匹配EventTags的  GamePlay 事件触发（或者EventTags）是空的
	 * EventReceived 会话将会被触发，并且携带着标签和事件数据。
	 * 
	 * If StopWhenAbilityEnds is true, this montage will be aborted if the ability 
	 * ends normally. It is always stopped when the ability is explicitly cancelled.
	 * 如果 StopWhenAbilityEnds 为true，当能力正常结束时，该蒙太奇也会终止。
	 * 不过当能力被明确取消时，蒙太奇也会停止。
	 * 
	 * On normal execution, OnBlendOut is called when the montage is blending out, 
	 * and OnCompleted when it is completely done playing
	 * 当正常执行时，OnBlendOut 被执行当蒙太奇正在被混合的时候，而 OnCompleted 是在完成播放时执行。
	 * 
	 * OnInterrupted is called if another montage overwrites this, 
	 * and OnCancelled is called if the ability or task is cancelled
	 * 当另一个蒙太奇重写此蒙太奇时，OnInterrupted将会被调用，而能力或者任务取消时，调用OnCancelled
	 *
	 * @param TaskInstanceName Set to override the name of this task, for later querying
	 *	任务实例的名称，方便后续的查询
	 *
	 * @param MontageToPlay The montage to play on the character
	 *  角色将要播放的蒙太奇动画
	 * 
	 * @param EventTags Any gameplay events matching this tag will 
	 * activate the EventReceived callback. If empty, all events will trigger callback
	 * 匹配到该标签的所有 gameplay events 都将会激活EventReceived事件的回调。如果为空，所有事件都会触发回调
	 *
	 * @param Rate Change to play the montage faster or slower
	 * 播放比率，让蒙太奇动画播放的快点或者满意的
	 * 
	 * @param bStopWhenAbilityEnds If true, this montage will be aborted 
	 * if the ability ends normally. 
	 * It is always stopped when the ability is explicitly cancelled
	 * 当能力系统结束时，是否终止蒙太奇播放，true 表示会停止。不过当能力被确定取消时，该蒙太奇也会停止
	 * 
	 * @param AnimRootMotionTranslationScale Change to modify size of root motion or 
	 * set to 0 to block it entirely
	 * 根骨骼动画缩放，当设置为0时表示完全阻塞根骨骼动画
	 */
	UFUNCTION(BlueprintCallable, Category="Ability|Tasks", meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", BlueprintInternalUseOnly = "TRUE"))
	static URPGAbilityTask_PlayMontageAndWaitForEvent* PlayMontageAndWaitForEvent(
		UGameplayAbility* OwningAbility,
		FName TaskInstanceName,
		UAnimMontage* MontageToPlay,
		FGameplayTagContainer EventTags,
		float Rate = 1.f,
		FName StartSection = NAME_None,
		bool bStopWhenAbilityEnds = true,
		float AnimRootMotionTranslationScale = 1.f);

private:
	/** 
	* Montage that is playing 
	* 当前正在播放的蒙太奇动画
	*/
	UPROPERTY()
	UAnimMontage* MontageToPlay;

	/** 
	* List of tags to match against gameplay events
	* 匹配当前事件的游戏列表
	*/
	UPROPERTY()
	FGameplayTagContainer EventTags;

	/** 
	* Playback rate 
	* 播放比率
	*/
	UPROPERTY()
	float Rate;

	/** 
	* Section to start montage from 
	* 开始播放的 Section

	*/
	UPROPERTY()
	FName StartSection;

	/** 
	* Modifies how root motion movement to apply
	* 修改根骨骼动画
	*/
	UPROPERTY()
	float AnimRootMotionTranslationScale;

	/** 
	* Rather montage should be aborted if ability ends 
	* 当能力结束时，蒙太奇动画释放被终止
	*/
	UPROPERTY()
	bool bStopWhenAbilityEnds;

	/**
	* Checks if the ability is playing a montage and stops that montage, 
	* returns true if a montage was stopped, false if not. 
	* 停止当前正在播放的蒙太奇，如果返回 true 表示停止成功，否则表示停止失败
	*/
	bool StopPlayingMontage();

	/** 
	* Returns our ability system component 
	* 获取目标的能力系统组件
	*/
	URPGAbilitySystemComponent* GetTargetASC();

	// 蒙太奇混合
	void OnMontageBlendingOut(UAnimMontage* Montage, bool bInterrupted);
	// 能力取消
	void OnAbilityCancelled();
	// 蒙太奇结束
	void OnMontageEnded(UAnimMontage* Montage, bool bInterrupted);
	// gamePlay 事件
	void OnGameplayEvent(FGameplayTag EventTag, const FGameplayEventData* Payload);

	FOnMontageBlendingOutStarted BlendingOutDelegate;
	FOnMontageEnded MontageEndedDelegate;
	FDelegateHandle CancelledHandle;
	FDelegateHandle EventHandle;
};
```

