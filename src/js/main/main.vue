<script setup lang="ts">
import { onMounted, ref } from "vue";
import { csi, evalES, openLinkInBrowser, subscribeBackgroundColor } from "../lib/utils/bolt";
import { child_process } from "../lib/cep/node";
import "../index.scss";

const { spawn } = child_process;

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

    // 生成输出文件名
    const projectName = (await evalES('app.project.name || "project"', true))
      .replace(/\.prproj$/, "")  // 去掉 .prproj 扩展名
      .replace(/[<>:"/\\|?*]/g, "_");
    const seqName = (await evalES('app.project.activeSequence.name || "sequence"', true)).replace(/[<>:"/\\|?*]/g, "_");
    const outputFile = p1.homePath + "/haoone/plugin_data/" + projectName + "_" + seqName + ".wav";
    const exportPresetPath = p1.homePath + "/haoone/plugin_data/config/haoone.epr";

    const result = await evalES(
      'app.project.activeSequence.exportAsMediaDirect("' + outputFile + '", "' + exportPresetPath + '", 0)',
      true
    );

    if (result === "No Error" || result === 0) {
      statusMessage.value = "音频导出完成";

      // 获取 hooone 程序路径
      const step5a = await evalES(`
        JSON.stringify((function() {
          try {
            var isWindows = $.os.indexOf("Windows") !== -1;
            var DEBUG = true;

            var fileExists = function(path) {
              var f = new File(path);
              return f.exists;
            };

            // macOS 路径
            var macPaths = [
              "/Applications/haoone.app/Contents/MacOS/haoone",
              "/usr/local/bin/haoone",
              "/opt/haoone/haoone"
            ];

            for (var m = 0; m < macPaths.length; m++) {
              if (fileExists(macPaths[m])) {
                return { success: true, path: macPaths[m] };
              }
            }

            // 尝试用户本地 bin
            var homePath = Folder("~").fsName;
            if (homePath) {
              var localBinPath = homePath + "/.local/bin/haoone";
              if (fileExists(localBinPath)) {
                return { success: true, path: localBinPath };
              }
            }

            // Windows 路径检测
            if (isWindows) {
              // Windows 常见安装路径
              var windowsPaths = [
                "C:\\Program Files\\haoone\\haoone.exe",
                "C:\\Program Files (x86)\\haoone\\haoone.exe"
              ];

              // 尝试常见路径
              for (var i = 0; i < windowsPaths.length; i++) {
                if (fileExists(windowsPaths[i])) {
                  return { success: true, path: windowsPaths[i] };
                }
              }

              // 尝试从 LOCALAPPDATA 获取
              var localAppData = Folder("").fsName;
              if (localAppData) {
                var appDataPath = localAppData + "\\haoone\\haoone.exe";
                if (fileExists(appDataPath)) {
                  return { success: true, path: appDataPath };
                }
              }

              // 遍历所有盘符查找
              var drives = ["C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"];
              for (var d = 0; d < drives.length; d++) {
                var drivePaths = [
                  drives[d] + ":\\Program Files\\haoone\\haoone.exe",
                  drives[d] + ":\\Program Files (x86)\\haoone\\haoone.exe",
                  drives[d] + ":\\haoone\\haoone.exe"
                ];
                for (var p = 0; p < drivePaths.length; p++) {
                  if (fileExists(drivePaths[p])) {
                    return { success: true, path: drivePaths[p] };
                  }
                }
              }

              return { success: true, path: "haoone.exe" }; // 最后尝试 PATH
            }

            return { success: false, error: "haoone not found" };
          } catch(e) {
            return { success: false, error: e.toString() };
          }
        })())
      `, true);

      let p5a = JSON.parse(step5a);
      if (!p5a.success) {
        statusMessage.value = "失败: " + p5a.error;
        return;
      }

      const haoonePath = p5a.path;

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

        proc.on('close', (code) => {
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
            // 转录完成，现在导入字幕到时间线
            statusMessage.value = "正在导入字幕到项目面板...";

            (async () => {
              try {
                // 步骤6: 使用 importFiles 导入 SRT 到项目面板
                const step6 = await evalES(`
                  JSON.stringify((function() {
                    try {
                      var srtPath = "${srtFilePath.replace(/\\/g, '\\\\')}";
                      var activeSeq = app.project.activeSequence;
                      if (!activeSeq) return { success: false, error: "No active sequence" };

                      var srtFile = new File(srtPath);
                      if (!srtFile.exists) return { success: false, error: "SRT file not found" };

                      // 使用 importFiles 导入 SRT 到项目面板
                      var filesToImport = [srtPath];
                      app.project.importFiles(
                        filesToImport,
                        false,
                        app.project.getInsertionBin(),
                        false
                      );

                      return { success: true, message: "SRT imported to project" };
                    } catch(e) {
                      return { success: false, error: e.toString() };
                    }
                  })())
                `, true);

                let p6 = JSON.parse(step6);
                if (p6.success) {
                  statusMessage.value = "字幕已导入项目面板，请手动添加到时间线";
                } else {
                  statusMessage.value = "导入失败: " + p6.error + "，但字幕文件已生成: " + srtFilePath;
                }
                srtPath.value = srtFilePath;

                // 保存 srt 路径到 pr-plugin.json
                await evalES(`
                  (function() {
                    try {
                      // 获取用户目录
                      var homePath = "";
                      if ($.os.indexOf("Windows") !== -1) {
                        homePath = Folder.userData.fsName.replace(/\\\\Roaming$/, "");
                      } else {
                        homePath = Folder("~/Documents").fsName;
                      }
                      var configPath = homePath + "/haoone/plugin_data/config/pr-plugin.json";
                      var projectName = "${toUnicodeEscape(projectName)}";
                      var timelineName = "${toUnicodeEscape(timelineName)}";
                      var srtFilePathValue = "${toUnicodeEscape(srtFilePath.replace(/\\/g, '\\\\'))}";
                      var f = new File(configPath);
                      if (f.exists) {
                        f.open("r");
                        var content = f.read();
                        f.close();
                        var config = JSON.parse(content);
                        config[projectName + "_" + timelineName] = srtFilePathValue;
                        f.open("w");
                        f.write(JSON.stringify(config, null, 2));
                        f.close();
                      }
                    } catch(e) {
                      // 静默处理错误
                    }
                  })()
                `, true);
              } catch (e: any) {
                statusMessage.value = "导入失败: " + (e?.message || String(e)) + "，但字幕文件已生成: " + srtFilePath;
                srtPath.value = srtFilePath;
              }
            })();
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

const handleSyncSubtitle = () => {
  statusMessage.value = "正在同步字幕...";
};

onMounted(async () => {
  if (window.cep) {
    subscribeBackgroundColor((c: string) => (backgroundColor.value = c));
  }

  // 首次打开时创建配置文件
  const defaultConfig = {
    "enable-online-transcript": true,
    "max-subtitle-length": 25,
    "enable-ai-correction": false
  };

  // 使用 evalES 检查并创建配置文件，并读取 srt 路径
  const initResult = await evalES(`
    (async function() {
      try {
        // 获取用户目录
        var homePath = "";
        if ($.os.indexOf("Windows") !== -1) {
          homePath = Folder.userData.fsName.replace(/\\\\Roaming$/, "");
        } else {
          homePath = Folder("~/Documents").fsName;
        }

        var configDir = homePath + "/haoone/plugin_data/config";
        var configPath = configDir + "/pr-plugin.json";
        var defaultConfig = ${JSON.stringify(JSON.stringify(defaultConfig))};

        var f = new File(configPath);
        if (!f.exists) {
          // 创建目录
          var folder = new Folder(configDir);
          if (!folder.exists) {
            folder.create();
          }
          // 写入配置文件
          var writeFile = new File(configPath);
          writeFile.open("w");
          writeFile.write(defaultConfig);
          writeFile.close();
          return { success: true, srtPath: "" };
        } else {
          // 读取配置文件
          f.open("r");
          var content = f.read();
          f.close();
          var config = JSON.parse(content);

          // 获取当前项目名和时间线名
          var projectName = "";
          var timelineName = "";
          try {
            projectName = app.project.name.replace(/\\.prproj$/, "").replace(/[<>:"/\\\\|?*]/g, "_");
          } catch(e) {}
          try {
            timelineName = app.project.activeSequence.name.replace(/[<>:"/\\\\|?*]/g, "_");
          } catch(e) {}

          // 查找对应的 srt 路径
          var key = projectName + "_" + timelineName;
          var srtPath = config[key] || "";

          return { success: true, srtPath: srtPath, projectName: projectName, timelineName: timelineName };
        }
      } catch(e) {
        return { success: false, error: e.toString(), srtPath: "" };
      }
    })()
  `, true);

  try {
    const result = JSON.parse(initResult);
    if (result.success && result.srtPath) {
      srtPath.value = result.srtPath;
    }
  } catch (e) {
    // 静默处理解析错误
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
          同步字幕
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