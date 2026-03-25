<script setup lang="ts">
import { onMounted, ref } from "vue";
import { os, path } from "../lib/cep/node";
import {
  csi,
  evalES,
  openLinkInBrowser,
  subscribeBackgroundColor,
  evalTS,
} from "../lib/utils/bolt";
import "../index.scss";

const backgroundColor = ref("#282c34");

const count = ref(0);

//* Demonstration of Traditional string eval-based ExtendScript Interaction
const jsxTest = () => {
  console.log(evalES(`helloWorld("${csi.getApplicationID()}")`));
};

//* Demonstration of End-to-End Type-safe ExtendScript Interaction
const jsxTestTS = () => {
  evalTS("helloStr", "test").then((res) => {
    console.log(res);
  });
  evalTS("helloNum", 1000).then((res) => {
    console.log(typeof res, res);
  });
  evalTS("helloArrayStr", ["ddddd", "aaaaaa", "zzzzzzz"]).then((res) => {
    console.log(typeof res, res);
  });
  evalTS("helloObj", { height: 90, width: 100 }).then((res) => {
    console.log(typeof res, res);
    console.log(res.x);
    console.log(res.y);
  });
  evalTS("helloVoid").then(() => {
    console.log("function returning void complete");
  });
  evalTS("helloError", "test").catch((e) => {
    console.log("there was an error", e);
  });
};

const nodeTest = () => {
  alert(
    `Node.js ${process.version}\nPlatform: ${
      os.platform
    }\nFolder: ${path.basename(window.cep_node.global.__dirname)}`
  );
};

// 测试调用 ExtendScript
const testExtendScript = () => {
  evalTS("testExtendScript").then((res) => {
    console.log("ExtendScript 返回:", res);
    alert(`ExtendScript 测试成功！返回值: ${res}`);
  }).catch((e) => {
    console.error("ExtendScript 错误:", e);
    alert(`ExtendScript 调用失败: ${e}`);
  });
};

onMounted(() => {
  if (window.cep) {
    subscribeBackgroundColor((c: string) => (backgroundColor.value = c));
  }
});
</script>

<template>
  <div class="app" :style="{ backgroundColor: backgroundColor }">
    <header class="app-header">
      <img src="../assets/bolt-cep.svg" class="icon" />
      <div class="stack-icons">
        <div>
          <img src="../assets/vite.svg" />
          Vite
        </div>
        +
        <div>
          <img src="../assets/vue.svg" />
          Vue
        </div>
        +
        <div>
          <img src="../assets/typescript.svg" />
          TypeScript
        </div>
        +
        <div>
          <img src="../assets/sass.svg" />
          Sass
        </div>
      </div>
      <div class="button-group">
        <button @click="count++">Count is: {{ count }}</button>
        <button @click="nodeTest">
          <img class="icon-button" src="../assets/node-js.svg" />
        </button>
        <button @click="jsxTest">
          <img class="icon-button" src="../assets/adobe.svg" />
        </button>
        <button @click="jsxTestTS">Ts</button>
        <button @click="testExtendScript">测试 ExtendScript</button>
      </div>

      <p>Edit <code>main.vue</code> and save to test HMR updates.</p>
      <p>
        <button
          @click="
            () => openLinkInBrowser('https://github.com/hyperbrew/bolt-cep')
          "
        >
          Bolt Docs
        </button>
        |
        <button @click="() => openLinkInBrowser('https://v3.vuejs.org/')">
          Vue Docs
        </button>
        |
        <button
          @click="
            () => openLinkInBrowser('https://vitejs.dev/guide/features.html')
          "
        >
          Vite Docs
        </button>
      </p>
    </header>
  </div>
</template>

<style lang="scss">
@use "../variables.scss" as *;
</style>
