<template>
  <div class="chat-container">
    <h1>AI 聊天助手</h1>
    
    <div class="chat-box">
      <div v-for="(message, index) in messages" :key="index" :class="['message', message.role]">
        <div class="message-content">
          {{ message.content }}
        </div>
      </div>
      <div v-if="isLoading" class="message assistant">
        <div class="message-content">
          {{ partialResponse }}<span class="cursor">|</span>
        </div>
      </div>
    </div>
    
    <div class="input-area">
      <input
        v-model="userInput"
        @keyup.enter="sendMessage"
        placeholder="输入你的问题..."
        :disabled="isLoading"
      />
      <button @click="sendMessage" :disabled="isLoading || !userInput.trim()">
        {{ isLoading ? '生成中...' : '发送' }}
      </button>
    </div>
  </div>
</template>

<script setup>
/* 完全保留您原有的script部分 */
import { ref } from 'vue';
import axios from 'axios';

const userInput = ref('');
const messages = ref([]);
const partialResponse = ref('');
const isLoading = ref(false);

const formatTime = () => {
  return new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
};

const rawStreamData = ref('');

const sendMessage = async () => {
  if (!userInput.value.trim() || isLoading.value) return;
  
  messages.value.push({
    role: 'user',
    content: userInput.value
  });
  
  const prompt = userInput.value;
  userInput.value = '';
  partialResponse.value = '';
  rawStreamData.value = '';
  isLoading.value = true;
  
  try {
    const response = await fetch('http://localhost:8000/api/chat', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        prompt: prompt,
        model: 'deepsex',
        stream: true
      })
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) {
        messages.value.push({
          role: 'assistant',
          content: partialResponse.value
        });
        break;
      }
      
      const chunk = decoder.decode(value);
      rawStreamData.value += chunk;
      
      const lines = rawStreamData.value.split('\n');
      rawStreamData.value = lines.pop() || '';
      
      for (const line of lines) {
        if (!line.trim()) continue;
        try {
          const data = JSON.parse(line);
          if (data.response) {
            partialResponse.value += data.response;
          }
        } catch (e) {
          console.error('解析JSON失败:', e, '原始数据:', line);
        }
      }
    }
  } catch (error) {
    console.error('请求出错:', error);
    messages.value.push({
      role: 'assistant',
      content: '错误: ' + error.message
    });
  } finally {
    isLoading.value = false;
    partialResponse.value = '';
  }
};
</script>

<style scoped>
.chat-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  color: #333;
  background-color: #f5f7fa;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.chat-container h1 {
  text-align: center;
  color: #2c3e50;
  margin-bottom: 30px;
  font-weight: 600;
  font-size: 24px;
  position: relative;
  padding-bottom: 15px;
}

.chat-container h1::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 60px;
  height: 3px;
  background-color: #3498db;
  border-radius: 3px;
}

.chat-box {
  height: 500px;
  overflow-y: auto;
  padding: 20px;
  background-color: white;
  border-radius: 10px;
  margin-bottom: 20px;
  box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.03);
  scroll-behavior: smooth;
}

.chat-box::-webkit-scrollbar {
  width: 6px;
}

.chat-box::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 10px;
}

.chat-box::-webkit-scrollbar-thumb {
  background: #ccc;
  border-radius: 10px;
}

.chat-box::-webkit-scrollbar-thumb:hover {
  background: #aaa;
}

.message {
  margin-bottom: 16px;
  max-width: 70%;
  animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.message.user {
  margin-left: auto;
}

.message.assistant {
  margin-right: auto;
}

.message-content {
  padding: 12px 18px;
  border-radius: 18px;
  line-height: 1.6;
  position: relative;
  word-wrap: break-word;
}

.user .message-content {
  background-color: #3498db;
  color: white;
  border-bottom-right-radius: 4px;
  box-shadow: 0 2px 8px rgba(52, 152, 219, 0.2);
}

.assistant .message-content {
  background-color: #f1f5f9;
  color: #333;
  border-bottom-left-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.input-area {
  display: flex;
  gap: 10px;
}

.input-area input {
  flex: 1;
  padding: 14px 18px;
  border: 1px solid #ddd;
  border-radius: 25px;
  font-size: 16px;
  transition: all 0.3s ease;
  outline: none;
}

.input-area input:focus {
  border-color: #3498db;
  box-shadow: 0 0 0 3px rgba(52, 152, 219, 0.1);
}

.input-area input:disabled {
  background-color: #f9f9f9;
  cursor: not-allowed;
  opacity: 0.7;
}

.input-area button {
  padding: 14px 24px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 25px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.input-area button:not(:disabled):hover {
  background-color: #2980b9;
  transform: translateY(-1px);
}

.input-area button:not(:disabled):active {
  transform: translateY(1px);
}

.input-area button:disabled {
  background-color: #bdc3c7;
  cursor: not-allowed;
}

.cursor {
  animation: blink 1s step-end infinite;
}

@keyframes blink {
  from, to { opacity: 1; }
  50% { opacity: 0; }
}

/* 响应式设计 */
@media (max-width: 600px) {
  .chat-container {
    padding: 10px;
    border-radius: 0;
    height: 100vh;
    box-shadow: none;
  }
  
  .chat-box {
    height: calc(100vh - 180px);
    padding: 15px;
  }
  
  .message {
    max-width: 85%;
  }
  
  .input-area {
    gap: 8px;
  }
  
  .input-area input {
    padding: 12px 15px;
    font-size: 15px;
  }
  
  .input-area button {
    padding: 12px 20px;
    font-size: 15px;
  }
}

</style>
