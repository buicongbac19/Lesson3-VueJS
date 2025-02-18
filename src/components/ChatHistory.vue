<script setup>
import { defineProps, defineEmits } from "vue";

const props = defineProps({
  history: {
    type: Array,
    required: true,
  },
  currentIndex: {
    type: Number,
    required: true,
  },
  onSelectChat: {
    type: Function,
    required: true,
  },
  onDeleteChat: {
    type: Function,
    required: true,
  },
});

const emit = defineEmits(["deleteChat"]);

const handleDelete = (index, event) => {
  event.stopPropagation(); // Ngăn không cho sự kiện click lan đến chat-item
  props.onDeleteChat(index);
};
</script>

<template>
  <div class="chat-history">
    <div
      v-for="(chat, index) in history"
      :key="index"
      :class="['chat-item', { active: index === currentIndex }]"
      @click="onSelectChat(index)"
    >
      <div class="chat-preview">
        <div class="preview-header">
          <i class="fa-solid fa-message"></i>
          <span class="timestamp">{{
            new Date(chat.timestamp).toLocaleString()
          }}</span>
          <button class="delete-btn" @click="handleDelete(index, $event)">
            x
          </button>
        </div>
        <p class="preview-text">
          {{ chat.title }}
        </p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.chat-history {
  width: 260px;
  height: 100%;
  border-right: 1px solid #e0e0e0;
  background-color: #f5f5f5;
  overflow-y: auto;
}

.chat-item {
  padding: 0.75rem;
  border-bottom: 1px solid #e0e0e0;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
}

.chat-item::before {
  content: "";
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  width: 4px;
  background-color: #2196f3;
  opacity: 0;
  transition: opacity 0.3s ease;
}

.chat-item.active {
  background-color: #e3f2fd;
}

.chat-item.active::before {
  opacity: 1;
}

.chat-item:hover:not(.active) {
  background-color: #e0e0e0;
}

.chat-preview {
  font-size: 0.9rem;
}

.preview-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: #666;
  margin-bottom: 0.5rem;
  position: relative;
}

.timestamp {
  font-size: 0.8rem;
  flex: 1;
}

.preview-text {
  color: #333;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-weight: 500;
}

.delete-btn {
  background: none;
  border: none;
  color: #dc3545;
  cursor: pointer;
  padding: 0.25rem;
  font-size: 0.9rem;
  opacity: 0;
  transition: opacity 0.2s;
}

.chat-item:hover .delete-btn {
  opacity: 1;
}

.delete-btn:hover {
  color: #c82333;
}
</style>
