# 游戏中的角色系统初步分析

到了这里我们基本上已经了解游戏的加载过程以及游戏中的商品、关卡等信息，接下来我们将会重点研究游戏中的角色信息。下面是游戏中角色关系图，由`RPGCharacterBase`作为基类而派生出的各种角色，我们先从`RPGCharacterBase`入手。

![image-20200712140404483](./images/image-20200712140404483.png)

## 1 RPGCharacterBasse

![image-20200712141204238](./images/image-20200712141204238.png)

整个类继承了`Character`和`IAbilitySystemInterface`,`Character` 顾名思义，UE4 自带的角色类，而`IAbilitySystemInterface` 则是 UE4 的一个关于角色技能的一个插件，这里我们只需稍作了解，后续会有专门的章节来对`GamePlayerAbilities` 做分析。

![image-20200712141425642](./images/image-20200712141425642.png)

官方文档：https://docs.unrealengine.com/zh-CN/Gameplay/GameplayAbilitySystem/index.html

![image-20200712142059151](./images/image-20200712142059151.png)

### 1.1 UAbilitySystemComponent

一个接口用于暴露角色的能力系统

![image-20200712142551983](./images/image-20200712142551983.png)

#### 1.1.1 GetAbilitySystemComponent

再`RPGCharacterBase`重写了此方法

![image-20200712142712463](./images/image-20200712142712463.png)

具体实现：

![image-20200712143023713](./images/image-20200712143023713.png)

再`RPGCharacterBase`中定义了一个属性，来保存技能系统：

![image-20200712143218209](./images/image-20200712143218209.png)

### 1.2 构造函数

![image-20200712143512098](./images/image-20200712143512098.png)

#### 1.2.1 URPGAttributeSet

可以被技能系统修改的技能列表

![image-20200712143610819](./images/image-20200712143610819.png)

#### 1.2.2 CharacterLevel

角色等级，当角色被创建时，最好不要直接修改这个值。

![image-20200712143812520](./images/image-20200712143812520.png)

#### 1.2.3 bAbilitiesInitialized

`Protected`属性，表示是否已经初始化技能系统，如果为`true`,表示我们已经初始化技能系统。

![image-20200712144001922](./images/image-20200712144001922.png)

### 1.3 PossessedBy

当角色被控制时调用，[官方文档](https://docs.unrealengine.com/en-US/API/Runtime/Engine/GameFramework/ACharacter/PossessedBy/index.html)这样解释：Called when this Pawn is possessed. Only called on the server (or in standalone).

![image-20200712144540324](./images/image-20200712144540324.png)

具体的实现：

![image-20200712145225526](./images/image-20200712145225526.png)

#### 1.3.1 InventorySource

角色背包缓存的指针，可以为 null。

![image-20200712145850057](./images/image-20200712145850057.png)

#### 1.3.2 OnItemSlotChanged 和 RefreshSlottedGameplayAbilities

![image-20200712145606894](./images/image-20200712145606894.png)

#### 1.3.3 InitAbilityActorInfo

```
初始化技能—— [ActorInfo] 是一个结构体，有着我们正在扮演角色的信息以及谁控制了我们。
@param OwnerActor 一个逻辑上角色被该组件持有
@Param AvatarActor 一个物理的角色，我们正在事件中扮演且再世界中真实存在的，经成是一个[Pawn]，但是不能是塔，建筑物等。这个参数可以和OwnerActor相同。
```

![image-20200712150435011](./images/image-20200712150435011.png)

具体的实现细节就不追究了，有兴趣的自己看源码吧，这里只给出一张实现部分贴图：

![image-20200712151037122](./images/image-20200712151037122.png)

#### 1.3.4 AddStartupGameplayAbilities

应用初始化的技能和效果

![image-20200712153829065](./images/image-20200712153829065.png)

![image-20200712153514183](./images/image-20200712153514183.png)

#### 1.3.5 GameplayAbilities

当角色被创建的时候授予角色的技能，这个将会由`标签`或者`事件`激活，但是不能绑定到输入事件。

![，image-20200712151428482](./images/image-20200712151428482.png)

#### 1.3.6 PassiveGameplayEffects

当角色被创建时，添加消极的技能效果。举个例子，当角色被打时，会产生流血的消极效果。

![image-20200712151725364](./images/image-20200712151725364.png)

#### 1.3.7 AddSlottedGameplayAbilities

如果需要，添加技能插槽。

![image-20200712152139490](./images/image-20200712152139490.png)

![image-20200712153335731](./images/image-20200712153335731.png)

#### 1.3.8 FillSlottedAbilitySpecs

从背包或者默认系统中填充技能。

![image-20200712152354737](./images/image-20200712152354737.png)

```c++
void ARPGCharacterBase::FillSlottedAbilitySpecs(TMap<FRPGItemSlot, FGameplayAbilitySpec>& SlottedAbilitySpecs)
{
	// 添加默认的技能
	for (const TPair<FRPGItemSlot, TSubclassOf<URPGGameplayAbility>>& DefaultPair : DefaultSlottedAbilities)
	{
		if (DefaultPair.Value.Get())
		{
			SlottedAbilitySpecs.Add(DefaultPair.Key, FGameplayAbilitySpec(DefaultPair.Value, GetCharacterLevel(), INDEX_NONE, this));
		}
	}

	// 添加背包中潜在的技能
	if (InventorySource)
	{
		const TMap<FRPGItemSlot, URPGItem*>& SlottedItemMap = InventorySource->GetSlottedItemMap();

		for (const TPair<FRPGItemSlot, URPGItem*>& ItemPair : SlottedItemMap)
		{
			URPGItem* SlottedItem = ItemPair.Value;

			// 默认使用角色等级作为技能等级
			int32 AbilityLevel = GetCharacterLevel();

			if (SlottedItem && SlottedItem->ItemType.GetName() == FName(TEXT("Weapon")))
			{
				// 从插槽中复写技能等级
				AbilityLevel = SlottedItem->AbilityLevel;
			}

			if (SlottedItem && SlottedItem->GrantedAbility)
			{
				// 添加默认的技能
				SlottedAbilitySpecs.Add(ItemPair.Key, FGameplayAbilitySpec(SlottedItem->GrantedAbility, AbilityLevel, INDEX_NONE, SlottedItem));
			}
		}
	}
}
```



#### 1.3.9 DefaultSlottedAbilities

默认的能力插槽，一个 Map 集合关于技能系统的class, 它应该被绑定再能力系统添加到背包中。

![image-20200712152638980](./images/image-20200712152638980.png)