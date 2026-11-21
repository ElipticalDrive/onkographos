# Cesium-global-cloud-atmosphere
TAA Ray Marching  Achieved global volumetric cloud and atmospheric scattering enhancement effects
# Cesium 体积云渲染阶段性总结

总算可以完结这个系列了（bushi）。

虽然都是抽空研究，但实现这个效果的时间还是**超出预期**了；不过也正因为这样，算是把**整套渲染管线的完整流程**真正过了一遍，对很多之前“会用但不透”的东西理解深了不少。

## 一些个人总结

- **Shader 真不是人写的**
- **Shader 的移植也是一种能力（装货）**

## 当前实现情况

- 云体使用 raymarching 实现  
- 支持低分辨率渲染（0.5 scale）
- 接入 TAA（Temporal Anti-Aliasing）
- 实现了 **针对 Cesium 的深度遮挡**
  - 当前可以正确遮挡地形
  - 但仍然存在**闪烁问题**
  - 原因暂时未完全定位（大概率和深度 / 时序一致性有关）

## 已知问题 & TODO

- **TAA 仍然有一定拖影**
  - 推测是 velocity / depth 对齐还存在问题
- **云阴影**
  - 先埋个坑
  - 有空再继续研究

## 效果展示

> 上效果

![](./cloud1.png)
![](./cloud2.png)
![](./cloud3.png)
![](./cloud4.png)
![](./cloud5.png)

> 视频效果  
> 显卡：GT 1030  
> 分辨率：0.5  
> 倍速播放（硬件实在过于垃圾）

[【Cesium大气散射+全球体积云效果】B站视频](https://www.bilibili.com/video/BV1CMBeBbEkc/?share_source=copy_web)

## 参考项目
- 不得不说这个作者真的很牛,学习了
- [three-geospatial GitHub]([https://github.com/CesiumGS/cesium](https://github.com/takram-design-engineering/three-geospatial/))
