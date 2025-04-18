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
    const projectType = ref<string>('PC');
    const isDropdownOpen = ref<boolean>(false);
    const generatedCode = ref<string>(
      '// AI 提示词将在这里展示\n// 上传界面截图，快速获取专业提示词'
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
        message.error('只能上传图片文件！');
      }
      const isLt2M = file.size / 1024 / 1024 < 2;
      if (!isLt2M) {
        message.error('图片必须小于2MB！');
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
        message.error('只能上传图片文件！');
        return false;
      }
      const isValidSize = file.size / 1024 / 1024 < 2;
      if (!isValidSize) {
        message.error('图片必须小于2MB！');
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
        const url = `/api/upload-image`;

        // 构建 FormData
        const formData = new FormData();
        formData.append('image', selectedFile.value);
        formData.append('platform', projectType.value);

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
            message.success('提示词已复制到剪贴板');
          })
          .catch((err) => {
            console.error('Failed to copy: ', err);
            message.error('复制提示词失败');
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
        <img src="../assets/logo.svg" alt="Logo" />
        <div class="logo-text">
          <span class="company-name">君南圣达</span>
          <div class="logo-divider"></div>
          <span class="product-name">提示词工坊</span>
        </div>
      </div>
      <div class="nav-right">
        <!-- <button class="dashboard-btn">仪表盘</button> -->
      </div>
    </div>

    <div class="main-section">
      <div class="content-left">
        <div class="headline">
          <h1>AI编程助手的智能提示词生成器</h1>
          <p class="subheading">
            专为开发者打造，只需上传界面截图，即可生成专业的AI编程提示词
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
                  alt="预览图"
                />
              </div>
              <div class="file-info">
                <p class="file-name">{{ selectedFile?.name }}</p>
                <p class="file-size">
                  {{ formatFileSize(selectedFile?.size) }}
                </p>
              </div>
            </div>
            <p v-if="!hasFile" class="drag-text">拖拽图片到这里</p>
            <p v-if="!hasFile" class="click-text">或点击选择文件</p>
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
            更换文件
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
                  d="M4 19.5C4 18.837 4.26339 18.2011 4.73223 17.7322C5.20107 17.2634 5.83696 17 6.5 17H20"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
                <path
                  d="M6.5 2H20V22H6.5C5.83696 22 5.20107 21.7366 4.73223 21.2678C4.26339 20.7989 4 20.163 4 19.5V4.5C4 3.83696 4.26339 3.20107 4.73223 2.73223C5.20107 2.26339 5.83696 2 6.5 2Z"
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
                    v-for="type in ['PC', 'APP']"
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
              <span>生成的提示词</span>
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
                复制
              </button>
            </div>
          </div>
          <div class="code-area">
            <div v-html="renderedMarkdown"></div>
          </div>
          <div class="editor-footer">
            <div class="file-info"></div>
            <div class="line-info">提示词工坊</div>
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
  background-image: url('../assets/placeholder.png');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
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
  padding: 16px max(24px, 5%);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  background: linear-gradient(
    to right,
    rgba(0, 0, 0, 0.4) 0%,
    rgba(0, 0, 0, 0.2) 100%
  );
  backdrop-filter: blur(10px);
  position: relative;
  z-index: 10;
  flex: 0 0 auto;
}

.logo {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 600;
  position: relative;
}

.logo img {
  width: 80px;
  height: auto;
  margin-right: 4px;
  filter: brightness(0) invert(1);
}

.logo-text {
  display: flex;
  align-items: center;
}

.company-name {
  font-size: 26px;
  font-weight: 600;
  color: #ffffff;
  letter-spacing: 0.5px;
  position: relative;
  padding: 4px 12px 4px 0;
  transition: all 0.3s ease;
}

.logo-divider {
  width: 2px;
  height: 24px;
  background: linear-gradient(
    to bottom,
    rgba(90, 156, 248, 0.2) 0%,
    rgba(97, 61, 235, 0.2) 100%
  );
  margin: 0 16px;
  border-radius: 1px;
}

.product-name {
  font-size: 18px;
  color: rgba(255, 255, 255, 0.85);
  position: relative;
  transition: color 0.3s ease;
}

.product-name:hover {
  color: #ffffff;
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
  flex: 1 0 auto;
  display: flex;
  padding: clamp(40px, 5vh, 80px) clamp(20px, 5vw, 80px);
  gap: clamp(30px, 4vw, 60px);
  justify-content: center;
  align-items: stretch;
  max-width: 1800px;
  margin: 0 auto;
  width: 100%;
  min-height: calc(100vh - 73px);
}

.content-left {
  flex: 1 1 40%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  min-width: 300px;
  max-width: 750px;
  padding: 0 clamp(20px, 3vw, 40px);
}

.headline {
  display: flex;
  flex-direction: column;
  gap: clamp(20px, 3vh, 35px);
}

.headline h1 {
  font-size: clamp(32px, 5vw, 56px);
  line-height: 1.2;
  font-weight: 700;
  background: linear-gradient(45deg, #ffffff 0%, #e0e0e0 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  letter-spacing: -0.5px;
}

.subheading {
  font-size: clamp(16px, 2vw, 22px);
  line-height: 1.5;
  color: rgba(255, 255, 255, 0.8);
  letter-spacing: 0.2px;
}

.content-right {
  flex: 1 1 60%;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: clamp(40px, 5vw, 80px);
  min-width: 320px;
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
  flex: 0 1 480px;
  min-width: 300px;
  background: linear-gradient(
    180deg,
    rgba(30, 30, 40, 0.4) 0%,
    rgba(20, 20, 28, 0.4) 100%
  );
  border: 1px solid rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 16px;
  padding: clamp(20px, 3vw, 30px);
  z-index: 1;
  box-shadow: 0 14px 35px rgba(0, 0, 0, 0.2);
  display: flex;
  flex-direction: column;
  gap: 20px;
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
  height: 260px;
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
  flex: 0 1 580px;
  min-width: 300px;
  height: clamp(500px, 70vh, 700px);
  background-color: rgba(22, 22, 30, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 15px;
  display: flex;
  flex-direction: column;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.2),
              inset 0 0 1px rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(20px);
  transition: all 0.3s ease;
}

.code-editor-container:hover {
  border-color: #5a9cf8;
  background: rgba(90, 156, 248, 0.05);
  transform: translateY(-1px);
}

.editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 22px;
  background-color: rgba(35, 35, 45, 0.4);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 15px 15px 0 0;
}

.editor-title {
  display: flex;
  align-items: center;
  gap: 11px;
  font-weight: 500;
  font-size: 20px;
  
  svg {
    opacity: 0.8;
    transition: opacity 0.2s ease;
  }
  
  &:hover svg {
    opacity: 1;
  }
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
  background-color: rgba(20, 20, 28, 0.3);
  position: relative;
  border-radius: 0 0 15px 15px;
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
  border-radius: 0 0 15px 15px;
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

@media (max-width: 1024px) {
  .main-section {
    flex-direction: column;
    align-items: center;
  }

  .content-left {
    text-align: center;
    align-items: center;
    padding: 0;
  }

  .content-right {
    flex-direction: column;
    width: 100%;
    max-width: 800px;
  }

  .card, .code-editor-container {
    width: 100%;
  }
}

@media (max-width: 480px) {
  .navbar {
    padding: 12px 16px;
  }

  .logo-text {
    font-size: 16px;
  }

  .company-name {
    font-size: 18px;
  }

  .product-name {
    font-size: 16px;
  }
}
</style>
