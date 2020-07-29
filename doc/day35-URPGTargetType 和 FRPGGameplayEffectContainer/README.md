# URPGTargetType 和 FRPGGameplayEffectContainer

## 1 URPGTargetType

```c++
/**
 * Class that is used to determine targeting for abilities
 * It is meant to be blueprinted to run target logic
 * This does not subclass GameplayAbilityTargetActor because this class is never 
 * instanced into the world
 * This can be used as a basis for a game-specific targeting blueprint
 * If your targeting is more complicated you may need to instance into the world once or 
 * as a pooled actor
 *
 * 这个类被用于技能系统锁定目标，对蓝图来说，这意味着去执行目标物的逻辑
 * 这个类不是 GameplayAbilityTargetActor 子类，因为这个类从来不再世界场景中实例化
 * 这个类可以被用来作为game-specific的基础目标蓝图，如果你的目标足够复杂，
 * 你需要在世界场景中实例化它或者让他作为一个Actor池
 * 
 */
UCLASS(Blueprintable, meta = (ShowWorldContextPin))
class ACTIONRPG_API URPGTargetType : public UObject
{
	GENERATED_BODY()

public:
	// Constructor and overrides
	URPGTargetType() {}

	/** 
	* Called to determine targets to apply gameplay effects to 
	* 调用此方法将获取哪些目标将被应用到游戏效果中
	*/
	UFUNCTION(BlueprintNativeEvent)
	void GetTargets(
		ARPGCharacterBase* TargetingCharacter, 
		AActor* TargetingActor, 
		FGameplayEventData EventData, 
		TArray<FHitResult>& OutHitResults, 
		TArray<AActor*>& OutActors
	) const;
};

```



## 2 FRPGGameplayEffectContainer

```c++
/**
 * Struct defining a list of gameplay effects, a tag, and targeting info
 * These containers are defined statically in blueprints or assets and 
 * then turn into Specs at runtime
 *
 * 这是一个定义了一系列游戏效果、标签、和目标信息的结构体
 * 这个容器在蓝图或者资源中是被定义静态的，并且在运行时转换为 Specs
 */
USTRUCT(BlueprintType)
struct FRPGGameplayEffectContainer
{
	GENERATED_BODY()

public:
	FRPGGameplayEffectContainer() {}

	/** 
	* Sets the way that targeting happens 
	* 目标物类型
	*/
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = GameplayEffectContainer)
	TSubclassOf<URPGTargetType> TargetType;

	/** 
	* List of gameplay effects to apply to the targets 
	* 目标物身上一系列的游戏效果。
	*/
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = GameplayEffectContainer)
	TArray<TSubclassOf<UGameplayEffect>> TargetGameplayEffectClasses;
};

```

## 3 FRPGGameplayEffectContainerSpec

```c++
/** 
* A "processed" version of RPGGameplayEffectContainer that can be passed 
* around and eventually applied 
*
* 一个可以被处理的版本关于 RPGGameplayEffectContainer，这样就可以让 RPGGameplayEffectContainer
* 被传递和事件化的使用。
*/
USTRUCT(BlueprintType)
struct FRPGGameplayEffectContainerSpec
{
	GENERATED_BODY()

public:
	FRPGGameplayEffectContainerSpec() {}

	/**
	* Computed target data 
	* 用来计算目标物的数据
	*/
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = GameplayEffectContainer)
	FGameplayAbilityTargetDataHandle TargetData;

	/** 
	* List of gameplay effects to apply to the targets 
	* 目标物身上的游戏效果
	*/
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = GameplayEffectContainer)
	TArray<FGameplayEffectSpecHandle> TargetGameplayEffectSpecs;

	/** 
	* Returns true if this has any valid effect specs 
	* 返回 true 代表有合法的效果
	*/
	bool HasValidEffects() const;

	/** 
	* Returns true if this has any valid targets 
	* 返回 true 代表有合法的游戏目标
	*/
	bool HasValidTargets() const;

	/** Adds new targets to target data 
	* 在目标数据中添加新目标
	*/
	void AddTargets(const TArray<FHitResult>& HitResults, const TArray<AActor*>& TargetActors);
};
```

明天我们来研究上面这些方法的实现