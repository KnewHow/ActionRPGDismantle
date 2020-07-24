# BP_Character 初探（一）

![image-20200723195302728](./images/image-20200723195302728.png)

前置摄像机效果：

![image-20200723195402204](./images/image-20200723195402204.png)

## 1 Event Begin Player

![image-20200723195447615](./images/image-20200723195447615.png)



## 2 玩家输入事

### 2.1 InputAction Run

![image-20200723195656066](./images/image-20200723195656066.png)

### 2.2 InputAction NormalAttack

![image-20200723200312118](./images/image-20200723200312118.png)

先有个概念，具体的实现我们下一次在研究

### 2.3 InputAction  SpecialAttack

![image-20200723200349903](./images/image-20200723200349903.png)

### 2.4 InputAction   UseItem

![image-20200723200402022](./images/image-20200723200402022.png)

### 2.5 InputAction Roll

![image-20200723200951879](./images/image-20200723200951879.png)

播放翻滚的动画函数：

![image-20200723201046454](./images/image-20200723201046454.png)



角色移动的蒙太奇动画：

![image-20200724200804212](./images/image-20200724200804212.png)

而实际上他使用了带位移的根骨骼动画：

![image-20200724200855552](./images/image-20200724200855552.png)

而在动画蓝图中做了这样的配置：

![image-20200724200940969](./images/image-20200724200940969.png)

### 2.6 InputActioen ChangeWeapon

![image-20200723202355829](./images/image-20200723202355829.png)

![image-20200723202339930](./images/image-20200723202339930.png)

![image-20200723202646818](./images/image-20200723202646818.png)