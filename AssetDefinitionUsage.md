# 【UE5】使用 AssetDefinition 添加自定义资产到内容浏览器

在开发过程中，为了使用上的方便或者进一步的编辑器开发，首先会将经常使用的自定义资产添加到内容浏览器的右键菜单中，新的`AssetDefinition`相比过去的`AssetTypeActions`则更加便利强大。

## 起因

偶然发现了这样一条弃用声明：

``` cpp
// UE_DEPRECATED(5.2, "The AssetDefinition system is replacing AssetTypeActions and UAssetDefinition_DataAsset replaced this.  Please see the Conversion Guide in AssetDefinition.h")
```

虽然还没有取消注释，引擎内部的大部分`AssetTypeActions`已经被换成`AssetDefinition`，看样子`AssetDefinition`要替代过去的`AssetTypeActions`是大势所趋。

本文简要记录一下`AssetDefinition`的使用和 snippet，不过它能够做到的远不止如此，虽然还没有正式文档，好奇的小伙伴可以查看相关源码。

## `AssetDefinition`的使用

假设我们要添加一个叫 AbilitySet 的`UDataAsset`到右键菜单中，这个 AbilitySet 在一个叫`MyCustom`的模块/插件中，需要如下三个步骤：

### 示例插件结构（仅供参考）

基本的插件可以有如下文件：

---

- `MyCustom`
    - MyCustom.uplugin
    - `Source`
        - `MyCustom`
            - MyCustom.Build.cs
            - `Public`
                - AbilitySet.h
            - `Private`
                - AbilitySet.cpp
        - `MyCustomEditor`
            - MyCustomEditor.Build.cs
            - `Private`
                - `AssetTools`
                    - AssetDefinition_AbilitySet.h
                    - AssetDefinition_AbilitySet.cpp
                - `Factories`
                    - AbilitySetFactory.h
                    - AbilitySetFactory.cpp
                - MyCustomEditorModule.cpp


---

#### `MyCustom.uplugin`:

``` csharp
"Modules": [
    {
        "Name": "MyCustom",
        "Type": "Runtime",
        "LoadingPhase": "Default"
    },
    {
        "Name": "MyCustomEditor",
        "Type": "Editor",
        "LoadingPhase": "PostEngineInit"
    }
```

---

#### `MyCustomEditor.Build.cs`:

在编辑器模块中，我们需要添加`AssetDefinition`依赖：

``` csharp
PrivateDependencyModuleNames.AddRange(
    new string[] {
        "Core",
        "CoreUObject",
        "UnrealEd",
        "MyCustom",
        "AssetDefinition",
    }
);
```

---

### 创建该资产的 Factory 类

为了在内容浏览器新建自定义资产，我们先实现对应资产的`UFactory`类：

#### `Private/Factories/AbilitySetFactory.h`:

``` cpp
#pragma once

#include "Factories/Factory.h"
#include "AbilitySetFactory.generated.h"

/**
 * The factory for the AbilitySet asset type.
 * This basically integrates the new asset type into the editor, so you can right click and create a new AbilitySet asset.
 */
UCLASS(hidecategories=Object)
class UAbilitySetFactory : public UFactory
{
    GENERATED_BODY()

public:
    UAbilitySetFactory();

    //~UFactory interface
    virtual UObject* FactoryCreateNew(UClass* InClass, UObject* InParent, FName InName, EObjectFlags Flags, UObject* Context, FFeedbackContext* Warn) override;
    virtual bool ShouldShowInNewMenu() const override;
    virtual FText GetDisplayName() const override;
    virtual uint32 GetMenuCategories() const override;
    virtual FText GetToolTip() const override;
    virtual FString GetDefaultNewAssetName() const override;
    //~End of UFactory interface
};
```

---

#### `Private/Factories/AbilitySetFactory.cpp`:

在相关函数中返回我们的资产信息：

``` cpp
#include "Factories/AbilitySetFactory.h"
#include "AbilitySystem/AbilitySet.h"
#include "AssetTypeCategories.h"

#define LOCTEXT_NAMESPACE "AbilitySetFactory"

UAbilitySetFactory::UAbilitySetFactory()
{
    bCreateNew = true;
    bEditAfterNew = true;
    SupportedClass = UAbilitySet::StaticClass();
}

UObject* UAbilitySetFactory::FactoryCreateNew(UClass* InClass, UObject* InParent, FName InName, EObjectFlags Flags, UObject* Context, FFeedbackContext* Warn)
{
    check(InClass->IsChildOf(UAbilitySet::StaticClass()));
    return NewObject<UAbilitySet>(InParent, InClass, InName, Flags | RF_Transactional, Context);;
}

bool UAbilitySetFactory::ShouldShowInNewMenu() const
{
    return true;
}

FText UAbilitySetFactory::GetDisplayName() const
{
    return LOCTEXT("AbilitySet_DisplayName", "Ability Set");
}

uint32 UAbilitySetFactory::GetMenuCategories() const
{
    return EAssetTypeCategories::Gameplay;
}

FText UAbilitySetFactory::GetToolTip() const
{
    return LOCTEXT("AbilitySet_Tooltip", "Data used by the ability set to grant gameplay abilities.");
}

FString UAbilitySetFactory::GetDefaultNewAssetName() const
{
    return FString(TEXT("AbilitySet_NewSet"));
}

#undef LOCTEXT_NAMESPACE
```

---

### 创建该资产的 AssetDefinition 类

#### `Private/AssetTools/AssetDefinition_AbilitySet.h`:

``` cpp
#pragma once

#include "AssetDefinitionDefault.h"
#include "AssetDefinition_AbilitySet.generated.h"

UCLASS()
class UAssetDefinition_AbilitySet : public UAssetDefinitionDefault
{
    GENERATED_BODY()

public:
    //~UAssetDefinition interface
    FText GetAssetDisplayName() const override final;
    TSoftClassPtr<UObject> GetAssetClass() const override final;
    FLinearColor GetAssetColor() const override final;
    FText GetAssetDescription(const FAssetData& AssetData) const override final;
    TConstArrayView<FAssetCategoryPath> GetAssetCategories() const override final;
    //~End of UAssetDefinition interface
};
```

---

#### `Private/AssetTools/AssetDefinition_AbilitySet.cpp`:

设定资产的基本属性：

``` cpp
#include "AssetTools/AssetDefinition_AbilitySet.h"
#include "AbilitySystem/AbilitySet.h"

#define LOCTEXT_NAMESPACE "AssetDefinition_AbilitySet"

FText UAssetDefinition_AbilitySet::GetAssetDisplayName() const
{
    return LOCTEXT("AssetTypeActions_AbilitySet", "Ability Set");
}

TSoftClassPtr<UObject> UAssetDefinition_AbilitySet::GetAssetClass() const
{
    return UAbilitySet::StaticClass();
}

FLinearColor UAssetDefinition_AbilitySet::GetAssetColor() const
{
    return FLinearColor(FColor(201, 29, 85));
}

FText UAssetDefinition_AbilitySet::GetAssetDescription(const FAssetData& AssetData) const
{
    return LOCTEXT("AssetTypeActions_AbilitySetDesc", "Data used by the ability set to grant gameplay abilities.");
}

TConstArrayView<FAssetCategoryPath> UAssetDefinition_AbilitySet::GetAssetCategories() const
{
    static const auto Categories = { EAssetCategoryPaths::Gameplay };
    return Categories;
}
```

然后保存，编译，启动编辑器就会发现 AbilitySet 出现在内容浏览器右键 Gameplay 分类下。

---

### 实现自己的分类

上文是添加在引擎内置的`EAssetTypeCategories::Gameplay`类别下，如果我们想添加引擎没有的分类，可以和从前一样使用 AssetTool 的`RegisterAdvancedAssetCategory`进行注册，但也有更加简单直接的方法，直接返回新建分类即可：

``` cpp
#define LOCTEXT_NAMESPACE "AssetDefinition_AbilitySet"
// ...
TConstArrayView<FAssetCategoryPath> UAssetDefinition_AbilitySet::GetAssetCategories() const
{
    static const auto Categories = { FAssetCategoryPath(LOCTEXT("MyCustomAssetsCategoryName", "MyCustom")) };
    return Categories;
}
// ...
#undef LOCTEXT_NAMESPACE
```

如果你还想要一个子菜单，可以有以下两种风格写法：

``` cpp
{
    //static const auto Categories = { FAssetCategoryPath(EAssetCategoryPaths::Gameplay, LOCTEXT("MyCustom_SubCategory", "MyCustomSub")) };
    static const auto Categories = { EAssetCategoryPaths::Gameplay / LOCTEXT("MyCustomAssetsSubCategoryName", "MyCustomSub") };
    return Categories;
}
```

## 额外的 EngineAssetDefinitions 模块/插件

和过去的`AssetTypeActions`类似，Epic 也写好了一些常见资产的`AssetDefinition`可供继承，这些类放在`EngineAssetDefinitions`模块中。

上文的 AbilitySet 资产除了可以继承自`UAssetDefinitionDefault`，也可直接继承`UAssetDefinition_DataAsset`。

## 转换现有的 AssetTypeActions 到 AssetDefinition

首先是参考写在`AssetDefinition.h`里的转换指南，对新老 API 的对应关系感到疑惑的话，还可参考`UAssetDefinition_AssetTypeActionsProxy`这个类中的做法。

如有疏漏不妥之处，还望不吝赐教。

---

This work is licensed under CC BY-NC 4.0