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
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}

.canvas {
  display: block;
}

.setting {
  position: absolute;
  right: 0;
  left: 0;
  bottom: 10px;
  margin: 0 auto;
  width: min-content;
}

.mode {
  flex: 1;
  cursor: pointer;
  text-align: center;
  padding: 2px 4px;

  &:hover,
  &.active {
    color: #123456;
    background-color: #eee;
  }

  &-can {
    display: flex;
  }
}

.slider {
  margin-top: 5px;
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
</style>

<template>
  <canvas class="canvas" ref="canvas" />

  <div class="setting">
    <div class="mode-can">
      <div
        class="mode"
        v-for="mode in modeCfg.list"
        :class="{
          active: mode === modeCfg.curr,
        }"
        :onclick="() => (modeCfg.curr = mode)"
      >
        {{ mode }}
      </div>
    </div>
    <div>
      <input type="checkbox" v-model="modeCfg.wireframe" />网格
      <!-- <input type="checkbox" v-model="modeCfg.point" />顶点-->
    </div>
    <input
      class="slider"
      type="range"
      min="0"
      max="100"
      step="1"
      v-model="modeCfg.slide"
    />
  </div>

  <div class="drop-can" :class="drop.can">
    <div
      class="drop"
      :ondragenter="() => (drop.left = 'enter')"
      :ondragleave="() => (drop.left = '')"
      :ondrop="onOriginalModelDrop"
      :class="drop.left"
    >
      原模型
    </div>
    <div
      class="drop"
      :ondragenter="() => (drop.right = 'enter')"
      :ondragleave="() => (drop.right = '')"
      :ondrop="onDiffModelDrop"
      :class="drop.right"
    >
      对比模型
    </div>
  </div>
</template>

<script lang="ts">
import { ref, onMounted, reactive, watchEffect } from 'vue';
import {
  AmbientLight,
  DirectionalLight,
  Mesh,
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

export default {
  setup() {
    const { innerWidth: w, innerHeight: h } = window;
    // const baseUrl = 'http://192.168.10.140:8080/';
    const baseUrl = 'http://127.0.0.1:8080/';
    const gltfLoader = new GLTFLoader();

    const canvas = ref<HTMLCanvasElement>();
    const modeCfg = reactive({
      wireframe: false,
      point: false,
      slide: 50,
      curr: 'slide',
      list: ['diff', 'slide', 'onion'],
    });
    const drop = reactive({
      left: '',
      right: '',
      can: '',
      leftFile: baseUrl + 'PrimaryIonDrive.glb',
      rightFile: baseUrl + 'PrimaryIonDrive-EXT_MESH_QUANTIZATION.glb',
    });
    const modelUpdate = reactive({
      original: 0,
      diff: 0,
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

    gltfLoader.setMeshoptDecoder(MeshoptDecoder);

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

          gl_FragColor = vec4((originalColor - diffColor).xyz, 1.0);
          // gl_FragColor = texture2D(originalTex, texcoord);
          // gl_FragColor = texture2D(diffTex, texcoord);
        }
        `,
      });
      const mesh = new Mesh(geometry, material);
      scene.add(mesh);
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
        material.uniforms.slide.value = modeCfg.slide / 100;
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
          gl_FragColor = normalize(texture2D(originalTex, texcoord) * slide + texture2D(diffTex, texcoord) * (1.0 - slide));
        }
        `,
      });
      const mesh = new Mesh(geometry, material);

      watchEffect(() => {
        material.uniforms.slide.value = modeCfg.slide / 100;
      });

      scene.add(mesh);
      return scene;
    };

    onMounted(async () => {
      const renderer = new WebGL1Renderer({
        canvas: canvas.value,
        antialias: true,
        alpha: true,
      });
      renderer.setSize(w, h);
      renderer.setPixelRatio(devicePixelRatio);
      renderer.outputEncoding = sRGBEncoding;

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
        gltfLoader.loadAsync(drop.leftFile).then(model => {
          if (originalModel) originalScene.remove(originalModel.scene);
          originalModel = model;
          originalScene.add(model.scene);
          modelUpdate.original++;
        });
      });

      watchEffect(async () => {
        gltfLoader.loadAsync(drop.rightFile).then(model => {
          if (diffModel) diffScene.remove(diffModel.scene);
          diffModel = model;
          diffScene.add(model.scene);
          modelUpdate.diff++;
        });
      });

      watchEffect(() => {
        const walker = (child: Object3D) => {
          if (child instanceof Mesh && child.material) {
            // @ts-ignore
            child.material;
            // @ts-ignore
            child.material.wireframe = modeCfg.wireframe;
            // @ts-ignore
            // child.material.flatShading = modeCfg.wireframe;
            // @ts-ignore
            // child.material.vertexColors = modeCfg.wireframe;
            // child.material.map = null;
          }
        };
        // 建立依赖关系
        console.log(modelUpdate.original, modelUpdate.diff);
        originalModel?.scene.traverse(walker);
        diffModel?.scene.traverse(walker);
      });

      // const originalPoints = new Group();
      // const diffPoints = new Group();
      // // setup points scene
      // let maxLen = 1;
      // const walker = (child: Object3D, group: Group) => {
      //   if (child instanceof Mesh) {
      //     if (maxLen > 0) {
      //       group.add(new Points(child.geometry, pointMaterial));
      //       maxLen--;
      //     }
      //   }
      // };
      // originalModel.scene.traverse(i => walker(i, originalPoints));
      // diffModel.scene.traverse(i => walker(i, diffPoints));
      // originalScene.add(originalPoints);
      // diffModeScene.add(diffPoints);

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
      onDiffModelDrop,
      onOriginalModelDrop,
    };
  },
};
</script>
