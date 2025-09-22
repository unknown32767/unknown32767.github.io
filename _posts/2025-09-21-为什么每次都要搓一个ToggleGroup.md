---
layout: post
title: "为什么每次都要搓一个ToggleGroup"
date: 2025-09-21 23:00:00 +0800
categories: blog
---

时至今日UMG仍然没有对应Unity ToggleGroup的玩意，所以每次给项目搓UI就会发现又得自己写一个。

虽然ToggleGroup也没什么复杂的，不过硬要说的话仍然有一些设计上的讨论空间。

如果不希望动到Checkbox的代码，那么ToggleGroup需要负责给每一个受管理的的Checkbox绑定`OnCheckStateChanged`回调，在那里确定所有Checkbox的新状态。

由于`OnCheckStateChanged`本身还不提供this，ToggleGroup在收到回调后需要自己手动检查一遍所有Checkbox状态来找到哪一个Checkbox被点击了。

具体的实现可以参考[1](https://gwb.tencent.com/community/detail/120606)。重写`OnWidgetRebuilt`可以保证在初始化及运行期间添加子Checkbox时自动的更新Checkbox列表。

但是如果我们接受修改Checkbox的话就会简单一些：Checkbox记录所属的ToggleGroup，在用户尝试更改状态的时候Checkbox自己就能根据ToggleGroup记录的状态和设置去决定是否真的更改自己的状态。

```diff
@@ -1,19 +1,5 @@
-void UToggle::OnCheckStateChangedCallback(ECheckBoxState NewState)
+void UCheckBox::SlateOnCheckStateChangedCallback(ECheckBoxState NewState)
 {
-	if (IsValid(ToggleGroup))
-	{
-		if (NewState == ECheckBoxState::Checked)
-		{
-			ToggleGroup->OnToggleChecked(this);
-		}
-		else if (NewState == ECheckBoxState::Unchecked && ToggleGroup->GetLastCheckedToggle() == this && !ToggleGroup->AllowSwitchOff)
-		{
-			OnToggleTouched.Broadcast();
-			SetIsChecked(true);
-			return;
-		}
-	}
-
 PRAGMA_DISABLE_DEPRECATION_WARNINGS
 	if (CheckedState != NewState)
 	{
```

而ToggleGroup本身要做的事就只有一个：在另一个Checkbox被勾选的时候，解除上一个勾选中的Checkbox。

```cpp
void UToggleGroup::OnToggleChecked(TObjectPtr<UToggle> Toggle)
{
	if (Toggle == LastCheckedToggle)
	{
		return;
	}

	if (IsValid(LastCheckedToggle))
	{
		LastCheckedToggle->SetIsChecked(false);
	}
	LastCheckedToggle = Toggle;
}
```