<script setup lang="ts">
import { onMounted, ref } from "vue";
import { csi, evalES, openLinkInBrowser, subscribeBackgroundColor } from "../lib/utils/bolt";
import { child_process, os, path } from "../lib/cep/node";
import "../index.scss";

const { spawn } = child_process;
const fs = require("fs");

// 获取用户文档目录路径 (Node.js)
const getUserDocsPath = (): string => {
  return os.homedir() + "/Documents";
};

// 使用 Node.js 查找 haoone 程序路径
const findHaoonePath = (): string => {
  const isWindows = process.platform === "win32";
  const home = os.homedir();

  const paths: string[] = isWindows
    ? [
        // 优先使用环境变量获取路径
        process.env.PROGRAMFILES && path.join(process.env.PROGRAMFILES, "haoone", "haoone.exe"),
        process.env["PROGRAMFILES(X86)"] && path.join(process.env["PROGRAMFILES(X86)"], "haoone", "haoone.exe"),
        process.env.LOCALAPPDATA && path.join(process.env.LOCALAPPDATA, "haoone", "haoone.exe"),
        // 兜底硬编码路径
        "C:\\Program Files\\haoone\\haoone.exe",
        path.join(home, "AppData", "Local", "haoone", "haoone.exe")
      ].filter(Boolean) as string[]
    : [
        "/Applications/haoone.app/Contents/MacOS/haoone",
        "/usr/local/bin/haoone",
        "/opt/haoone/haoone",
        path.join(home, ".local/bin/haoone"),
      ];

  return paths.find(p => fs.existsSync(p)) || "";
};

const PLUGIN_NAME = "haoone AI 字幕";
const VERSION = "9.0.0";

// 语言选项
const LANGUAGE_OPTIONS = ["中文", "英文"];

// 模型选项
const MODEL_OPTIONS = [
  { name: "中英-2026", id: "Qwen3-ASR-0.6B" }
];

// 语言映射
const languageMap: Record<string, string> = {
  "中文": "zh",
  "英文": "en"
};

// 模型映射
const modelMap: Record<string, string> = {
  "中英-2026": "Qwen3-ASR-0.6B"
};

// UI 状态
const backgroundColor = ref("#282c34");
const enableOnlineTranscript = ref(false);
const selectedModel = ref(MODEL_OPTIONS[0].name);
const selectedLanguage = ref("中文");
const enableAICorrection = ref(false);
const maxLineLength = ref("25");
const statusMessage = ref("等待任务");
const srtPath = ref("");

// 将字符串转换为 Unicode 转义序列（用于 ExtendScript）
const toUnicodeEscape = (str: string): string => {
  return str.replace(/[\u0080-\uffff]/g, (c) => {
    return "\\u" + ("0000" + c.charCodeAt(0).toString(16)).slice(-4);
  });
};

// 从 Unicode 转义序列还原字符串
const fromUnicodeEscape = (str: string): string => {
  return str.replace(/\\u([0-9a-fA-F]{4})/g, (match, hex) => {
    return String.fromCharCode(parseInt(hex, 16));
  });
};

// 根据是否启用远程转录来控制模型选择器的显示
const showModelSelector = computed(() => !enableOnlineTranscript.value);

import { computed } from "vue";

const handleVisitWebsite = () => {
  openLinkInBrowser("https://guide.haoai.pro/guide");
};

const handleOpenSrtDirectory = () => {
  if (!srtPath.value) return;
  // 获取字幕文件所在目录路径
  const srtDir = srtPath.value.substring(0, srtPath.value.lastIndexOf("/"));
  openLinkInBrowser("file://" + srtDir);
};

const handleRunSubtitle = async () => {
  statusMessage.value = "导出音频中...";
  try {
    // 步骤1: 获取 Documents 路径
    // 步骤1: 使用 Node.js 获取 Documents 路径
    const homePath = getUserDocsPath();

    // 步骤2: 检查预设文件是否存在 (使用 Node.js)
    const presetPath = homePath + "/haoone/plugin_data/config/haoone.epr";
    if (!fs.existsSync(presetPath)) {
      statusMessage.value = "预设文件不存在: " + presetPath;
      return;
    }

    // 步骤3: 检查序列 (使用简化版本)
    const step3 = await evalES('app.project.activeSequence ? app.project.activeSequence.name : ""', true);
    if (!step3 || step3 === "undefined" || step3 === "") {
      statusMessage.value = "没有活动的序列";
      return;
    }

    // 步骤4: 执行导出

    // 生成输出文件名
    const projectName = (await evalES('app.project.name || "project"', true))
      .replace(/\.prproj$/, "")  // 去掉 .prproj 扩展名
      .replace(/[<>:"/\\|?*]/g, "_");
    const seqName = (await evalES('app.project.activeSequence.name || "sequence"', true)).replace(/[<>:"/\\|?*]/g, "_");
    const outputFile = homePath + "/haoone/plugin_data/" + projectName + "_" + seqName + ".wav";
    const exportPresetPath = homePath + "/haoone/plugin_data/config/haoone.epr";

    const result = await evalES(
      'app.project.activeSequence.exportAsMediaDirect("' + outputFile + '", "' + exportPresetPath + '", 0)',
      true
    );

    if (result === "No Error" || result === 0) {
      statusMessage.value = "音频导出完成";

      // 获取 hooone 程序路径 - 简化版本
      const haoonePath = findHaoonePath();
      if (!haoonePath) {
        statusMessage.value = "未找到 haoone 程序";
        return;
      }

      // 获取参数
      const language = languageMap[selectedLanguage.value] || "zh";
      const modelId = modelMap[selectedModel.value] || "Qwen3-ASR-0.6B";
      const isAICorrection = enableAICorrection.value ? "true" : "false";
      const isOnline = enableOnlineTranscript.value ? "true" : "false";
      const maxLength = maxLineLength.value;

      // 获取时间线名称
      const timelineName = seqName;
      statusMessage.value = "运行转录命令...";
      // 构建命令
      const hoooneCmd = '"' + haoonePath + '" timeline-transcribe' +
        ' --timeline-name "' + timelineName + '"' +
        ' --audio-file-path "' + outputFile.replace(/\\/g, "/") + '"' +
        ' --language ' + language +
        ' --enable-ai-correction ' + isAICorrection +
        ' --enable-online-transcript ' + isOnline +
        ' --max-subtitle-length ' + maxLength +
        ' --model ' + modelId;

      // 使用 Node.js 执行命令 - 实时显示进度
      return new Promise((resolve, reject) => {
        const args = hoooneCmd.split(' ').slice(1); // 去掉路径，只保留参数
        const cmdPath = hoooneCmd.match(/^"([^"]+)"/)?.[1] || hoooneCmd.split(' ')[0].replace(/"/g, '');

        // 清理参数中的引号
        const cleanArgs = args.map(arg => arg.replace(/^"|"$/g, ''));

        const proc = spawn(cmdPath, cleanArgs, {
          shell: true,
          encoding: 'utf8'
        });

        let output = "";
        let lastProgress = "";
        let errorMessage = "";

        // 提取进度的统一函数
        const processOutput = (str: string) => {
          // 实时检查是否有错误
          const errorMatch = str.match(/haoone_error=(.+?)(?:\n|$)/);
          if (errorMatch && errorMatch[1]) {
            errorMessage = errorMatch[1].trim();
            statusMessage.value = "错误: " + errorMessage;
          }

          // 提取进度消息 (==== 和 ==== 之间的内容)
          const msgMatch = str.match(/====([\s\S]*?)====/);
          if (msgMatch && msgMatch[1]) {
            const lines = msgMatch[1].split(/\r?\n/);
            for (const line of lines) {
              const trimmed = line.trim();
              if (trimmed && trimmed !== lastProgress) {
                lastProgress = trimmed;
                statusMessage.value = trimmed;
              }
            }
          }
        };

        proc.stdout.on('data', (data) => {
          const str = data.toString();
          output += str;
          processOutput(str);
        });

        proc.stderr.on('data', (data) => {
          const str = data.toString();
          output += str;
          processOutput(str);
        });

        proc.on('close', async (code) => {
          // 解析输出结果
          let srtFilePath = "";
          let jsonFilePath = "";
          let errorMessage = "";

          // 提取 srt_file_path
          const srtMatch = output.match(/srt_file_path=(.+?)(?:\n|$)/);
          if (srtMatch && srtMatch[1]) {
            srtFilePath = srtMatch[1].trim();
          }

          // 提取 json_file_path
          const jsonMatch = output.match(/json_file_path=(.+?)(?:\n|$)/);
          if (jsonMatch && jsonMatch[1]) {
            jsonFilePath = jsonMatch[1].trim();
          }

          // 检查是否有错误
          const errorMatch = output.match(/haoone_error=(.+?)(?:\n|$)/);
          if (errorMatch && errorMatch[1]) {
            errorMessage = errorMatch[1].trim();
          }

          if (errorMessage) {
            statusMessage.value = "转录失败: " + errorMessage;
          } else if (srtFilePath) {
            // 转录完成，导入字幕到项目面板
            statusMessage.value = "正在导入字幕到项目面板...";
            srtPath.value = srtFilePath;

            // 保存配置到 pr-plugin.json
            try {
              const userDocsPath = getUserDocsPath();
              const configDir = userDocsPath + "/haoone/plugin_data/config";
              const configPath = configDir + "/pr-plugin.json";

              // 确保目录存在
              if (!fs.existsSync(configDir)) {
                fs.mkdirSync(configDir, { recursive: true });
              }

              // 读取或创建配置
              let config: Record<string, any> = {};
              if (fs.existsSync(configPath)) {
                try {
                  config = JSON.parse(fs.readFileSync(configPath, "utf8"));
                } catch (e) {
                  config = {};
                }
              }

              // 保存项目名_时间线名 -> srt路径
              const key = projectName + "_" + seqName;
              config[key] = srtFilePath;

              // 保存全局配置
              config["enable-online-transcript"] = enableOnlineTranscript.value;
              config["max-subtitle-length"] = parseInt(maxLineLength.value) || 25;
              config["enable-ai-correction"] = enableAICorrection.value;

              fs.writeFileSync(configPath, JSON.stringify(config, null, 2));
            } catch (e) {
              // 忽略保存错误
            }

            // 导入 SRT 到项目面板
            const importResult = await evalES(
              'var f = new File("' + srtFilePath.replace(/\\/g, '\\\\') + '"); if (f.exists) { app.project.importFiles(["' + srtFilePath.replace(/\\/g, '\\\\') + '"], false, app.project.getInsertionBin(), false); "success"; } else { "file not found"; }',
              true
            );

            if (importResult === "success") {
              statusMessage.value = "字幕已导入项目面板，请手动添加到时间线";
            } else {
              statusMessage.value = "导入失败: " + importResult + "，但字幕文件已生成: " + srtFilePath;
            }
          } else {
            statusMessage.value = "转录完成";
          }

          resolve(null);
        });

        proc.on('error', (error) => {
          statusMessage.value = "转录失败: " + error.message;
          resolve(null);
        });
      });
    } else {
      statusMessage.value = "导出失败: " + result;
    }
  } catch (error: any) {
    alert("JS错误: " + (error?.message || String(error)));
    statusMessage.value = "导出失败: " + (error?.message || String(error));
  }
};

const handleSyncSubtitle = async () => {
  if (!srtPath.value) {
    statusMessage.value = "没有可同步的字幕文件";
    return;
  }

  statusMessage.value = "正在同步字幕...";

  try {
    const oldPath = srtPath.value;
    const oldExt = path.extname(oldPath);
    const oldDir = path.dirname(oldPath);
    const oldNameWithoutExt = path.basename(oldPath, oldExt);

    // 检查文件名中是否有数字结尾
    const match = oldNameWithoutExt.match(/_(\d+)$/);
    let newName;

    if (match) {
      // 数字加1
      const num = parseInt(match[1]) + 1;
      newName = oldNameWithoutExt.replace(/_(\d+)$/, "_" + num) + oldExt;
    } else {
      // 添加 _1
      newName = oldNameWithoutExt + "_1" + oldExt;
    }

    const newPath = path.join(oldDir, newName);

    // 复制文件
    fs.copyFileSync(oldPath, newPath);

    // 更新 srtPath 显示
    srtPath.value = newPath;

    // 保存新路径到配置
    const userDocsPath = getUserDocsPath();
    const configDir = userDocsPath + "/haoone/plugin_data/config";
    const configPath = configDir + "/pr-plugin.json";

    // 获取项目名和时间线名
    const nameResult = await evalES(
      'var p = app.project; var s = app.project.activeSequence; (p ? p.name : "") + "|" + (s ? s.name : "")',
      true
    );
    const parts = nameResult.split("|");
    const projectName = (parts[0] || "").replace(/\.prproj$/, "").replace(/[<>:"/|?*]/g, "_");
    const seqName = (parts[1] || "").replace(/[<>:"/|?*]/g, "_");

    // 更新配置
    try {
      let config: Record<string, any> = {};
      if (fs.existsSync(configPath)) {
        try {
          config = JSON.parse(fs.readFileSync(configPath, "utf8"));
        } catch (e) {}
      }
      const key = projectName + "_" + seqName;
      config[key] = newPath;
      fs.writeFileSync(configPath, JSON.stringify(config, null, 2));
    } catch (e) {}

    // 导入到 PR 项目面板
    const importResult = await evalES(
      'var f = new File("' + newPath.replace(/\\/g, '\\\\') + '"); if (f.exists) { app.project.importFiles(["' + newPath.replace(/\\/g, '\\\\') + '"], false, app.project.getInsertionBin(), false); "success"; } else { "file not found"; }',
      true
    );

    if (importResult === "success") {
      statusMessage.value = "新的 srt 文件已经导入项目面板，文件名: " + newName;
    } else {
      statusMessage.value = "同步完成但导入失败: " + importResult;
    }
  } catch (error: any) {
    statusMessage.value = "同步失败: " + (error?.message || String(error));
  }
};

onMounted(async () => {
  if (window.cep) {
    subscribeBackgroundColor((c: string) => (backgroundColor.value = c));

    // 读取配置文件
    const userDocsPath = getUserDocsPath();
    const configPath = userDocsPath + "/haoone/plugin_data/config/pr-plugin.json";

    try {
      if (fs.existsSync(configPath)) {
        const content = fs.readFileSync(configPath, "utf8");
        const config = JSON.parse(content);

        // 获取项目名和时间线名
        const nameResult = await evalES(
          'var p = app.project; var s = app.project.activeSequence; (p ? p.name : "") + "|" + (s ? s.name : "")',
          true
        );

        const parts = nameResult.split("|");
        const projectName = (parts[0] || "").replace(/\.prproj$/, "").replace(/[<>:"/|?*]/g, "_");
        const seqName = (parts[1] || "").replace(/[<>:"/|?*]/g, "_");
        const key = projectName + "_" + seqName;

        // 如果有对应配置则读取
        if (config[key]) {
          srtPath.value = config[key];
        }

        // 读取全局配置
        if (config["enable-online-transcript"] !== undefined) {
          enableOnlineTranscript.value = config["enable-online-transcript"];
        }
        if (config["max-subtitle-length"] !== undefined) {
          maxLineLength.value = String(config["max-subtitle-length"]);
        }
        if (config["enable-ai-correction"] !== undefined) {
          enableAICorrection.value = config["enable-ai-correction"];
        }
      }
    } catch (e) {
      // 忽略错误
    }
  }
});
</script>

<template>
  <div class="app" :style="{ backgroundColor: backgroundColor }">
    <div class="haoone-container">
      <!-- 标题栏 -->
      <div class="haoone-header">
        <span class="plugin-name">{{ PLUGIN_NAME }} by 浩叔 haoai.pro</span>
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
      </div>
      <div class="form-row">
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
          同步haoone软件的字幕修改
        </button>
      </div>

      <!-- SRT路径显示 -->
      <div class="form-row srt-path-row">
        <label class="form-label">字幕文件路径</label>
        <div class="srt-path-display">{{ srtPath || '' }}</div>
        <button
          class="btn btn-small"
          @click="handleOpenSrtDirectory"
          :disabled="!srtPath"
        >
          打开目录
        </button>
      </div>
    </div>
  </div>
</template>

<style lang="scss">
@use "../variables.scss" as *;
</style>