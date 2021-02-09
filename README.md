# compressed-model-diff

一个用于对比模型压缩带来变化的工具[在线地址]()

比如使用 KHR_draco_mesh_compression、KHR_mesh_quantization、EXT_meshopt_compression 等压缩后，希望对比与压缩前渲染效果做对比

支持 3 个对比方式，diff/slide/onion

显示模型对比指标信息

0. 三角面数量
1. 模型大小
2.

### 压缩工具

KHR_draco_mesh_compression 可使用[gltf-pipeline](https://github.com/CesiumGS/gltf-pipeline)

KHR_mesh_quantization 和 EXT_meshopt_compression 可使用[gltfpack](https://github.com/zeux/meshoptimizer)
