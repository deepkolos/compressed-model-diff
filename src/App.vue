<style lang="less">
* {
  padding: 0;
  margin: 0;
}

body,
html,
#app,
.canvas {
  width: 100%;
  height: 100%;
  user-select: none;
  overflow: hidden;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}

.title {
  position: absolute;
  top: 20px;
  left: 0;
  right: 0;
  margin: 0 auto;
  font-size: larger;
  width: min-content;
  white-space: nowrap;
  color: black;
  text-decoration: none;
  text-shadow: 0 0 4px white;
  // color: white;
  // mix-blend-mode: difference;
  // background-clip: text;
  // -webkit-background-clip: text;
}

.canvas {
  display: block;
}

.setting {
  position: absolute;
  right: 0;
  left: 0;
  bottom: 15px;
  margin: 0 auto;
  width: min-content;
  padding: 5px;
}

.mode {
  flex: 1;
  cursor: pointer;
  text-align: center;
  padding: 3px 5px;

  &:hover,
  &.active {
    color: #123456;
    background-color: #eee;
  }

  &-can {
    display: flex;
    background-color: hsl(0deg 0% 100% / 62%);
    margin-bottom: 5px;
  }
}

.slider {
  margin-top: 5px;
  width: 200px;
}

.drop {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: larger;
  opacity: 0;
  transition: opacity ease 0.33s;

  &-can {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    display: flex;
    pointer-events: none;

    &.enter {
      pointer-events: all;
    }
  }

  &.enter {
    opacity: 1;
    background-color: #1234567a;
  }
}

.info {
  &-can {
    display: grid;
    position: absolute;
    right: 10px;
    top: 10px;
    grid-template-columns: repeat(3, auto);
    // grid-template-rows: repeat(4, 20px);
    grid-gap: 10px;
    background-color: white;
    padding: 5px;
    font-size: small;
    opacity: 0.3;
    transition: opacity ease 0.33s;

    &:hover {
      opacity: 1;
    }
  }

  &-first {
    grid-column-start: 2;
  }

  &-metrics {
    text-align: right;

    &::after {
      content: ':';
    }
  }
}

.usage {
  padding: 20px;
  margin: auto;
  white-space: nowrap;
  background: #123456;
  color: white;

  &-can {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    display: flex;
    align-items: center;
    justify-content: center;
  }
}

#wireframe-checkbox {
  margin-right: 5px;
}

.fade {
  @duration: 0.53s;
  @offsetY: 2%;

  &-enter {
    &-active {
      opacity: 0;
      transform: translateY(@offsetY);
      transition: all ease @duration;
    }

    &-to {
      transform: translateY(0%);
      opacity: 1;
    }
  }

  &-leave {
    &-active {
      opacity: 1;
      transform: translateY(0%);
      transition: all ease @duration;
    }

    &-to {
      opacity: 0;
      transform: translateY(-@offsetY);
    }
  }
}
</style>

<template>
  <a
    class="title"
    href="https://github.com/deepkolos/compressed-model-diff"
    draggable="false"
    >Compressed-Model-Diff</a
  >
  <canvas class="canvas" ref="canvas" />

  <transition name="fade" appear>
    <div class="usage-can" v-if="!drop.leftFile && !drop.rightFile">
      <div class="usage">请把模型文件拖入左右两边</div>
    </div>
  </transition>

  <div class="setting" draggable="false">
    <div class="mode-can">
      <div
        v-for="mode in modeCfg.list"
        draggable="false"
        class="mode"
        :class="{
          active: mode === modeCfg.curr,
        }"
        @click="() => (modeCfg.curr = mode)"
      >
        {{ mode }}
      </div>
    </div>
    <label for="wireframe-checkbox">
      <input
        id="wireframe-checkbox"
        type="checkbox"
        v-model="modeCfg.wireframe"
      />线框模式
      <!-- <input type="checkbox" v-model="modeCfg.point" />顶点-->
    </label>
    <input
      class="slider"
      type="range"
      min="0"
      max="100"
      step="1"
      v-model="modeCfg.slide"
    />
  </div>

  <div class="info-can">
    <div class="info-first">left</div>
    <div>right</div>

    <div class="info-metrics">size</div>
    <div>{{ (modelInfo.original.size / 1024 / 1024).toFixed(2) }}MB</div>
    <div>{{ (modelInfo.diff.size / 1024 / 1024).toFixed(2) }}MB</div>
    <div class="info-metrics">objects</div>
    <div>{{ modelInfo.original.objects }}</div>
    <div>{{ modelInfo.diff.objects }}</div>
    <div class="info-metrics">vertices</div>
    <div>{{ modelInfo.original.vertices }}</div>
    <div>{{ modelInfo.diff.vertices }}</div>
    <div class="info-metrics">triangles</div>
    <div>{{ modelInfo.original.triangles }}</div>
    <div>{{ modelInfo.diff.triangles }}</div>
    <div class="info-metrics">extensions</div>
    <div>
      <div v-for="ext in modelInfo.original.ext">{{ ext }}</div>
    </div>
    <div>
      <div v-for="ext in modelInfo.diff.ext">{{ ext }}</div>
    </div>
  </div>

  <div class="drop-can" :class="drop.can" draggable="false">
    <div
      class="drop"
      :class="drop.left"
      @dragenter="() => (drop.left = 'enter')"
      @dragleave="() => (drop.left = '')"
      @drop="onOriginalModelDrop"
    >
      左模型
    </div>
    <div
      class="drop"
      :class="drop.right"
      @dragenter="() => (drop.right = 'enter')"
      @dragleave="() => (drop.right = '')"
      @drop="onDiffModelDrop"
    >
      右模型
    </div>
  </div>
</template>

<script lang="ts">
import { ref, onMounted, reactive, watchEffect } from 'vue';
import {
  AmbientLight,
  BoxBufferGeometry,
  DirectionalLight,
  Mesh,
  MeshBasicMaterial,
  Object3D,
  OrthographicCamera,
  PerspectiveCamera,
  PlaneBufferGeometry,
  Scene,
  ShaderMaterial,
  sRGBEncoding,
  Vector2,
  WebGL1Renderer,
  WebGLRenderTarget,
} from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { GLTF, GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
// @ts-ignore
import { MeshoptDecoder } from 'three/examples/jsm/libs/meshopt_decoder.module';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader';

const EXT_MAP: { [k: string]: string } = {
  KHR_draco_mesh_compression: 'draco',
  KHR_texture_transform: 'texture_transform',
  KHR_mesh_quantization: 'quantization',
  EXT_meshopt_compression: 'meshopt',
};

export default {
  setup() {
    // const baseUrl = 'http://192.168.10.140:8080/';
    // const baseUrl = 'http://127.0.0.1:8080/';
    const gltfLoader = new GLTFLoader();
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('draco/');

    const canvas = ref<HTMLCanvasElement>();
    const modeCfg = reactive({
      wireframe: 'false',
      point: 'false',
      slide: '50',
      curr: 'slide',
      list: ['diff', 'slide', 'onion'],
    });
    const drop = reactive({
      left: '',
      right: '',
      can: '',
      leftFile: '',
      rightFile: '',
      // leftFile: baseUrl + 'PrimaryIonDrive.glb',
      // rightFile: baseUrl + 'PrimaryIonDrive-EXT_MESH_QUANTIZATION.glb',
    });
    const modelUpdate = reactive({
      original: 0,
      diff: 0,
    });
    const modelInfo = reactive({
      original: {
        size: 0,
        objects: 0,
        vertices: 0,
        triangles: 0,
        ext: [],
      },
      diff: {
        size: 0,
        objects: 0,
        vertices: 0,
        triangles: 0,
        ext: [],
      },
    });

    const onDiffModelDrop = (e: DragEvent) => {
      e.preventDefault();
      if (e.dataTransfer) {
        const file = e.dataTransfer.files.item(0);
        if (file) {
          drop.rightFile = URL.createObjectURL(file);
        }
      }
      drop.right = '';
      drop.can = '';
    };
    const onOriginalModelDrop = (e: DragEvent) => {
      e.preventDefault();
      if (e.dataTransfer) {
        const file = e.dataTransfer.files.item(0);
        if (file) {
          drop.leftFile = URL.createObjectURL(file);
        }
      }
      drop.left = '';
      drop.can = '';
    };

    window.addEventListener('dragover', e => e.preventDefault());
    window.addEventListener('dragenter', () => (drop.can = 'enter'));
    window.addEventListener('dragleave', () => {
      if (!drop.left && !drop.right) drop.can = '';
    });

    const createDiffModeScene = (
      original: WebGLRenderTarget,
      diff: WebGLRenderTarget,
    ) => {
      const scene = new Scene();
      const geometry = new PlaneBufferGeometry(2, 2, 1, 1);
      const material = new ShaderMaterial({
        uniforms: {
          originalTex: { value: original.texture },
          diffTex: { value: diff.texture },
        },
        depthWrite: false,
        vertexShader: `
        varying vec2 texcoord;

        void main() {
          texcoord = uv;
          gl_Position = vec4(position.xy, 0.0, 1.0);
        }`,
        fragmentShader: `
        varying vec2 texcoord;
        uniform sampler2D originalTex;
        uniform sampler2D diffTex;

        void main() {
          vec4 originalColor = texture2D(originalTex, texcoord);
          vec4 diffColor = texture2D(diffTex, texcoord);
          vec4 diff = originalColor - diffColor;

          if (diff.x <= 0.001 && diff.y <= 0.001 && diff.z <= 0.001) {
            gl_FragColor = vec4(0.0);
          } else {
            gl_FragColor = vec4(vec3(diff.x + diff.y + diff.z), 1.0);
          }
        }
        `,
      });
      const mesh = new Mesh(geometry, material);

      const geometryPreview = new BoxBufferGeometry(0.5, 0.5, 1, 1); // new PlaneBufferGeometry(1, 1, 1, 1);
      const meshOriginal = new Mesh(
        geometryPreview,
        new MeshBasicMaterial({ map: original.texture }),
      );
      const meshDiff = new Mesh(
        geometryPreview,
        new MeshBasicMaterial({ map: diff.texture }),
      );

      const offset = (2 - 0.5) / 2;
      meshOriginal.position.set(-offset, offset, 0);
      meshDiff.position.set(offset, offset, 0);
      meshDiff.scale.set(-1, -1, 1);
      meshOriginal.scale.set(-1, -1, 1);

      scene.add(mesh, meshOriginal, meshDiff);
      return scene;
    };

    const createSlideModeScene = (
      original: WebGLRenderTarget,
      diff: WebGLRenderTarget,
    ) => {
      const scene = new Scene();
      const geometry = new PlaneBufferGeometry(2, 2, 1, 1);
      const material = new ShaderMaterial({
        uniforms: {
          originalTex: { value: original.texture },
          diffTex: { value: diff.texture },
          slide: { value: 0.5 },
        },
        depthWrite: false,
        vertexShader: `
        varying vec2 texcoord;

        void main() {
          texcoord = uv;
          gl_Position = vec4(position.xy, 0.0, 1.0);
        }`,
        fragmentShader: `
        varying vec2 texcoord;
        uniform sampler2D originalTex;
        uniform sampler2D diffTex;
        uniform float slide;

        void main() {
          if (texcoord.x < slide) {
            gl_FragColor = texture2D(originalTex, texcoord);
          } else {
            gl_FragColor = texture2D(diffTex, texcoord);
          }
        }
        `,
      });
      const mesh = new Mesh(geometry, material);

      watchEffect(() => {
        material.uniforms.slide.value = ~~modeCfg.slide / 100;
      });

      scene.add(mesh);
      return scene;
    };

    const createOnionModeScene = (
      original: WebGLRenderTarget,
      diff: WebGLRenderTarget,
    ) => {
      const scene = new Scene();
      const geometry = new PlaneBufferGeometry(2, 2, 1, 1);
      const material = new ShaderMaterial({
        uniforms: {
          originalTex: { value: original.texture },
          diffTex: { value: diff.texture },
          slide: { value: 0.5 },
        },
        depthWrite: false,
        vertexShader: `
        varying vec2 texcoord;

        void main() {
          texcoord = uv;
          gl_Position = vec4(position.xy, 0.0, 1.0);
        }`,
        fragmentShader: `
        varying vec2 texcoord;
        uniform sampler2D originalTex;
        uniform sampler2D diffTex;
        uniform float slide;

        void main() {
          vec3 s_color = texture2D(originalTex, texcoord).xyz * slide;
          vec3 d_color = texture2D(diffTex, texcoord).xyz * (1.0 - slide);
          gl_FragColor = vec4(s_color + d_color, 1.0);
        }
        `,
      });
      const mesh = new Mesh(geometry, material);

      watchEffect(() => {
        material.uniforms.slide.value = ~~modeCfg.slide / 100;
      });

      scene.add(mesh);
      return scene;
    };

    onMounted(async () => {
      const { innerWidth: w, innerHeight: h } = window;
      gltfLoader.setMeshoptDecoder(MeshoptDecoder);
      gltfLoader.setDRACOLoader(dracoLoader);
      const renderer = new WebGL1Renderer({
        canvas: canvas.value,
        antialias: true,
        alpha: true,
      });
      renderer.setSize(w, h);
      renderer.setPixelRatio(devicePixelRatio);
      renderer.outputEncoding = sRGBEncoding;

      window.addEventListener('resize', () => {
        const { innerWidth: w, innerHeight: h } = window;
        renderer.setSize(w, h);
        camera.aspect = w / h;
        camera.updateProjectionMatrix();
        renderer.setSize(w, h);
      });

      const drawRect = new Vector2();
      renderer.getDrawingBufferSize(drawRect);

      const camera = new PerspectiveCamera(75, w / h, 0.1, 1000);
      const flatCamera = new OrthographicCamera(-1, 1, -1, 1, 0.1, 1000);
      camera.position.z = 0.2;

      const originalTarget = new WebGLRenderTarget(drawRect.x, drawRect.y);
      const diffTarget = new WebGLRenderTarget(drawRect.x, drawRect.y);
      originalTarget.texture.encoding = sRGBEncoding;
      diffTarget.texture.encoding = sRGBEncoding;

      // setup controls
      const controls = new OrbitControls(camera, canvas.value);
      controls.enableDamping = true;
      controls.dampingFactor = 0.75;

      // setup scene
      const originalScene = new Scene();
      const diffScene = new Scene();
      const diffModeScene = createDiffModeScene(originalTarget, diffTarget);
      const slideModeScene = createSlideModeScene(originalTarget, diffTarget);
      const onionModeScene = createOnionModeScene(originalTarget, diffTarget);
      let originalModel: GLTF;
      let diffModel: GLTF;

      watchEffect(() => {
        drop.leftFile &&
          gltfLoader
            .loadAsync(drop.leftFile, e => (modelInfo.original.size = e.total))
            .then(model => {
              if (originalModel) originalScene.remove(originalModel.scene);
              originalModel = model;
              originalScene.add(model.scene);
              modelUpdate.original++;

              modelInfo.original.ext = (
                model.parser.json.extensionsUsed || []
              ).map((i: string) => EXT_MAP[i] || i);
            });
      });

      watchEffect(async () => {
        drop.rightFile &&
          gltfLoader
            .loadAsync(drop.rightFile, e => (modelInfo.diff.size = e.total))
            .then(model => {
              if (diffModel) diffScene.remove(diffModel.scene);
              diffModel = model;
              diffScene.add(model.scene);
              modelUpdate.diff++;
              modelInfo.diff.ext = (model.parser.json.extensionsUsed || []).map(
                (i: string) => EXT_MAP[i] || i,
              );
            });
      });

      watchEffect(() => {
        const originalInfo = {
          objects: 0,
          vertices: 0,
          triangles: 0,
        };
        const diffInfo = {
          objects: 0,
          vertices: 0,
          triangles: 0,
        };

        const walker = (child: Object3D, info: typeof originalInfo) => {
          if (child instanceof Mesh && child.material) {
            // @ts-ignore
            child.material.wireframe = modeCfg.wireframe;
          }

          // 统计顶点，三角形，object数目
          info.objects++;
          if (child instanceof Mesh) {
            const geometry = child.geometry;
            if (geometry.isBufferGeometry) {
              info.vertices += geometry.attributes.position.count;

              if (geometry.index !== null) {
                info.triangles += geometry.index.count / 3;
              } else {
                info.triangles += geometry.attributes.position.count / 3;
              }
            }
          }
        };
        // 建立响应依赖的关系，因为reactive里面存储的对象也会被响应式化，但是由于场景是是否大的一棵树，所以不适合
        console.log(modelUpdate.original, modelUpdate.diff);

        originalModel?.scene.traverse(obj => walker(obj, originalInfo));
        diffModel?.scene.traverse(obj => walker(obj, diffInfo));

        modelInfo.original = { ...modelInfo.original, ...originalInfo };
        modelInfo.diff = { ...modelInfo.diff, ...diffInfo };
      });

      // setup lights
      originalScene.add(new AmbientLight(0xffffff, 1.0)); // can't share lights between scenes
      originalScene.add(new DirectionalLight(0xffffff, 1.0));
      diffScene.add(new AmbientLight(0xffffff, 1.0));
      diffScene.add(new DirectionalLight(0xffffff, 1.0));

      const MODE_MAP = {
        diff: diffModeScene,
        slide: slideModeScene,
        onion: onionModeScene,
      };

      // render loop
      const animate = () => {
        requestAnimationFrame(animate);
        controls.update();
        renderer.setRenderTarget(originalTarget);
        renderer.render(originalScene, camera);
        renderer.setRenderTarget(diffTarget);
        renderer.render(diffScene, camera);
        renderer.setRenderTarget(null);
        // @ts-ignore
        renderer.render(MODE_MAP[modeCfg.curr], flatCamera);
      };

      animate();
    });

    // expose to template
    return {
      canvas,
      modeCfg,
      drop,
      modelInfo,
      onDiffModelDrop,
      onOriginalModelDrop,
    };
  },
};
</script>
