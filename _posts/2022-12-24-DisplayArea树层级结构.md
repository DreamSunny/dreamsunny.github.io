---
layout:     post
title:      "DisplayArea树层级结构"
subtitle:   ""
date:       2022-12-24 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
---

![img](/img/DisplayArea/class_hierarchy.png)

- RootWindowContainer：最顶层的管理者，直接管理 DisplayContent 。

- DisplayContent：对应一个真实或者虚拟的显示设备。

- TaskDisplayArea：是系统中所有应用任务的父节点，用于管理 Task 。

- Task：代表一个任务，可以包含多个 Activity 。

- ActivityRecord：对应一个 Activity 节点。

- WindowState：对应一个窗口。

  

# 1.Window相关概念

**Window Type主要分为三大类：**

- Application windows（应用窗口）: 1~99

- Sub-windows（子窗口）: 1000~1999

- System windows（系统窗口）: 2000~2999

**Window Layer分为36层：**

```Java
default int getMaxWindowLayer() {
    return 36;
}
```

**Z-Order计算：**

```Java
mBaseLayer = WindowLayer * 10000 + 1000;
mSubLayer = SubWindowLayer;
```

**Window Type与Window Layer对应关系如下：**

| Window Type                                | VALUE    | Window Layer            | Leaf Type                       |
| ------------------------------------------ | -------- | ----------------------- | ------------------------------- |
| *TYPE_BASE_APPLICATION*                    | 1        | *APPLICATION_LAYER = 2* | *LEAF_TYPE_TASK_CONTAINERS = 1* |
| *TYPE_APPLICATION*                         | 2        |                         |                                 |
| *TYPE_APPLICATION_STARTING*                | 3        |                         |                                 |
| *TYPE_DRAWN_APPLICATION*                   | 4        |                         |                                 |
| *LAST_APPLICATION_WINDOW*                  | 99       |                         |                                 |
| ***TYPE_APPLICATION_PANEL***               | **1000** | **1**                   | -                               |
| ***TYPE_APPLICATION_MEDIA***               | **1001** | **-2**                  | -                               |
| ***TYPE_APPLICATION_SUB_PANEL***           | **1002** | **2**                   | -                               |
| ***TYPE_APPLICATION_ATTACHED_DIALOG***     | **1003** | **1**                   | -                               |
| ***TYPE_APPLICATION_MEDIA_OVERLAY***       | **1004** | **-1**                  | -                               |
| ***TYPE_APPLICATION_ABOVE_SUB_PANEL***     | **1005** | **3**                   | -                               |
| *TYPE_STATUS_BAR*                          | 2000     | 15                      | 0                               |
| *TYPE_SEARCH_BAR*                          | 2001     | 4                       | 0                               |
| *TYPE_PHONE*                               | 2002     | 3                       | 0                               |
| *TYPE_SYSTEM_ALERT*                        | 2003     | 12\|9                   | 0                               |
| *TYPE_KEYGUARD*                            | 2004     | 3                       | 0                               |
| *TYPE_TOAST*                               | 2005     | 7                       | 0                               |
| *TYPE_SYSTEM_OVERLAY*                      | 2006     | 23\|20                  | 0                               |
| *TYPE_PRIORITY_PHONE*                      | 2007     | 8                       | 0                               |
| *TYPE_SYSTEM_DIALOG*                       | 2008     | 6                       | 0                               |
| *TYPE_KEYGUARD_DIALOG*                     | 2009     | 19                      | 0                               |
| *TYPE_SYSTEM_ERROR*                        | 2010     | 27\|9                   | 0                               |
| *TYPE_INPUT_METHOD*                        | 2011     | 13                      | *LEAF_TYPE_IME_CONTAINERS = 2*  |
| *TYPE_INPUT_METHOD_DIALOG*                 | 2012     | 14                      |                                 |
| *TYPE_WALLPAPER*                           | 2013     | 1                       | 0                               |
| *TYPE_STATUS_BAR_PANEL*                    | 2014     | 3                       | 0                               |
| *TYPE_SECURE_SYSTEM_OVERLAY*               | 2015     | 33                      | 0                               |
| *TYPE_DRAG*                                | 2016     | 30                      | 0                               |
| *TYPE_STATUS_BAR_SUB_PANEL*                | 2017     | 18                      | 0                               |
| *TYPE_POINTER*                             | 2018     | 35                      | 0                               |
| *TYPE_NAVIGATION_BAR*                      | 2019     | 24                      | 0                               |
| *TYPE_VOLUME_OVERLAY*                      | 2020     | 22                      | 0                               |
| *TYPE_BOOT_PROGRESS*                       | 2021     | 34                      | 0                               |
| *TYPE_INPUT_CONSUMER*                      | 2022     | 5                       | 0                               |
| *TYPE_NAVIGATION_BAR_PANEL*                | 2024     | 25                      | 0                               |
| *TYPE_DISPLAY_OVERLAY*                     | 2026     | 29                      | 0                               |
| *TYPE_MAGNIFICATION_OVERLAY*               | 2027     | 28                      | 0                               |
| *TYPE_PRIVATE_PRESENTATION*                | 2030     | 3                       | 0                               |
| *TYPE_VOICE_INTERACTION*                   | 2031     | 21                      | 0                               |
| *TYPE_ACCESSIBILITY_OVERLAY*               | 2032     | 31                      | 0                               |
| *TYPE_VOICE_INTERACTION_STARTING*          | 2033     | 20                      | 0                               |
| *TYPE_DOCK_DIVIDER*                        | 2034     | 3                       | 0                               |
| *TYPE_QS_DIALOG*                           | 2035     | 3                       | 0                               |
| *TYPE_SCREENSHOT*                          | 2036     | 26                      | 0                               |
| *TYPE_PRESENTATION*                        | 2037     | 3                       | 0                               |
| *TYPE_APPLICATION_OVERLAY*                 | 2038     | 11                      | 0                               |
| *TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY* | 2039     | 32                      | 0                               |
| *TYPE_NOTIFICATION_SHADE*                  | 2040     | 17                      | 0                               |
| *TYPE_STATUS_BAR_ADDITIONAL*               | 2041     | 16                      | 0                               |
| *TYPE_CARWITH_NAVIGATION_BAR*              | 2998     | 24                      | 0                               |

**常见Window及Window Type：**

| Window             | Window Type                 |
| ------------------ | --------------------------- |
| Activity           | *TYPE_BASE_APPLICATION*     |
| TaskSnapshotWindow | *TYPE_APPLICATION_STARTING* |
| Dialog             | *TYPE_APPLICATION*          |
| PopupWindow        | *TYPE_APPLICATION_PANEL*    |
| Toast              | *TYPE_TOAST*                |



# 2.为什么要构建DisplayArea树？

## 2.1 需求分析

手机系统有N个类型的Window，需要划分Layer维护Window显示Z-order；

有N个Feature（功能），每个Feature能够影响N个Layer层；

窗口类型是可增加的，Feature是可增删的；



## 2.2 方案设计

构建DisplayArea树，创建Feature节点，支持动态增删Feature，叶子节点维护Layer层级。

以 DefaultTaskDisplayArea 节点为例：

其父节点为 5\|0\|12 -> 3\|0\|14 -> 6\|0\|14 -> 4\|0\|31 -> DisplayContent，其含义为该节点的所有子节点均支持 FullscreenMagnification\|5、OneHanded\|3、HideDisplayCutout\|6、WindowedMagnification\|4 功能（Feature）。

同时 DefaultTaskDisplayArea 节点的子节点的层级范围为 2\|2，即所有子节点均为应用窗口，其Layer值是固定的。



# 3.构建DisplayArea树流程

## 3.1 创建RootWindowContainer

1.SystemServer.main(..)
2.SystemServer.run()
3.SystemServer.startOtherServices(..)
4.vm = WindowManagerService.main(..)

```Java
// WindowManagerService.java
private WindowManagerService(..) {
    mRoot = new RootWindowContainer(this);
}
```



## 3.2 创建DisplayContent

5.ActivityManagerService.setWindowManager(wm)
6.ActivityTaskManagerService.setWindowManager(wm)
7.RootWindowContainer.setWindowManager(wm)

```Java
// RootWindowContainer.java
private final ImeContainer mImeWindowsContainer = new ImeContainer(mWmService);
void setWindowManager(WindowManagerService wm) {
    ..
    final Display[] displays = mDisplayManager.getDisplays();
    for (int displayNdx = 0; displayNdx < displays.length; ++displayNdx) {
        final Display display = displays[displayNdx];
        final DisplayContent displayContent = new DisplayContent(display, this);
        addChild(displayContent, POSITION_BOTTOM);
        if (displayContent.mDisplayId == DEFAULT_DISPLAY) {
            mDefaultDisplay = displayContent;
        }
    }
    ..
}
```



## 3.3 创建DefaultTaskDisplayArea

```Java
// DisplayContent.java
DisplayContent(Display display, RootWindowContainer root) {
    ..
    final Transaction pendingTransaction = getPendingTransaction();
    configureSurfaces(pendingTransaction);
    pendingTransaction.apply();
    ..
}

private void configureSurfaces(Transaction transaction) {
    ..
    if (mDisplayAreaPolicy == null) {
        // Setup the policy and build the display area hierarchy.
        // Build the hierarchy only after creating the surface so it is reparented correctly
        mDisplayAreaPolicy = mWmService.getDisplayAreaPolicyProvider().instantiate(
                mWmService, this /* content */, this /* root */,
                mImeWindowsContainer);
    }
    ..
}

//DisplayAreaPolicy.java
static final class DefaultProvider implements DisplayAreaPolicy.Provider {
    @Override
    public DisplayAreaPolicy instantiate(WindowManagerService wmService,
            DisplayContent content, RootDisplayArea root,
            DisplayArea.Tokens imeContainer) {
        final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
        final List<TaskDisplayArea> tdaList = new ArrayList<>();
        tdaList.add(defaultTaskDisplayArea);

        // Define the features that will be supported under the root of the whole logical
        // display. The policy will build the DisplayArea hierarchy based on this.
        final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
        // Set the essential containers (even if the display doesn't support IME).
        rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
        if (content.isTrusted()) {
            // Only trusted display can have system decorations.
            configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
        }

        // Instantiate the policy with the hierarchy defined above. This will create and attach
        // all the necessary DisplayAreas to the root.
        return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
    }
    
    private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
            WindowManagerService wmService, DisplayContent content) {
        rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                FEATURE_WINDOWED_MAGNIFICATION)
                .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                .build());
        if (content.isDefaultDisplay) {
            rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                    FEATURE_HIDE_DISPLAY_CUTOUT)
                    .all()
                    .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                            TYPE_NOTIFICATION_SHADE)
                    .build())
                    .addFeature(new Feature.Builder(wmService.mPolicy, "OneHanded",
                            FEATURE_ONE_HANDED)
                            .all()
                            .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL,
                                    TYPE_SECURE_SYSTEM_OVERLAY)
                            .build());
        }
        rootHierarchy
                .addFeature(new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                        FEATURE_FULLSCREEN_MAGNIFICATION)
                        .all()
                        .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                                TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                                TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                        .build())
                .addFeature(new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                        FEATURE_IME_PLACEHOLDER)
                        .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                        .build());
    }
}
```


在 configureTrustedHierarchyBuilder(..) 方法中配置Feature及其能够影响到的Layer层：

| Feature名               | Feature ID | 对应功能                         |
| ----------------------- | ---------- | -------------------------------- |
| WindowedMagnification   | 4          | 窗口放大镜功能。                 |
| HideDisplayCutout       | 6          | 隐藏刘海功能。                   |
| OneHanded               | 3          | 单手模式。                       |
| FullscreenMagnification | 5          | 屏幕放大镜功能。                 |
| ImePlaceholder          | 7          | 特殊情况下用来放置输入法的节点。 |



在Feature类中使用一个长度为 getMaxWindowLayer() + 1 的boolean数组，用来标识该Feature能够影响到的Layer层：

![img](/img/DisplayArea/array1.png)



## 3.4 DisplayAreaPolicy构建过程

```Java
// DisplayAreaPolicyBuilder.java
private void build(@Nullable List<HierarchyBuilder> displayAreaGroupHierarchyBuilders) {
    final WindowManagerPolicy policy = mRoot.mWmService.mPolicy;
    final int maxWindowLayerCount = policy.getMaxWindowLayer() + 1;
    final DisplayArea.Tokens[] displayAreaForLayer =
            new DisplayArea.Tokens[maxWindowLayerCount];
    final Map<Feature, List<DisplayArea<WindowContainer>>> featureAreas =
            new ArrayMap<>(mFeatures.size());
    for (int i = 0; i < mFeatures.size(); i++) {
        featureAreas.put(mFeatures.get(i), new ArrayList<>());
    }
    ...
}
```


### 3.4.1 构建root节点：

```Java
PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
final PendingArea root = new PendingArea(null, 0, null);
Arrays.fill(areaForLayer, root);
```


###  3.4.2 构建featureArea：

```Java
// Create DisplayAreas to cover all defined features.
final int size = mFeatures.size();
for (int i = 0; i < size; i++) {
    final Feature feature = mFeatures.get(i);
    PendingArea featureArea = null;
    for (int layer = 0; layer < maxWindowLayerCount; layer++) {
        if (feature.mWindowLayers[layer]) {
            if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                areaForLayer[layer].mChildren.add(featureArea);
            }
            areaForLayer[layer] = featureArea;
        } else {
            featureArea = null;
        }
    }
}
```

![img](/img/DisplayArea/array2.png)

黄色部分与上图是相对应的，创建的PendingArea显示格式为：Feature ID\|minLayer

构建出如下一棵树：

![img](/img/DisplayArea/tree1.png)



###  3.4.3 构建 leafArea：

```Java
// Create Tokens as leaf for every layer.
PendingArea leafArea = null;
int leafType = LEAF_TYPE_TOKENS;
for (int layer = 0; layer < maxWindowLayerCount; layer++) {
    int type = typeOfLayer(policy, layer);
    if (leafArea == null || leafArea.mParent != areaForLayer[layer]
            || type != leafType) {
        leafArea = new PendingArea(null /* feature */, layer, areaForLayer[layer]);
        areaForLayer[layer].mChildren.add(leafArea);
        leafType = type;
        if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
            addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
            addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                    displayAreaGroupHierarchyBuilders);
            leafArea.mSkipTokens = true;
        } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
            leafArea.mExisting = mImeContainer;
            leafArea.mSkipTokens = true;
        }
    }
    leafArea.mMaxLayer = layer;
}

private static int typeOfLayer(WindowManagerPolicy policy, int layer) {
    if (layer == APPLICATION_LAYER) {
        return LEAF_TYPE_TASK_CONTAINERS; // 容纳App窗口的TaskDisplayArea
    } else if (layer == policy.getWindowLayerFromTypeLw(TYPE_INPUT_METHOD)
            || layer == policy.getWindowLayerFromTypeLw(TYPE_INPUT_METHOD_DIALOG)) {
        return LEAF_TYPE_IME_CONTAINERS; // 容纳输入法窗口的ImeContainer
    } else {
        return LEAF_TYPE_TOKENS; // 容纳其他非App类型窗口的DisplayArea.Tokens
    }
}
```

![img](/img/DisplayArea/array3.png)

黄色部分表示复用同一个leafArea，更新其mMaxLayer值，这里创建的PendingArea显示格式为：minLayer\|maxLayer



###  3.4.4 更新整棵树所有节点的maxLayer

```Java
int computeMaxLayer() {
    for (int i = 0; i < mChildren.size(); i++) {
        mMaxLayer = Math.max(mMaxLayer, mChildren.get(i).computeMaxLayer());
    }
    return mMaxLayer;
}
```

###  3.4.5 构建整棵树

```Java
void instantiateChildren(DisplayArea<DisplayArea> parent, DisplayArea.Tokens[] areaForLayer,
        int level, Map<Feature, List<DisplayArea<WindowContainer>>> areas) {
    mChildren.sort(Comparator.comparingInt(pendingArea -> pendingArea.mMinLayer));
    for (int i = 0; i < mChildren.size(); i++) {
        final PendingArea child = mChildren.get(i);
        final DisplayArea area = child.createArea(parent, areaForLayer);
        if (area == null) {
            // TaskDisplayArea and ImeContainer can be set at different hierarchy, so it can
            // be null.
            continue;
        }
        parent.addChild(area, WindowContainer.POSITION_TOP);
        if (child.mFeature != null) {
            areas.get(child.mFeature).add(area);
        }
        child.instantiateChildren(area, areaForLayer, level + 1, areas);
    }
}

private DisplayArea createArea(DisplayArea<DisplayArea> parent,
        DisplayArea.Tokens[] areaForLayer) {
    if (mExisting != null) {
        // LEAF_TYPE_TASK_CONTAINERS
        // LEAF_TYPE_IME_CONTAINERS
        if (mExisting.asTokens() != null) {
            // Store the WindowToken container for layers
            fillAreaForLayers(mExisting.asTokens(), areaForLayer);
        }
        return mExisting;
    }
    if (mSkipTokens) {
        // LEAF_TYPE_TASK_CONTAINERS
        // LEAF_TYPE_IME_CONTAINERS
        return null;
    }
    DisplayArea.Type type;
    if (mMinLayer > APPLICATION_LAYER) {
        type = DisplayArea.Type.ABOVE_TASKS; // 位于App窗口之下的非App窗口
    } else if (mMaxLayer < APPLICATION_LAYER) {
        type = DisplayArea.Type.BELOW_TASKS; // 位于App窗口之上的非App窗口
    } else {
        type = DisplayArea.Type.ANY; // App窗口
    }
    if (mFeature == null) {
        final DisplayArea.Tokens leaf = new DisplayArea.Tokens(parent.mWmService, type,
                "Leaf:" + mMinLayer + ":" + mMaxLayer);
        fillAreaForLayers(leaf, areaForLayer);
        return leaf;
    } else {
        return mFeature.mNewDisplayAreaSupplier.create(parent.mWmService, type,
                mFeature.mName + ":" + mMinLayer + ":" + mMaxLayer, mFeature.mId);
    }
}
```


整棵树的根节点是 DisplayContent，这里创建的DisplayArea显示格式为：Feature ID\|minLayer\|maxLayer；叶子节点的显示格式为：minLayer\|maxLayer

![img](/img/DisplayArea/tree2.png)



执行命令：adb shell dumpsys activity containers

```Java
ACTIVITY MANAGER CONTAINERS (dumpsys activity containers)
ROOT 
  #0 Display 0 name="内置屏幕"
   #2 Leaf:36:36 
    #1 WindowToken{7fb09c2 type=2024 android.os.BinderProxy@da9960d} 
     #0 e873d3 RoundCornerBottom 
    #0 WindowToken{2c592ae type=2024 android.os.BinderProxy@5ce3f29} 
     #0 164a84f RoundCornerTop 
   #1 HideDisplayCutout:32:35 
    #2 OneHanded:34:35 
     #0 FullscreenMagnification:34:35 
      #0 Leaf:34:35 
    #1 FullscreenMagnification:33:33 
     #0 Leaf:33:33 
    #0 OneHanded:32:32 
     #0 Leaf:32:32 
   #0 WindowedMagnification:0:31 
    #6 HideDisplayCutout:26:31 
     #0 OneHanded:26:31 
      #2 FullscreenMagnification:29:31 
       #0 Leaf:29:31 
      #1 Leaf:28:28 
       #1 WindowToken{6cffb72 type=2027 android.os.BinderProxy@9b9807d} 
        #0 e90cc40 GestureStubRight 
       #0 WindowToken{288fa0d type=2027 android.os.BinderProxy@3713ca4} 
        #0 1d77f10 GestureStubLeft 
      #0 FullscreenMagnification:26:27 
       #0 Leaf:26:27 
    #5 Leaf:24:25 
     #3 WindowToken{2cfd75e type=2024 android.os.BinderProxy@a18e899} 
      #0 6d1873f GestureStubHome 
     #2 WindowToken{4a05ae3 type=2024 android.os.BinderProxy@b34e312} 
      #0 28a0ce0 SecondaryHomeHandle0 
     #1 WindowToken{6986f7 type=2024 android.os.BinderProxy@f7da2f6} 
      #0 e17e964 pip-dismiss-overlay 
     #0 WindowToken{dfc6ab6 type=2019 android.os.BinderProxy@e0bfb78} 
      #0 bc113b7 NavigationBar0 
    #4 HideDisplayCutout:18:23 
     #0 OneHanded:18:23 
      #0 FullscreenMagnification:18:23 
       #0 Leaf:18:23 
        #1 WindowToken{2ecacd2 type=2017 android.os.BinderProxy@63e9b5d} 
         #0 7d9c1a3 control_center 
        #0 WindowToken{7960200 type=2017 android.os.BinderProxy@7c7083} 
         #0 b7b0a39 NotificationModalWindowManager 
    #3 OneHanded:17:17 
     #0 FullscreenMagnification:17:17 
      #0 Leaf:17:17 
       #0 WindowToken{37bf169 type=2040 android.os.BinderProxy@80f5833} 
        #0 f89f7ee NotificationShade 
    #2 HideDisplayCutout:16:16 
     #0 OneHanded:16:16 
      #0 FullscreenMagnification:16:16 
       #0 Leaf:16:16 
    #1 OneHanded:15:15 
     #0 FullscreenMagnification:15:15 
      #0 Leaf:15:15 
       #0 WindowToken{ac189b4 type=2000 android.os.BinderProxy@75516c6} 
        #0 194a9dd StatusBar 
    #0 HideDisplayCutout:0:14 
     #0 OneHanded:0:14 
      #1 ImePlaceholder:13:14 
       #0 ImeContainer 
        #0 WindowToken{720aa04 type=2011 android.os.Binder@d506617} 
         #0 f966473 InputMethod 
      #0 FullscreenMagnification:0:12 
       #2 Leaf:3:12 
        #0 WindowToken{ec3f545 type=2038 android.os.BinderProxy@fd8013a} 
         #0 329a9af ShellDropTarget 
       #1 DefaultTaskDisplayArea 
        #2 Task=18 
         #0 Task=19 
          #0 ActivityRecord{296e207 u0 com.miui.home/.launcher.Launcher} t19} 
           #0 9eb98f2 com.miui.home/com.miui.home.launcher.Launcher 
        #1 Task=4 
        #0 Task=5 
         #1 Task=7 
         #0 Task=6 
       #0 Leaf:0:1 
        #1 WallpaperWindowToken{d4d41d3 token=android.os.BinderProxy@2da4fc2} 
         #0 d44d722 com.miui.miwallpaper.wallpaperservice.MiuiKeyguardPictorialWallpaper 
        #0 WallpaperWindowToken{22dbaad token=android.os.Binder@fe41bc4} 
         #0 8db5a66 com.miui.miwallpaper.wallpaperservice.ImageWallpaper 
 
```

可以看到Activity是在DefaultTaskDisplayArea节点下，其他新创建的Window会根据Layer值插入到对应Leaf节点上。



# 4.参考资料

[DisplayArea层级结构（一） —— DisplayArea层级结构的生成](https://juejin.cn/post/7140289958085935141)  
[DisplayArea层级结构（二） —— 向DisplayArea层级结构添加窗口](https://juejin.cn/post/7140289685879783432)  
[DisplayArea层级结构（三） —— 总结](https://juejin.cn/post/7140289813516648456)  
[WMS 层级结构 && DisplayAreaGroup 引入](https://blog.csdn.net/shensky711/article/details/121530510)  
[窗口层次: DisplayArea树](https://blog.csdn.net/jieliaoyuan8279/article/details/123157937)  