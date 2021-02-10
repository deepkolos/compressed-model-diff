# compressed-model-diff

一个用于对比模型压缩带来变化的工具[在线地址]()

比如使用 KHR_draco_mesh_compression、KHR_mesh_quantization、EXT_meshopt_compression 等压缩后，希望对比与压缩前渲染效果做对比

支持 3 个对比方式，diff/slide/onion

显示模型对比指标信息

0. 模型大小
1. 对象数量
2. 顶点数量
3. 三角面数量
4. 依赖的扩展列表

<div>
  <img src="https://raw.githubusercontent.com/deepkolos/compressed-model-diff/master/screenshot/1.jpg" width="250" alt="" style="display:inline-block;"/>
  <img src="https://raw.githubusercontent.com/deepkolos/compressed-model-diff/master/screenshot/3.jpg" width="250" alt="" style="display:inline-block;"/>
  <img src="https://raw.githubusercontent.com/deepkolos/compressed-model-diff/master/screenshot/4.jpg" width="250" alt="" style="display:inline-block;"/>
</div>

### 压缩工具

KHR_draco_mesh_compression 可使用[gltf-pipeline](https://github.com/CesiumGS/gltf-pipeline)

KHR_mesh_quantization 和 EXT_meshopt_compression 可使用[gltfpack](https://github.com/zeux/meshoptimizer)

目前发现 quantization 的法线精度丢失会导致纹理偏了，因为只用了 4byte 存储

# TODO

0. 计算相机的初始位置
1. 更加自然的控制，最好可以 FPS 游戏一样
