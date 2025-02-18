<script setup>
import { RouterLink, RouterView } from "vue-router";
import { ref, onMounted } from "vue";

const messages = ref([]);
const newMessage = ref("");
const isLoading = ref(false);
const messagesContainer = ref(null);

const scrollToBottom = () => {
  if (messagesContainer.value) {
    setTimeout(() => {
      messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight;
    }, 50);
  }
};

onMounted(() => {
  messages.value.push({
    role: "assistant",
    content: "Xin chào, tôi có thể giúp gì cho bạn?",
  });
  scrollToBottom();
});

const sendMessage = async () => {
  if (!newMessage.value.trim()) return;

  const userMessage = newMessage.value;

  newMessage.value = "";
  messages.value.push({
    role: "user",
    content: userMessage,
  });
  scrollToBottom();

  isLoading.value = true;

  try {
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
              role: "user",
              content: userMessage,
            },
          ],
          temperature: 0.7,
          max_tokens: 1000,
        }),
      }
    );

    const data = await response.json();

    messages.value.push({
      role: "assistant",
      content: data.choices[0].message.content,
    });
    scrollToBottom();
  } catch (error) {
    console.error("Lỗi:", error);
    messages.value.push({
      role: "assistant",
      content: "Xin lỗi, đã xảy ra lỗi. Vui lòng thử lại.",
    });
    scrollToBottom();
  }

  isLoading.value = false;
};
</script>

<template>
  <RouterView />

  <div class="chat-container">
    <div ref="messagesContainer" class="messages">
      <div
        v-for="(message, index) in messages"
        :key="index"
        :class="['message', message.role]"
      >
        <p>{{ message.content }}</p>
      </div>
      <div v-if="isLoading" class="loading">Đang xử lý...</div>
    </div>

    <div class="input-container">
      <input
        v-model="newMessage"
        @keyup.enter="sendMessage"
        placeholder="Nhập tin nhắn..."
        type="text"
      />
      <button @click="sendMessage" :disabled="isLoading">Gửi</button>
    </div>
  </div>
</template>

<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

nav {
  width: 100%;
  font-size: 12px;
  text-align: center;
  margin-top: 2rem;
}

nav a.router-link-exact-active {
  color: var(--color-text);
}

nav a.router-link-exact-active:hover {
  background-color: transparent;
}

nav a {
  display: inline-block;
  padding: 0 1rem;
  border-left: 1px solid var(--color-border);
}

nav a:first-of-type {
  border: 0;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }

  nav {
    text-align: left;
    margin-left: -1rem;
    font-size: 1rem;

    padding: 1rem 0;
    margin-top: 1rem;
  }
}

.chat-container {
  width: 80vw;
  margin: 0 auto;
  padding: 20px;
  height: 80vh;
  display: flex;
  flex-direction: column;
}

.messages {
  flex: 1;
  overflow-y: auto;
  margin-bottom: 20px;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.message {
  margin-bottom: 15px;
  padding: 10px;
  border-radius: 8px;
}

.user {
  background-color: #e3f2fd;
  margin-left: 20%;
}

.assistant {
  background-color: #f5f5f5;
  margin-right: 20%;
}

.loading {
  text-align: center;
  color: #666;
  font-style: italic;
}

.input-container {
  display: flex;
  gap: 10px;
}

input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}

button {
  padding: 10px 20px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: #45a049;
}
</style>
