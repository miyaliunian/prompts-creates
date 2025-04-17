<!-- 
Create a file named vite.config.ts in your project root (or edit if it exists)
with the following content:

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3002',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  }
})
-->

<script lang="ts">
import {
  defineComponent,
  ref,
  reactive,
  computed,
  onMounted,
  onUnmounted,
} from 'vue';
import { message } from 'ant-design-vue';
import type { UploadProps, UploadFile } from 'ant-design-vue';
import { marked } from 'marked';

export default defineComponent({
  name: 'ImageUploader',
  setup() {
    const fileList = ref<UploadFile[]>([]);
    const loading = ref<boolean>(false);
    const imageResult = ref<string>('');
    const previewVisible = ref<boolean>(false);
    const previewImage = ref<string>('');
    const thumbnailUrl = ref<string>('');
    const selectedAnalysisType = ref<string>('general');
    const fileInput = ref<HTMLInputElement | null>(null);
    const hasFile = ref<boolean>(false);
    const selectedFile = ref<File | null>(null);
    const filePath = ref<string>('');
    const projectType = ref<string>('PCAdmin');
    const isDropdownOpen = ref<boolean>(false);
    const generatedCode = ref<string>(
      '// Your generated prompt will appear here\n// Upload an image to begin'
    );
    const codeEditorTheme = ref<string>('vs-dark');
    const apiToken = ref<string>(
      'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo2LCJ1c2VybmFtZSI6ImR1Y2FmZWNhdDUiLCJpYXQiOjE2NTk2MjU3MTYsImV4cCI6MTY2MDIzMDUxNn0.3iVVEaTK03XYdYZUX6E6hBXqdLNCv0M7Irf1yHLmWQs'
    );
    const lineCount = ref<number>(2);
    const logs = ref<string[]>([]);
    const displayLogs = ref<string[]>([]);
    const isGenerating = ref<boolean>(false);
    const dropdownLeft = ref(0);
    const dropdownTop = ref(0);
    const dropdownWidth = ref(0);

    const state = reactive({
      analyzeButtonDisabled: true,
    });

    // 添加 marked 配置
    marked.setOptions({
      gfm: true,
      breaks: true,
    });

    // 添加计算属性用于渲染 markdown
    const renderedMarkdown = computed(() => {
      return marked(generatedCode.value);
    });

    const handleChange: UploadProps['onChange'] = (info) => {
      fileList.value = info.fileList.slice(-1); // Only keep the latest file

      // Update button state based on file list
      state.analyzeButtonDisabled = fileList.value.length === 0;
    };

    const beforeUpload = (file: any) => {
      const isImage = file.type.startsWith('image/');
      if (!isImage) {
        message.error('You can only upload image files!');
      }
      const isLt2M = file.size / 1024 / 1024 < 2;
      if (!isLt2M) {
        message.error('Image must be smaller than 2MB!');
      }
      return isImage && isLt2M;
    };

    const customRequest = ({ file, onSuccess }: any) => {
      // This prevents actual upload and just adds to fileList
      setTimeout(() => {
        onSuccess('ok');
      }, 0);
    };

    const handlePreview = async (file: UploadFile) => {
      if (!file.url && !file.preview) {
        file.preview = await getBase64(file.originFileObj!);
      }
      previewImage.value = file.url || file.preview || '';
      previewVisible.value = true;
    };

    const getBase64 = (file: File): Promise<string> => {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => resolve(reader.result as string);
        reader.onerror = (error) => reject(error);
      });
    };

    const triggerFileInput = () => {
      if (fileInput.value) {
        fileInput.value.click();
      }
    };

    const handleFileChange = (event: Event) => {
      const target = event.target as HTMLInputElement;
      if (target && target.files && target.files.length > 0) {
        const file = target.files[0];
        if (isValidFile(file)) {
          selectedFile.value = file;
          hasFile.value = true;
          // Generate thumbnail
          generateThumbnail(file);
          // Get file path
          filePath.value = URL.createObjectURL(file);
        }
      }
    };

    const handleFileDrop = (event: DragEvent) => {
      if (event.dataTransfer && event.dataTransfer.files.length > 0) {
        const file = event.dataTransfer.files[0];
        if (isValidFile(file)) {
          selectedFile.value = file;
          hasFile.value = true;
          // Generate thumbnail
          generateThumbnail(file);
          // Get file path
          filePath.value = URL.createObjectURL(file);
        }
      }
    };

    const generateThumbnail = (file: File) => {
      if (!file.type.startsWith('image/')) return;

      const reader = new FileReader();
      reader.onload = (e) => {
        if (e.target && e.target.result) {
          thumbnailUrl.value = e.target.result as string;
        }
      };
      reader.readAsDataURL(file);
    };

    const isValidFile = (file: File): boolean => {
      const isImage = file.type.startsWith('image/');
      if (!isImage) {
        message.error('You can only upload image files!');
        return false;
      }
      const isValidSize = file.size / 1024 / 1024 < 2;
      if (!isValidSize) {
        message.error('Image must be smaller than 2MB!');
        return false;
      }
      return true;
    };

    const toggleDropdown = () => {
      isDropdownOpen.value = !isDropdownOpen.value;
      if (isDropdownOpen.value) {
        updateDropdownPosition();
        // 添加点击外部关闭功能
        setTimeout(() => {
          window.addEventListener('click', handleClickOutside);
        });
      } else {
        window.removeEventListener('click', handleClickOutside);
      }
    };

    const selectProjectType = (type: string) => {
      projectType.value = type;
      isDropdownOpen.value = false;
    };

    const updateDropdownPosition = () => {
      const trigger = document.querySelector('.select-dropdown') as HTMLElement;
      if (trigger) {
        const rect = trigger.getBoundingClientRect();
        dropdownLeft.value = rect.left + window.scrollX;
        dropdownTop.value = rect.bottom + window.scrollY + 4;
        dropdownWidth.value = rect.width;
      }
    };

    const handleClickOutside = (e: MouseEvent) => {
      const trigger = document.querySelector('.select-dropdown');
      const dropdown = document.querySelector('.dropdown-menu');
      if (
        !trigger?.contains(e.target as Node) &&
        !dropdown?.contains(e.target as Node)
      ) {
        isDropdownOpen.value = false;
      }
    };

    // 打印速度（毫秒）
    const PRINT_SPEED = 500;

    const printLog = async (log: string) => {
      displayLogs.value.push('');
      const index = displayLogs.value.length - 1;
      await new Promise((resolve) => setTimeout(resolve, PRINT_SPEED));
      displayLogs.value[index] = log;
      
      // 自动滚动到底部
      setTimeout(() => {
        const logsArea = document.querySelector('.logs-area');
        if (logsArea) {
          logsArea.scrollTop = logsArea.scrollHeight;
        }
      }, 0);
    };

    const analyzeImage = async () => {
      if (!selectedFile.value) return;

      loading.value = true;
      isGenerating.value = true;
      imageResult.value = '';
      generatedCode.value = '';
      logs.value = [];
      displayLogs.value = [];
      
      // 确保日志区域滚动到顶部
      const logsArea = document.querySelector('.logs-area');
      if (logsArea) {
        logsArea.scrollTop = 0;
      }

      try {
        // 构建 URL，包含项目类型参数
        const url = `/api/upload-image?projectType=${encodeURIComponent(
          projectType.value
        )}`;

        // 构建 FormData
        const formData = new FormData();
        formData.append('image', selectedFile.value);

        // 使用 POST 方法上传文件并建立流式连接
        const response = await fetch(url, {
          method: 'POST',
          body: formData,
          headers: {
            Authorization: `Bearer ${apiToken.value}`,
          },
        });

        if (!response.ok) {
          throw new Error('上传失败');
        }

        // 获取响应的 reader
        const reader = response.body?.getReader();
        const decoder = new TextDecoder();

        if (!reader) {
          throw new Error('无法读取响应流');
        }

        let buffer = '';

        // 读取流数据
        while (true) {
          const { done, value } = await reader.read();

          if (done) {
            break;
          }

          try {
            // 将新的数据添加到缓冲区
            buffer += decoder.decode(value, { stream: true });

            // 按行分割并处理完整的行
            const lines = buffer.split('\n');
            // 保留最后一个可能不完整的行
            buffer = lines.pop() || '';

            for (const line of lines) {
              const trimmedLine = line.trim();
              if (!trimmedLine) continue;

              // 解析事件和数据
              const eventMatch = trimmedLine.match(/^event: (.+)$/);
              const dataMatch = trimmedLine.match(/^data: (.+)$/);

              if (eventMatch && eventMatch[1] === 'log') {
                // 等待下一行的 data
                continue;
              }

              if (dataMatch) {
                const data = JSON.parse(dataMatch[1]);

                // 检查前一行是否是 event: log
                const prevLine = lines[lines.indexOf(trimmedLine) - 1];
                if (prevLine?.includes('event: log')) {
                  // 添加日志消息
                  logs.value.push(data.message);
                  await printLog(data.message);
                } else if (prevLine?.includes('event: complete')) {
                  // 设置最终生成的代码
                  generatedCode.value = data.promptText;
                  message.success('提示词生成完成');
                }
              }
            }
          } catch (error) {
            console.error('Error parsing chunk:', error);
          }
        }
      } catch (error) {
        console.error('Error analyzing image:', error);
        message.error('图片分析失败');
      } finally {
        loading.value = false;
        isGenerating.value = false;
      }
    };

    const copyToClipboard = () => {
      if (generatedCode.value) {
        navigator.clipboard
          .writeText(generatedCode.value)
          .then(() => {
            message.success('Prompt copied to clipboard');
          })
          .catch((err) => {
            console.error('Failed to copy: ', err);
            message.error('Failed to copy prompt');
          });
      }
    };

    const formatFileSize = (size?: number): string => {
      if (!size) return '';

      if (size < 1024) {
        return size + ' B';
      } else if (size < 1024 * 1024) {
        return (size / 1024).toFixed(1) + ' KB';
      } else {
        return (size / (1024 * 1024)).toFixed(1) + ' MB';
      }
    };

    onMounted(() => {
      window.addEventListener('scroll', updateDropdownPosition, true);
      window.addEventListener('resize', updateDropdownPosition);
    });

    onUnmounted(() => {
      window.removeEventListener('scroll', updateDropdownPosition, true);
      window.removeEventListener('resize', updateDropdownPosition);
      window.removeEventListener('click', handleClickOutside);
    });

    return {
      fileList,
      loading,
      imageResult,
      previewVisible,
      previewImage,
      thumbnailUrl,
      selectedAnalysisType,
      state,
      handleChange,
      beforeUpload,
      customRequest,
      handlePreview,
      analyzeImage,
      fileInput,
      hasFile,
      selectedFile,
      filePath,
      triggerFileInput,
      handleFileChange,
      handleFileDrop,
      projectType,
      isDropdownOpen,
      toggleDropdown,
      selectProjectType,
      generatedCode,
      copyToClipboard,
      formatFileSize,
      apiToken,
      lineCount,
      renderedMarkdown,
      logs,
      displayLogs,
      isGenerating,
      dropdownLeft,
      dropdownTop,
      dropdownWidth,
    };
  },
});
</script>

<template>
  <div class="app-container">
    <div class="navbar">
      <div class="logo">
        <svg
          width="24"
          height="24"
          viewBox="0 0 24 24"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            d="M12 2L2 7L12 12L22 7L12 2Z"
            stroke="#5A9CF8"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
          <path
            d="M2 17L12 22L22 17"
            stroke="#5A9CF8"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
          <path
            d="M2 12L12 17L22 12"
            stroke="#5A9CF8"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
        <span>PromptCraft</span>
      </div>
      <div class="nav-right">
        <button class="dashboard-btn">Dashboard</button>
      </div>
    </div>

    <div class="main-section">
      <div class="content-left">
        <div class="headline">
          <h1>Create powerful prompts for AI coding tools</h1>
          <p class="subheading">
            Built for the next generation of AI coders. Upload images of full
            applications.
          </p>
        </div>
      </div>

      <div class="content-right">
        <div class="card">
          <div
            class="upload-area"
            @click="triggerFileInput"
            @dragover.prevent
            @drop.prevent="handleFileDrop"
          >
            <div v-if="!hasFile" class="upload-icon">
              <svg
                width="24"
                height="24"
                viewBox="0 0 24 24"
                fill="none"
                xmlns="http://www.w3.org/2000/svg"
              >
                <path
                  d="M12 16V8M12 8L9 11M12 8L15 11"
                  stroke="white"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
                <path
                  d="M12 22C17.5228 22 22 17.5228 22 12C22 6.47715 17.5228 2 12 2C6.47715 2 2 6.47715 2 12C2 17.5228 6.47715 22 12 22Z"
                  stroke="white"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
              </svg>
            </div>
            <div v-if="hasFile" class="file-selected-container">
              <div class="thumbnail-preview">
                <img
                  v-if="thumbnailUrl"
                  :src="thumbnailUrl"
                  class="thumbnail-image"
                  alt="Thumbnail"
                />
              </div>
              <div class="file-info">
                <p class="file-name">{{ selectedFile?.name }}</p>
                <p class="file-size">
                  {{ formatFileSize(selectedFile?.size) }}
                </p>
              </div>
            </div>
            <p v-if="!hasFile" class="drag-text">Drag & drop your image here</p>
            <p v-if="!hasFile" class="click-text">or click to select a file</p>
            <input
              type="file"
              ref="fileInput"
              class="hidden-input"
              accept="image/*"
              @change="handleFileChange"
            />
          </div>

          <button
            v-if="hasFile"
            class="change-file-btn"
            @click="triggerFileInput"
          >
            Change file
          </button>

          <div v-if="isGenerating || displayLogs.length > 0" class="logs-area">
            <div
              v-for="(log, index) in displayLogs"
              :key="index"
              class="log-item"
              :class="{ 'new-log': index === displayLogs.length - 1 }"
            >
              {{ log }}
            </div>
          </div>

          <button
            class="generate-btn"
            :disabled="!hasFile || loading"
            @click="analyzeImage"
          >
            <span v-if="loading" class="loading-spinner">
              <svg
                width="24"
                height="24"
                viewBox="0 0 24 24"
                fill="none"
                xmlns="http://www.w3.org/2000/svg"
              >
                <path
                  opacity="0.2"
                  d="M12 2C6.48 2 2 6.48 2 12C2 17.52 6.48 22 12 22C17.52 22 22 17.52 22 12C22 6.48 17.52 2 12 2ZM12 20C7.59 20 4 16.41 4 12C4 7.59 7.59 4 12 4C16.41 4 20 7.59 20 12C20 16.41 16.41 20 12 20Z"
                  fill="white"
                />
                <path
                  d="M12 2V4C16.41 4 20 7.59 20 12H22C22 6.48 17.52 2 12 2Z"
                  fill="white"
                >
                  <animateTransform
                    attributeName="transform"
                    attributeType="XML"
                    type="rotate"
                    from="0 12 12"
                    to="360 12 12"
                    dur="1s"
                    repeatCount="indefinite"
                  />
                </path>
              </svg>
            </span>
            <span v-else class="btn-icon">
              <svg
                width="24"
                height="24"
                viewBox="0 0 24 24"
                fill="none"
                xmlns="http://www.w3.org/2000/svg"
              >
                <path
                  d="M12 2L2 7L12 12L22 7L12 2Z"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
                <path
                  d="M2 17L12 22L22 17"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
                <path
                  d="M2 12L12 17L22 12"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
              </svg>
            </span>
            <span>生成提示词</span>
          </button>

          <div class="project-type">
            <p>选择项目类型</p>
            <div class="select-container">
              <div
                class="select-dropdown"
                :class="{ active: isDropdownOpen }"
                @click="toggleDropdown"
              >
                <span>{{ projectType }}</span>
                <svg
                  width="16"
                  height="16"
                  viewBox="0 0 24 24"
                  fill="none"
                  xmlns="http://www.w3.org/2000/svg"
                >
                  <path
                    d="M6 9L12 15L18 9"
                    stroke="white"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                </svg>
              </div>

              <Teleport to="body">
                <div
                  v-if="isDropdownOpen"
                  class="dropdown-menu"
                  :class="{ active: isDropdownOpen }"
                  :style="{
                    left: dropdownLeft + 'px',
                    top: dropdownTop + 'px',
                    width: dropdownWidth + 'px',
                  }"
                >
                  <div
                    v-for="type in ['PCAdmin', 'WebApp']"
                    :key="type"
                    class="dropdown-item"
                    :class="{ selected: projectType === type }"
                    @click="selectProjectType(type)"
                  >
                    {{ type }}
                  </div>
                </div>
              </Teleport>
            </div>
          </div>
        </div>

        <div class="code-editor-container">
          <div class="editor-header">
            <div class="editor-title">
              <svg
                width="20"
                height="20"
                viewBox="0 0 24 24"
                fill="none"
                xmlns="http://www.w3.org/2000/svg"
              >
                <path
                  d="M16 18L22 12L16 6"
                  stroke="#5A9CF8"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
                <path
                  d="M8 6L2 12L8 18"
                  stroke="#5A9CF8"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
              </svg>
              <span>Generated Prompt</span>
            </div>
            <div class="editor-actions">
              <button class="action-button" @click="copyToClipboard">
                <svg
                  width="16"
                  height="16"
                  viewBox="0 0 24 24"
                  fill="none"
                  xmlns="http://www.w3.org/2000/svg"
                >
                  <path
                    d="M20 9H11C9.89543 9 9 9.89543 9 11V20C9 21.1046 9.89543 22 11 22H20C21.1046 22 22 21.1046 22 20V11C22 9.89543 21.1046 9 20 9Z"
                    stroke="white"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                  <path
                    d="M5 15H4C3.46957 15 2.96086 14.7893 2.58579 14.4142C2.21071 14.0391 2 13.5304 2 13V4C2 3.46957 2.21071 2.96086 2.58579 2.58579C2.96086 2.21071 3.46957 2 4 2H13C13.5304 2 14.0391 2.21071 14.4142 2.58579C14.7893 2.96086 15 3.46957 15 4V5"
                    stroke="white"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                </svg>
                Copy
              </button>
            </div>
          </div>
          <div class="code-area">
            <div v-html="renderedMarkdown"></div>
          </div>
          <div class="editor-footer">
            <div class="file-info">index.js</div>
            <div class="line-info">{{ lineCount }} lines</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  width: 100%;
  min-height: 100vh;
  background-color: #000000;
  color: white;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  display: flex;
  flex-direction: column;
}

.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 24px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.logo {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 600;
}

.nav-right {
  display: flex;
  align-items: center;
}

.dashboard-btn {
  padding: 5px 12px;
  background-color: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  color: white;
  font-size: 14px;
  cursor: pointer;
}

.main-section {
  flex: 1;
  display: flex;
  padding: 80px 140px 80px 30px;
  gap: 60px;
  justify-content: center;
  align-items: center;
  max-width: 1800px;
  margin: 0 auto;
  height: calc(100vh - 73px);
}

.content-left {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  max-width: 650px;
  padding-right: 20px;
  padding-left: 0;
  height: 700px;
  margin-left: 0;
}

.headline {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  width: 100%;
  text-align: left;
}

.headline h1 {
  font-size: 52px;
  line-height: 1.1;
  margin-bottom: 30px;
  font-weight: 700;
  width: 100%;
  word-wrap: break-word;
  overflow-wrap: break-word;
  hyphens: auto;
  max-width: 600px;
  display: inline-block;
}

.subheading {
  font-size: 20px;
  line-height: 1.4;
  color: rgba(255, 255, 255, 0.8);
  margin-bottom: 0;
  width: 100%;
  max-width: 580px;
}

.content-right {
  flex: 3;
  display: flex;
  justify-content: flex-start;
  align-items: center;
  position: relative;
  padding-left: 0;
  margin-left: 0;
  gap: 100px;
  height: 700px;
}

.content-right::before {
  content: '';
  position: absolute;
  width: 560px;
  height: 560px;
  background: radial-gradient(
    circle,
    rgba(97, 61, 235, 0.3) 0%,
    rgba(0, 0, 0, 0) 70%
  );
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 0;
  pointer-events: none;
}

.card {
  width: 532px;
  background: linear-gradient(
    180deg,
    rgba(30, 30, 40, 0.4) 0%,
    rgba(20, 20, 28, 0.4) 100%
  );
  border: 1px solid rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 16px;
  padding: 34px;
  z-index: 1;
  box-shadow: 0 14px 35px rgba(0, 0, 0, 0.2);
  height: fit-content;
  max-height: 700px;
  align-self: center;
  display: flex;
  flex-direction: column;
  gap: 20px;
  position: relative;
  overflow: hidden;
}

.card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 1px;
  background: linear-gradient(
    90deg,
    rgba(255, 255, 255, 0) 0%,
    rgba(255, 255, 255, 0.1) 50%,
    rgba(255, 255, 255, 0) 100%
  );
}

.upload-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 280px;
  border: 1px dashed rgba(255, 255, 255, 0.2);
  border-radius: 12px;
  background: rgba(0, 0, 0, 0.2);
  cursor: pointer;
  transition: all 0.3s ease;
  margin-bottom: 17px;
  position: relative;
  overflow: hidden;
}

.upload-area:hover {
  border-color: #5a9cf8;
  background: rgba(90, 156, 248, 0.05);
  transform: translateY(-1px);
}

.upload-icon {
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 50%;
  width: 67px;
  height: 67px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 22px;
}

.file-selected-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  padding: 20px;
}

.thumbnail-preview {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  max-height: 70%;
  overflow: hidden;
}

.thumbnail-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border-radius: 4px;
}

.file-info {
  margin-top: 10px;
  text-align: center;
}

.file-name {
  font-size: 18px;
  margin-bottom: 6px;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-size {
  font-size: 16px;
  color: rgba(255, 255, 255, 0.6);
  margin: 0;
}

.change-file-btn {
  width: 100%;
  padding: 14px;
  background-color: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  color: white;
  font-size: 18px;
  cursor: pointer;
  transition: background-color 0.3s;
  margin-bottom: 25px;
}

.change-file-btn:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.hidden-input {
  display: none;
}

.drag-text {
  font-size: 22px;
  margin-bottom: 6px;
}

.click-text {
  font-size: 20px;
  color: rgba(255, 255, 255, 0.6);
}

.generate-btn {
  width: 100%;
  padding: 17px;
  background: linear-gradient(45deg, #613deb 0%, #5a9cf8 100%);
  border: none;
  border-radius: 12px;
  color: white;
  font-size: 20px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  letter-spacing: 0.5px;
}

.generate-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(
    45deg,
    rgba(255, 255, 255, 0.1) 0%,
    rgba(255, 255, 255, 0) 100%
  );
  opacity: 0;
  transition: opacity 0.3s ease;
}

.generate-btn::after {
  content: '';
  position: absolute;
  width: 100%;
  height: 100%;
  background: linear-gradient(
    90deg,
    transparent 0%,
    rgba(255, 255, 255, 0.2) 50%,
    transparent 100%
  );
  top: 0;
  left: -100%;
  transition: left 0.5s ease;
}

.generate-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(97, 61, 235, 0.25);
}

.generate-btn:hover:not(:disabled)::before {
  opacity: 1;
}

.generate-btn:hover:not(:disabled)::after {
  left: 100%;
}

.generate-btn:active:not(:disabled) {
  transform: translateY(0);
  box-shadow: 0 4px 12px rgba(97, 61, 235, 0.2);
}

.generate-btn:disabled {
  background: linear-gradient(
    45deg,
    rgba(97, 61, 235, 0.4) 0%,
    rgba(90, 156, 248, 0.4) 100%
  );
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.loading-spinner {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* 按钮内部图标样式 */
.btn-icon {
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.3s ease;
}

.generate-btn:hover:not(:disabled) .btn-icon {
  transform: scale(1.1);
}

.project-type {
  margin-top: 8px;
}

.project-type p {
  font-size: 16px;
  margin-bottom: 12px;
  color: rgba(255, 255, 255, 0.8);
  font-weight: 500;
}

.select-container {
  position: relative;
  width: 100%;
}

.select-dropdown {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 16px;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  user-select: none;
}

.select-dropdown:hover {
  background: rgba(255, 255, 255, 0.08);
  border-color: rgba(255, 255, 255, 0.2);
}

.select-dropdown svg {
  transition: transform 0.3s ease;
}

.select-dropdown.active {
  border-color: #5a9cf8;
  background: rgba(90, 156, 248, 0.1);
}

.select-dropdown.active svg {
  transform: rotate(180deg);
}

.dropdown-menu {
  position: fixed;
  background: rgba(30, 30, 40, 0.98);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  padding: 6px;
  backdrop-filter: blur(10px);
  transform-origin: top;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  opacity: 0;
  transform: translateY(-10px) scale(0.95);
  pointer-events: none;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.3);
  z-index: 1000;
}

.dropdown-menu.active {
  opacity: 1;
  transform: translateY(0) scale(1);
  pointer-events: auto;
}

.dropdown-item {
  padding: 12px 14px;
  cursor: pointer;
  transition: all 0.2s ease;
  border-radius: 6px;
  color: rgba(255, 255, 255, 0.8);
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 15px;
}

.dropdown-item:hover {
  background: rgba(90, 156, 248, 0.1);
  color: #fff;
}

.dropdown-item.selected {
  color: #5a9cf8;
  background: rgba(90, 156, 248, 0.1);
}

.dropdown-item.selected::before {
  content: '✓';
  font-weight: bold;
  margin-right: 4px;
}

.code-editor-container {
  width: 630px;
  height: 700px;
  background-color: rgba(22, 22, 30, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 16px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  z-index: 1;
  box-shadow: 0 14px 35px rgba(0, 0, 0, 0.2);
  align-self: center;
}

.editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 22px;
  background-color: rgba(35, 35, 45, 0.8);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.editor-title {
  display: flex;
  align-items: center;
  gap: 11px;
  font-weight: 500;
  font-size: 20px;
}

.editor-actions {
  display: flex;
  align-items: center;
  gap: 14px;
}

.action-button {
  display: flex;
  align-items: center;
  gap: 8px;
  background-color: rgba(97, 61, 235, 0.2);
  border: 1px solid rgba(97, 61, 235, 0.3);
  color: white;
  padding: 8px 17px;
  border-radius: 6px;
  font-size: 18px;
  cursor: pointer;
  transition: all 0.2s;
}

.action-button:hover {
  background-color: rgba(97, 61, 235, 0.3);
}

.code-area {
  flex: 1;
  overflow: auto;
  padding: 22px;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 18px;
  line-height: 1.5;
  color: #d4d4d4;
  background-color: rgba(20, 20, 28, 0.9);

  /* 自定义滚动条样式 */
  &::-webkit-scrollbar {
    width: 8px;
    height: 8px;
  }

  &::-webkit-scrollbar-track {
    background: rgba(255, 255, 255, 0.05);
    border-radius: 4px;
  }

  &::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.2);
    border-radius: 4px;
    transition: background 0.2s ease;

    &:hover {
      background: rgba(255, 255, 255, 0.3);
    }
  }

  /* Firefox 滚动条样式 */
  scrollbar-width: thin;
  scrollbar-color: rgba(255, 255, 255, 0.2) rgba(255, 255, 255, 0.05);
}

/* 全局滚动条样式 */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: rgba(0, 0, 0, 0.2);
  border-radius: 4px;
}

::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  transition: background 0.2s ease;

  &:hover {
    background: rgba(255, 255, 255, 0.3);
  }
}

/* Firefox 全局滚动条样式 */
* {
  scrollbar-width: thin;
  scrollbar-color: rgba(255, 255, 255, 0.2) rgba(0, 0, 0, 0.2);
}

.code-area :deep(pre) {
  margin: 0;
  white-space: pre-wrap;
}

.code-area :deep(code) {
  font-family: inherit;
  background-color: rgba(0, 0, 0, 0.2);
  padding: 2px 4px;
  border-radius: 3px;
}

.code-area :deep(p) {
  margin: 0 0 1em 0;
}

.code-area :deep(h1),
.code-area :deep(h2),
.code-area :deep(h3),
.code-area :deep(h4),
.code-area :deep(h5),
.code-area :deep(h6) {
  margin: 1em 0 0.5em 0;
  color: #fff;
}

.code-area :deep(ul),
.code-area :deep(ol) {
  margin: 0 0 1em 0;
  padding-left: 2em;
}

.code-area :deep(li) {
  margin: 0.5em 0;
}

.code-area :deep(blockquote) {
  margin: 0 0 1em 0;
  padding-left: 1em;
  border-left: 4px solid rgba(255, 255, 255, 0.2);
  color: rgba(255, 255, 255, 0.7);
}

.code-area :deep(a) {
  color: #5a9cf8;
  text-decoration: none;
}

.code-area :deep(a:hover) {
  text-decoration: underline;
}

.code-area :deep(table) {
  border-collapse: collapse;
  width: 100%;
  margin: 0 0 1em 0;
}

.code-area :deep(th),
.code-area :deep(td) {
  border: 1px solid rgba(255, 255, 255, 0.2);
  padding: 0.5em;
  text-align: left;
}

.code-area :deep(th) {
  background-color: rgba(255, 255, 255, 0.1);
}

.editor-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 11px 22px;
  background-color: rgba(35, 35, 45, 0.8);
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  font-size: 17px;
  color: #adbac7;
}

.logs-area {
  flex: 1;
  min-height: 150px;
  max-height: 200px;
  overflow-y: auto;
  background: rgba(0, 0, 0, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  padding: 16px;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 14px;
  line-height: 1.6;
  color: #a0a0a0;
  scroll-behavior: smooth;
  position: relative;
  z-index: 1;
}

.logs-area::before {
  content: '';
  position: absolute;
  top: 0;
  left: 12px;
  right: 12px;
  height: 30px;
  background: linear-gradient(
    180deg,
    rgba(0, 0, 0, 0.2) 0%,
    rgba(0, 0, 0, 0) 100%
  );
  pointer-events: none;
  z-index: 2;
  border-radius: 12px 12px 0 0;
}

.logs-area::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 12px;
  right: 12px;
  height: 30px;
  background: linear-gradient(
    0deg,
    rgba(0, 0, 0, 0.2) 0%,
    rgba(0, 0, 0, 0) 100%
  );
  pointer-events: none;
  z-index: 2;
  border-radius: 0 0 12px 12px;
}

.log-item {
  margin-bottom: 12px;
  padding: 8px 12px;
  background: rgba(255, 255, 255, 0.02);
  border-radius: 6px;
  display: flex;
  align-items: flex-start;
  gap: 8px;
  opacity: 0.8;
  transition: all 0.3s ease;
  position: relative;
  border: 1px solid rgba(255, 255, 255, 0.03);
}

.log-item::before {
  content: '>';
  color: #5a9cf8;
  font-weight: bold;
  opacity: 0.8;
  transition: all 0.3s ease;
}

.log-item:hover {
  background: rgba(255, 255, 255, 0.04);
  border-color: rgba(255, 255, 255, 0.05);
}

.log-item:hover::before {
  opacity: 1;
  transform: translateX(2px);
}

.new-log {
  opacity: 1;
  color: #fff;
  background: rgba(90, 156, 248, 0.05);
  border-color: rgba(90, 156, 248, 0.1);
  transform: translateX(4px);
}

.new-log::before {
  opacity: 1;
  color: #5a9cf8;
}

/* 自定义滚动条 */
.logs-area::-webkit-scrollbar {
  width: 6px;
}

.logs-area::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.02);
  border-radius: 3px;
}

.logs-area::-webkit-scrollbar-thumb {
  background: rgba(90, 156, 248, 0.2);
  border-radius: 3px;
  transition: all 0.3s ease;
}

.logs-area::-webkit-scrollbar-thumb:hover {
  background: rgba(90, 156, 248, 0.3);
}
</style>
