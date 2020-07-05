# 	GameInstace中读取存档并且初始化商城信息

## 1 读取存档并初始背包信息

![image-20200705151823333](./images/image-20200705151823333.png)

### 1.1 SetSavingEnabled

![image-20200705151923528](./images/image-20200705151923528.png)

### 1.2 GetSaveSlotInfo

![image-20200705152007254](./images/image-20200705152007254.png)

其中`SaveSlot`和`UseIndex`的信息在`URPGGameInstanceBase`的构造函数中初始化：

![image-20200705152101508](./images/image-20200705152101508.png)

### 1.3 `HandleSaveGameLoaded` 和`WriteSaveGame`

这两个方法在之前讲过，都在`URPGGameInstanceBase`中，这里给给出部分截图：

![image-20200705152331817](./images/image-20200705152331817.png)

![image-20200705152350860](./images/image-20200705152350860.png)

## 2 加载商品信息

![image-20200705152858601](./images/image-20200705152858601.png)

### 2.1 ItemSlotsPerType

商场中的物品种类，定义在`URPGGameInstanceBase`中，在`BP_GameInstance`中做初始化：

![image-20200705152544617](./images/image-20200705152544617.png)

![image-20200705152604104](./images/image-20200705152604104.png)

### 2.2 资源的真正位置

![image-20200705153140191](./images/image-20200705153140191.png)

尝试断点追着武器信息，发现加载出六种武器，其实这些资源(包括武器)真正位置存储在这里:

![image-20200705153330868](./images/image-20200705153330868.png)

### 2.3 StoreItems

商城中的物品信息，一个`RPGItem`的数组，在蓝图中定义。

![image-20200705153429777](./images/image-20200705153429777.png)