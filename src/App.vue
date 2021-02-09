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

#wireframe-checkbox {
  margin-right: 5px;
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

  <div class="setting" draggable="false">
    <div class="mode-can">
      <div
        v-for="mode in modeCfg.list"
        draggable="false"
        class="mode"
        :class="{
          active: mode === modeCfg.curr,
        }"
        :onclick="() => (modeCfg.curr = mode)"
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

  <div class="drop-can" :class="drop.can" draggable="false">
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
  BoxBufferGeometry,
  Color,
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
            gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
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
          vec3 s_color = texture2D(originalTex, texcoord).xyz * slide;
          vec3 d_color = texture2D(diffTex, texcoord).xyz * (1.0 - slide);
          gl_FragColor = vec4(s_color + d_color, 1.0);
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
      gltfLoader.setMeshoptDecoder(MeshoptDecoder);
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
        // 建立响应依赖的关系，因为reactive里面存储的对象也会被响应式化，但是由于场景是是否大的一棵树，所以不适合
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
