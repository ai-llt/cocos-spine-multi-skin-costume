# Cocos Creator 3.8 Spine 多部位换装如何省内存？

> 运行时合并多个 Spine skin，实现上装/下装/套装独立切换、互不覆盖。
> 核心思想：**内存大小不再与皮肤数量挂钩，而是与皮肤尺寸种类挂钩。**

---

## 背景

作为刚从前端转行满一年的开发者，在我现在的项目中遇到了这样一个问题：项目角色有大约 12 个可换装部位，每个部位又有海量装备。

遗憾的是美术也是二把刀 —— 给我的 spine 文件将所有皮肤一起合批，显存尖叫。于是我不得不开始探索省内存的换装方案。

起初我觉得复杂度还相对可控，在网上也找到了不少优秀文章。但后面发现我所遭遇的情况完全不同——无论使用哪种方案，都会有大量的内存浪费。

---

## 我找到的方案

| 方案 | 问题 |
|------|------|
| **按部位合图** | 我的部位很多，很多部位几乎是常驻的，跟没合区别不大，还增加了 Draw Call |
| **动态合图 / 运行时合图** | 项目要求实时预览，频繁换装操作下动态合图性能开销太大 |
| **远程图片替换插槽附件** | 程序不知道哪些插槽有用。项目要求展示三个方向，每个部位有三个插槽。在前端插手管理 spine 内部状态让我生理不适 |
| **...其他方案** | 不一一列举了 |

---

## 转折点

某个周末，我灵光一闪，想到了通解：

> **利用 skin 自己管理内部的 slot，我再通过检查哪些插槽有附件，就可以实现上装/下装/套装独立切换、互不覆盖，还能保证内存不爆炸。**

### 核心思路

**让美术按部位给 skin，多部位装扮使用合并 skin 将多个 skin 合并成一个。**

细节：美术打包的 skin 是按照**分类**来打包的，而不需要打包所有该部位的皮肤。Spine 里的默认资源使用透明的默认纹理，供我后续替换成远程纹理。

### Skin 合并逻辑

```
activeSkins = {"shangzhuang", "xiazhuang"}
         ↓
combined = shangzhuang + xiazhuang 的所有附件
         ↓
skeleton.setSkin(combined)
```

每次按钮切换时，全量重新构建合并 skin，不需要维护"之前加没加过"的状态。

---

## 完整实现

### 1. 记录本体插槽

骨骼初始化时，采集角色本体的插槽（手脚、头脸等）。这些是玩家的永久身体部位，不是"装扮"，后续换装时要过滤掉。

```typescript
private baseSlots: Set<string> = new Set();

public onLoad(): void {
  const skeleton = this.spine._skeleton;
  const beforeNames = skeleton.slots
    .filter((slot) => slot.getAttachment() !== null)
    .map((slot) => slot.data.name);
  this.baseSlots = new Set(beforeNames);
}
```

### 2. 管理激活的 Skin

用一个 `Set<string>` 记录当前哪些 skin 是激活的。每个按钮做 toggle：

```typescript
private activeSkins: Set<string> = new Set();

// 上装按钮 —— toggle
public onEquipClick(): void {
  this.activeSkins.has("shangzhuang")
    ? this.activeSkins.delete("shangzhuang")
    : this.activeSkins.add("shangzhuang");
  this.applyCombinedSkin();
}

// 下装按钮 —— toggle
public onUnequipClick(): void {
  this.activeSkins.has("xiazhuang")
    ? this.activeSkins.delete("xiazhuang")
    : this.activeSkins.add("xiazhuang");
  this.applyCombinedSkin();
}
```

### 3. 合并并应用 Skin

从 `skeletonData` 中取出各个 skin，用 `addSkin` 合并成一个新 skin，然后 `setSkin`：

```typescript
private applyCombinedSkin(): void {
  const runtimeData = this.spine.skeletonData?.getRuntimeData();
  if (!runtimeData) return;

  const combined = new sp.spine.Skin("combined");
  for (const name of this.activeSkins) {
    const skin = runtimeData.findSkin(name);
    if (skin) combined.addSkin(skin);
  }
  this.spine._skeleton.setSkin(combined);
  this.spine.setSlotsToSetupPose();
}
```

### 4. 过滤本体 + 按需加载纹理

换装后对比 `baseSlots`，只处理新增的装扮插槽：

```typescript
const newSlots = this.spine._skeleton.slots.filter(
  (slot) =>
    slot.getAttachment() !== null && !this.baseSlots.has(slot.data.name)
);
```

对每个新插槽按需加载远程纹理：

```typescript
// 只有激活的插槽才加载纹理 → 内存可控
newSlots.forEach((slot) => {
  const tex2d = await loadRemoteImage(slot.data.name); // 伪代码
  this.spine.setSlotTexture(slot.data.name, tex2d, true);
});
```

---

## API 参考

| API | 说明 | 注意 |
|-----|------|------|
| `this.spine._skeleton` | 底层 spine Skeleton 对象 | Creator 3.8 已暴露，直接访问 |
| `skeleton.slots` | 所有插槽列表 | `sp.spine.Slot[]` |
| `slot.getAttachment()` | 获取附件 | **方法**，不是属性（WASM runtime） |
| `slot.data.name` | 插槽名称 | **属性**，可直接访问 |
| `skeleton.setSkin(skin)` | 应用 skin | 先 `addSkin` 合并再调用 |
| `skeleton.setSlotTexture()` | 替换插槽纹理 | 第三个参数 `true` = 新建附件实例 |

---

## 贴士

- **`addSkin` 是累加**：重复 add 同一个 skin 不会报错，但建议在外部做好 toggle 逻辑。
- **`setSlotsToSetupPose()` 必须在 `setSkin` 后调用**，否则 slot attachment 不会被填充。
- **本体插槽只需要采集一次**，在 `onLoad` 中记录即可。
- **skin 数据来自 spine 源文件**（`.skel` / `.json`），运行时 `findSkin` 只能查找已存在的 skin。

## 最终方案

```
① onLoad 时记录本体插槽
          ↓
② 按钮切换 → 更新 activeSkins 集合
          ↓
③ 合并 activeSkins 中所有 skin → setSkin
          ↓
④ 仅对新增的激活插槽按需加载纹理 → setSlotTexture
```

**不预加载、不全部挂载、不互相覆盖。内存大小与皮肤数量解耦。**

---

_基于 Cocos Creator 3.8 · Spine WASM runtime · 2026-07-14_
