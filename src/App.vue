<script setup>
import { ref, onMounted } from "vue";
import ChatHistory from "./components/ChatHistory.vue";
import { marked } from "marked";
import hljs from "highlight.js";
import "highlight.js/styles/github.css";

const messages = ref([]);
const newMessage = ref("");
const isLoading = ref(false);
const messagesContainer = ref(null);
const chatHistory = ref([]);
const currentChatIndex = ref(0);
const showThinking = ref(false);
const abortController = ref(null);
const typingInterval = ref(null);

// Cấu hình marked để hỗ trợ code highlighting
marked.setOptions({
  highlight: function (code, lang) {
    if (lang && hljs.getLanguage(lang)) {
      return hljs.highlight(code, { language: lang }).value;
    }
    return hljs.highlightAuto(code).value;
  },
  breaks: true,
});

const scrollToBottom = () => {
  if (messagesContainer.value) {
    setTimeout(() => {
      messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight;
    }, 50);
  }
};

const createNewChat = () => {
  chatHistory.value.unshift({
    timestamp: new Date(),
    title: "Cuộc trò chuyện mới",
    messages: [
      {
        role: "assistant",
        content: "Tôi có thể giúp gì cho bạn?",
      },
    ],
  });
  currentChatIndex.value = 0;
  messages.value = chatHistory.value[0].messages;
  scrollToBottom();
};

const selectChat = (index) => {
  currentChatIndex.value = index;
  messages.value = chatHistory.value[index].messages;
};

const deleteChat = (index) => {
  if (chatHistory.value.length > 1) {
    chatHistory.value.splice(index, 1);

    // Nếu xóa chat đang được chọn hoặc chat có index nhỏ hơn
    if (index <= currentChatIndex.value) {
      // Nếu xóa chat cuối cùng, chọn chat trước đó
      if (index === chatHistory.value.length) {
        currentChatIndex.value = Math.max(0, index - 1);
      } else {
        // Giữ nguyên index nhưng load chat mới
        currentChatIndex.value = Math.min(
          currentChatIndex.value,
          chatHistory.value.length - 1
        );
      }
      messages.value = chatHistory.value[currentChatIndex.value].messages;
    }
  } else {
    // Nếu đây là chat cuối cùng, tạo một chat mới
    createNewChat();
  }
};

onMounted(() => {
  createNewChat();
});

const processAIResponse = (content) => {
  if (!showThinking.value) {
    // Loại bỏ phần thinking của AI
    content = content.replace(/\[.*?\]/g, "");
  } else {
    // Khi bật show thinking, format lại các bước suy nghĩ
    content = content.replace(/\[(.*?)\]/g, (match, thinking) => {
      return `<div class="thinking-process">${thinking}</div>`;
    });
  }
  return marked(content);
};

const stopResponse = () => {
  if (abortController.value) {
    abortController.value.abort();
    abortController.value = null;
  }
  // Dừng interval nếu đang chạy
  if (typingInterval.value) {
    clearInterval(typingInterval.value);
    typingInterval.value = null;
  }
  isLoading.value = false;
};

const sendMessage = async () => {
  if (!newMessage.value.trim() || isLoading.value) return;

  const userMessage = newMessage.value;
  newMessage.value = "";

  messages.value.push({
    role: "user",
    content: userMessage,
  });

  // Nếu đây là tin nhắn đầu tiên của user, cập nhật title
  if (messages.value.length === 2) {
    const title =
      userMessage.length > 30
        ? userMessage.substring(0, 30) + "..."
        : userMessage;
    chatHistory.value[currentChatIndex.value].title = title;
  }

  // Cập nhật history
  chatHistory.value[currentChatIndex.value].messages = [...messages.value];

  scrollToBottom();
  isLoading.value = true;

  // Tạo controller mới cho request
  abortController.value = new AbortController();

  try {
    let finalResponse = "";

    if (showThinking.value) {
      const thinkingResponse = await fetch(
        "https://api.groq.com/openai/v1/chat/completions",
        {
          method: "POST",
          headers: {
            Authorization: `Bearer ${import.meta.env.VITE_GROQ_API_KEY}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            model: "mixtral-8x7b-32768",
            messages: [
              {
                role: "system",
                content:
                  "Bạn là một AI assistant thông minh. Hãy trả lời bằng tiếng Việt một cách rõ ràng và dễ hiểu.",
              },
              {
                role: "user",
                content:
                  "Hãy liệt kê các bước tổng quát để thực hiện câu hỏi sau, không cần quá chi tiết, chỉ cần các bước tổng quát: " +
                  userMessage,
              },
            ],
            temperature: 0.7,
            max_tokens: 1000,
          }),
          signal: abortController.value.signal,
        }
      );

      const thinkingData = await thinkingResponse.json();
      const thinkingContent = thinkingData.choices[0].message.content;

      messages.value.push({
        role: "assistant",
        content: "[",
      });

      let currentIndex = 0;
      await new Promise((resolve, reject) => {
        typingInterval.value = setInterval(() => {
          if (currentIndex < thinkingContent.length) {
            messages.value[
              messages.value.length - 1
            ].content = `[${thinkingContent.substring(0, currentIndex + 1)}`;
            currentIndex++;
            chatHistory.value[currentChatIndex.value].messages = [
              ...messages.value,
            ];
            scrollToBottom();
          } else {
            clearInterval(typingInterval.value);
            typingInterval.value = null;
            messages.value[messages.value.length - 1].content += "]\n\n";
            resolve();
          }
        }, 30);

        // Thêm cleanup function khi promise bị reject
        abortController.value.signal.addEventListener("abort", () => {
          clearInterval(typingInterval.value);
          typingInterval.value = null;
          reject(new DOMException("Aborted", "AbortError"));
        });
      });
    }

    const response = await fetch(
      "https://api.groq.com/openai/v1/chat/completions",
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${import.meta.env.VITE_GROQ_API_KEY}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          model: "mixtral-8x7b-32768",
          messages: [
            {
              role: "system",
              content:
                "Bạn là một AI assistant thông minh. Hãy trả lời bằng tiếng Việt một cách rõ ràng và dễ hiểu.",
            },
            {
              role: "user",
              content: userMessage,
            },
          ],
          temperature: 0.7,
          max_tokens: 1000,
        }),
        signal: abortController.value.signal,
      }
    );

    const data = await response.json();
    const mainResponse = data.choices[0].message.content;

    if (showThinking.value) {
      // Nếu đang show thinking, thêm kết quả chính vào ngay lập tức
      messages.value[messages.value.length - 1].content += mainResponse;
    } else {
      // Nếu không show thinking, tạo message mới với kết quả chính
      messages.value.push({
        role: "assistant",
        content: mainResponse,
      });
    }

    // Cập nhật history
    chatHistory.value[currentChatIndex.value].messages = [...messages.value];
    scrollToBottom();
  } catch (error) {
    if (error.name === "AbortError") {
      // Nếu không có message nào hoặc không phải đang show thinking
      if (!showThinking.value || messages.value.length === 0) {
        messages.value.push({
          role: "assistant",
          content: "Đã dừng trả lời.",
        });
      }
      // Nếu đang show thinking, giữ nguyên nội dung đã hiển thị
    } else {
      console.error("Lỗi:", error);
      messages.value.push({
        role: "assistant",
        content: "Xin lỗi, đã xảy ra lỗi. Vui lòng thử lại.",
      });
    }
    scrollToBottom();
  } finally {
    isLoading.value = false;
    abortController.value = null;
    if (typingInterval.value) {
      clearInterval(typingInterval.value);
      typingInterval.value = null;
    }
  }
};
</script>

<template>
  <div class="app-container">
    <ChatHistory
      :history="chatHistory"
      :currentIndex="currentChatIndex"
      :onSelectChat="selectChat"
      :onDeleteChat="deleteChat"
    />

    <div class="chat-container">
      <div class="header">
        <div class="left-controls">
          <button class="new-chat-btn" @click="createNewChat">
            <i class="fa-solid fa-plus"></i>
            NEW CHAT
          </button>
          <label class="think-toggle">
            <input type="checkbox" v-model="showThinking" />
            Show Think
          </label>
        </div>
        <div class="title">GROQ CLOUD</div>
      </div>

      <div ref="messagesContainer" class="messages">
        <div
          v-for="(message, index) in messages"
          :key="index"
          :class="['message', message.role]"
        >
          <div class="avatar">
            {{ message.role === "user" ? "M" : "G" }}
          </div>
          <div class="content">
            <div
              v-if="message.role === 'assistant'"
              v-html="processAIResponse(message.content)"
            ></div>
            <p v-else>{{ message.content }}</p>
          </div>
        </div>
        <div v-if="isLoading && showThinking" class="message assistant">
          <div class="avatar">G</div>
          <div class="content loading-content">
            <div class="typing-indicator">
              <span></span>
              <span></span>
              <span></span>
            </div>
          </div>
        </div>
      </div>

      <div class="input-container">
        <input
          v-model="newMessage"
          @keyup.enter="sendMessage"
          type="text"
          placeholder="Nhắn tin cho Groq Cloud..."
        />
        <button v-if="isLoading" class="stop-btn" @click="stopResponse">
          Stop
        </button>
        <button v-else class="send-btn" @click="sendMessage">Gửi</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  display: flex;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background-color: #ffffff;
  position: fixed;
  top: 0;
  left: 0;
}

.chat-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100%;
  position: relative;
}

.header {
  display: flex;
  align-items: center;
  padding: 0.5rem 1rem;
  background-color: #2196f3;
  color: white;
  height: 56px;
  position: sticky;
  top: 0;
  z-index: 10;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.left-controls {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.menu-btn {
  background: none;
  border: none;
  color: white;
  font-size: 1.2rem;
  cursor: pointer;
  padding: 0.5rem;
}

.new-chat-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: none;
  border: none;
  color: white;
  font-size: 0.9rem;
  cursor: pointer;
  padding: 0.5rem 1rem;
}

.think-toggle {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: white;
  font-size: 0.9rem;
}

.title {
  flex: 1;
  text-align: center;
  font-size: 1.2rem;
  font-weight: 500;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  background-color: #f8f9fa;
  height: calc(100vh - 56px - 76px);
}

.message {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
  padding: 0.5rem;
  max-width: 80%;
  width: fit-content;
}

.user {
  margin-left: auto;
  flex-direction: row-reverse;
}

.assistant {
  margin-right: auto;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  flex-shrink: 0;
}

.user .avatar {
  background-color: #2196f3;
  color: white;
}

.assistant .avatar {
  background-color: #4caf50;
  color: white;
}

.content {
  flex: 0 1 auto;
  padding: 1rem;
  border-radius: 8px;
  background-color: white;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  min-width: 0;
  word-wrap: break-word;
}

.user .content {
  background-color: #2196f3;
  color: white;
}

.assistant .content {
  background-color: white;
}

.user .content :deep(pre) {
  background-color: rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.user .content :deep(code) {
  color: white;
}

.loading-content {
  min-height: 32px;
  display: flex;
  align-items: center;
}

.typing-indicator {
  display: flex;
  gap: 4px;
  padding: 4px 8px;
}

.typing-indicator span {
  width: 8px;
  height: 8px;
  background-color: #90caf9;
  border-radius: 50%;
  animation: bounce 1.4s infinite ease-in-out;
}

.typing-indicator span:nth-child(1) {
  animation-delay: -0.32s;
}

.typing-indicator span:nth-child(2) {
  animation-delay: -0.16s;
}

@keyframes bounce {
  0%,
  80%,
  100% {
    transform: translateY(0);
  }
  40% {
    transform: translateY(-8px);
  }
}

.input-container {
  display: flex;
  gap: 0.5rem;
  padding: 1rem;
  background-color: white;
  border-top: 1px solid #ddd;
  position: sticky;
  bottom: 0;
  z-index: 10;
}

input {
  flex: 1;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
}

.send-btn {
  padding: 0.75rem 1.5rem;
  background-color: #2196f3;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
}

.send-btn:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.send-btn:hover:not(:disabled) {
  background-color: #1976d2;
}

/* Markdown styles */
.message :deep(pre) {
  background-color: #f6f8fa; /* Nền sáng hơn */
  padding: 1.2rem;
  border-radius: 8px;
  overflow-x: auto;
  margin: 0.5rem 0;
  max-width: 100%;
  white-space: pre-wrap;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  border: 1px solid #e1e4e8;
}

.message :deep(code) {
  font-family: "Fira Code", monospace;
  font-size: 0.95em;
  line-height: 1.5;
  color: #24292e; /* Màu chữ tối cho nền sáng */
}

/* Syntax highlighting cho code - màu sắc dễ đọc hơn */
.message :deep(.hljs-keyword) {
  color: #d73a49; /* đỏ */
}

.message :deep(.hljs-string) {
  color: #22863a; /* xanh lá */
}

.message :deep(.hljs-number) {
  color: #005cc5; /* xanh dương */
}

.message :deep(.hljs-function) {
  color: #6f42c1; /* tím */
}

.message :deep(.hljs-comment) {
  color: #6a737d; /* xám */
  font-style: italic;
}

.message :deep(.hljs-variable) {
  color: #24292e; /* đen */
}

.message :deep(.hljs-operator) {
  color: #d73a49; /* đỏ */
}

.message :deep(.hljs-class) {
  color: #6f42c1; /* tím */
}

.message :deep(.hljs-property) {
  color: #005cc5; /* xanh dương */
}

.message :deep(.hljs-include) {
  color: #22863a; /* xanh lá */
}

/* Style cho phần thinking */
.message.assistant .content :deep(.thinking-process) {
  color: #1976d2; /* Thay đổi từ màu xám sang màu xanh đậm */
  font-size: 0.95em;
  line-height: 1.5;
  margin-bottom: 1.5rem;
  padding: 0.8rem;
  background-color: #f5f9ff; /* Thêm background nhẹ */
  border-left: 4px solid #1976d2;
  border-radius: 4px;
}

/* Style cho các phần nhấn mạnh */
.message.assistant .content :deep(strong) {
  color: #e91e63; /* Màu hồng đậm */
  font-weight: 600;
}

.message.assistant .content :deep(em) {
  color: #2196f3; /* Màu xanh */
  font-style: normal;
  font-weight: 500;
}

/* Style cho inline code */
.message.assistant .content :deep(:not(pre) > code) {
  background-color: #f3f3f3;
  color: #e91e63;
  padding: 0.2rem 0.4rem;
  border-radius: 4px;
  font-size: 0.9em;
  border: 1px solid #ddd;
}

/* Style cho blockquote */
.message.assistant .content :deep(blockquote) {
  margin: 1rem 0;
  padding: 0.5rem 1rem;
  border-left: 4px solid #2196f3;
  background-color: #f5f9ff;
  color: #1976d2;
  font-style: italic;
}

/* Thêm style cho input khi bị disabled */
input:disabled {
  background-color: #f5f5f5;
  cursor: not-allowed;
}

.stop-btn {
  padding: 0.75rem 1.5rem;
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.stop-btn:hover {
  background-color: #c82333;
}
</style>
