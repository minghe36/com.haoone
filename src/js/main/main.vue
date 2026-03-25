<script setup lang="ts">
import { onMounted, ref } from "vue";
import {
  csi,
  evalES,
  openLinkInBrowser,
  subscribeBackgroundColor,
  evalTS,
} from "../lib/utils/bolt";
import "../index.scss";

const PLUGIN_NAME = "haoone AI 字幕";
const VERSION = "9.0.0";

// 语言选项
const LANGUAGE_OPTIONS = ["中文", "英文"];

// 模型选项
const MODEL_OPTIONS = [
  { name: "中英-2026", id: "Qwen3-ASR-0.6B" }
];

// UI 状态
const backgroundColor = ref("#282c34");
const enableOnlineTranscript = ref(false);
const selectedModel = ref(MODEL_OPTIONS[0].name);
const selectedLanguage = ref("中文");
const enableAICorrection = ref(false);
const maxLineLength = ref("25");
const statusMessage = ref("等待任务");
const srtPath = ref("");

// 根据是否启用远程转录来控制模型选择器的显示
const showModelSelector = computed(() => !enableOnlineTranscript.value);

import { computed } from "vue";

const handleVisitWebsite = () => {
  openLinkInBrowser("https://guide.haoai.pro/guide");
};

const handleRunSubtitle = async () => {
  statusMessage.value = "正在导出音频...";
  try {
    // 步骤1: 获取 homePath
    const step1 = await evalES(`
      JSON.stringify((function() {
        try {
          var homePath = "";
          if ($.os.indexOf("Windows") !== -1) {
            homePath = Folder.userData.fsName.replace(/\\\\Roaming$/, "");
          } else {
            homePath = Folder("~/Documents").fsName;
          }
          return { success: true, homePath: homePath };
        } catch(e) {
          return { success: false, error: e.toString() };
        }
      })())
    `, true);
    let p1 = JSON.parse(step1);
    if (!p1.success) {
      statusMessage.value = "失败: " + p1.error;
      return;
    }

    // 步骤2: 检查预设文件
    const presetPath = p1.homePath + "/haoone/plugin_data/config/haoone.epr";
    const step2 = await evalES(`
      JSON.stringify((function() {
        try {
          var f = new File("${presetPath.replace(/\\/g, '\\\\')}");
          return { success: true, exists: f.exists, path: f.fsName };
        } catch(e) {
          return { success: false, error: e.toString() };
        }
      })())
    `, true);
    let p2 = JSON.parse(step2);
    if (!p2.exists) {
      statusMessage.value = "预设文件不存在: " + presetPath;
      return;
    }

    // 步骤3: 检查序列
    const step3 = await evalES(`
      JSON.stringify((function() {
        try {
          var seq = app.project.activeSequence;
          if (!seq) return { success: false, error: "No active sequence" };
          return { success: true, seqName: seq.name };
        } catch(e) {
          return { success: false, error: e.toString() };
        }
      })())
    `, true);
    let p3 = JSON.parse(step3);
    if (!p3.success) {
      statusMessage.value = "失败: " + p3.error;
      return;
    }

    // 步骤4: 执行导出
    statusMessage.value = "正在导出音频...";

    const result = await evalES(
      'app.project.activeSequence.exportAsMediaDirect("/Users/xiewenhao/Documents/haoone/plugin_data/test.wav", "/Users/xiewenhao/Documents/haoone/plugin_data/config/haoone.epr", 0)',
      true
    );

    if (result === "No Error" || result === 0) {
      statusMessage.value = "导出成功";
      srtPath.value = "/Users/xiewenhao/Documents/haoone/plugin_data/test.wav";
    } else {
      statusMessage.value = "导出失败: " + result;
    }
  } catch (error: any) {
    alert("JS错误: " + (error?.message || String(error)));
    statusMessage.value = "导出失败: " + (error?.message || String(error));
  }
};

const handleSyncSubtitle = () => {
  statusMessage.value = "正在同步字幕...";
};

// 测试调用 ExtendScript
const testExtendScript = () => {
  evalTS("testExtendScript").then((res) => {
    console.log("ExtendScript 返回:", res);
    statusMessage.value = `ExtendScript 测试成功: ${res}`;
  }).catch((e) => {
    console.error("ExtendScript 错误:", e);
    statusMessage.value = `ExtendScript 错误: ${e}`;
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
    <div class="haoone-container">
      <!-- 标题栏 -->
      <div class="haoone-header">
        <span class="plugin-name">{{ PLUGIN_NAME }} v{{ VERSION }}</span>
        <button class="help-btn" @click="handleVisitWebsite">
          使用帮助
        </button>
      </div>

      <!-- 提示信息 -->
      <div class="notice-box">
        远程转录：3分钟算1次AI调用，最长支持45分钟视频
      </div>

      <!-- 远程转录与模型选择 -->
      <div class="form-row">
        <label class="form-label">启用远程转录</label>
        <input
          type="checkbox"
          class="checkbox"
          v-model="enableOnlineTranscript"
        />
        <template v-if="showModelSelector">
          <label class="form-label model-label">本地转录模型</label>
          <select
            class="select"
            v-model="selectedModel"
          >
            <option v-for="model in MODEL_OPTIONS" :key="model.id" :value="model.name">
              {{ model.name }}
            </option>
          </select>
        </template>
      </div>

      <!-- 语言选择 -->
      <div class="form-row">
        <label class="form-label">语言</label>
        <select
          class="select"
          v-model="selectedLanguage"
        >
          <option v-for="lang in LANGUAGE_OPTIONS" :key="lang" :value="lang">
            {{ lang }}
          </option>
        </select>
      </div>

      <!-- AI校正与智能拆行 -->
      <div class="form-row">
        <label class="form-label">AI校正与智能拆行</label>
        <input
          type="checkbox"
          class="checkbox"
          v-model="enableAICorrection"
        />
        <label class="form-label max-length-label">单行字幕最大长度</label>
        <input
          type="text"
          class="input-small"
          v-model="maxLineLength"
        />
      </div>

      <!-- 状态显示 -->
      <div class="status-box">
        {{ statusMessage }}
      </div>

      <!-- 操作按钮 -->
      <div class="button-group">
        <button class="btn btn-primary" @click="handleRunSubtitle">
          生成字幕
        </button>
        <button class="btn btn-info" @click="handleSyncSubtitle">
          同步字幕
        </button>
        <button class="btn btn-warning" @click="testExtendScript">
          测试 ExtendScript
        </button>
      </div>

      <!-- SRT路径显示 -->
      <div class="form-row srt-path-row">
        <label class="form-label">SRT路径</label>
        <input
          type="text"
          class="input-path"
          v-model="srtPath"
          readonly
          placeholder="自动生成后显示路径"
        />
      </div>
    </div>
  </div>
</template>

<style lang="scss">
@use "../variables.scss" as *;
</style>