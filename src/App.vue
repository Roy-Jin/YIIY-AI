<template>
  <div class="chat-header">
    <div class="chat-header-avatar"><img src="/favicon.ico"></div>
    <div class="chat-header-title">YIIY</div>
    <div class="chat-header-status"><span class="fa fa-circle" style="color: lightgreen">与你同在</span></div>
  </div>
  <div class="chat-body scrollable">
    <div class="chat-message" v-for="mess in history[currentChat]?.messages.filter(m => m.role !== 'system')"
      :class="computedClass(mess.role)">
      <div class="chat-message-content">
        <p v-html="mess.content"></p>
      </div>
    </div>
  </div>
  <div class="chat-footer">
    <label for="message-input">
      <div class="footer-options">
        <button @click="" class="emoji-button"><span class="fa fa-book"></span></button>
        <button @click="" class="file-button"><span class="fa fa-folder"></span></button>
        <button @click="" class="photo-button"><span class="fa fa-image"></span></button>
        <button @click="send()" class="send-button"><span class="fa fa-send"></span></button>
      </div>
      <textarea placeholder="Type a message..." v-model="message" class="scrollable"></textarea>
    </label>
  </div>
</template>

<script setup>
import { app as config } from '@/config/config.json';
import { ref, nextTick } from 'vue';
import { marked } from 'marked';

const initPrompt = (_prompt = []) => {
  let prompt = [];
  config.author?.able && // 开发者信息
    prompt.push({
      role: 'system', content: `你的开发者的信息：
      姓名：${config.author.name || "NAME"}(${config.author.email || "EMAIL"})
      简介：${config.author.desc || "DESC"}
      链接：${config.author.link || "LINK"}
      (在被问到作者信息时，请用简介且优美的语句作答)`.replace(/\s+/g, '')
    });
  config.ai.prompt && // 机器人提示信息
    prompt.push({
      role: 'system', content: config.ai.prompt
    });
  return [...prompt, ..._prompt];
};
let history = ref([{ "messages": initPrompt() }]);
let currentChat = ref(0);

const message = ref('');

const send = () => {
  // 将用户消息添加到对话历史中
  history.value[currentChat.value].messages.push({
    "role": "user",
    "content": message.value
  });

  // 清空输入框
  message.value = '';

  // 滚动到底部
  nextTick(() => {
    const chatBody = document.querySelector('.chat-body');
    chatBody.scrollTop = chatBody.scrollHeight;
  });

  // 调用后端GLM-4模型获取回复
  fetch(config.ai.url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${config.ai.key}`
    },
    body: JSON.stringify({ ...config.ai.params, ...{ "messages": history.value[currentChat.value].messages } })
  })
    .then(response => response.json())
    .then(data => {
      console.log(data);
      const reply = data.choices[0].message.content;
      history.value[currentChat.value].messages.push({
        "role": "assistant",
        "content": marked(reply)
      });

      // 滚动到底部
      nextTick(() => {
        const chatBody = document.querySelector('.chat-body');
        chatBody.scrollTop = chatBody.scrollHeight;
      });

    })
    .catch(error => {
      console.error('Error:', error);
    });
}

const computedClass = (role) => {
  let result;
  switch (role) {
    case 'user':
      result = 'chat-message-right';
      break;
    case 'assistant':
      result = 'chat-message-left';
      break;
    default:
      result = 'chat-message-system fa fa-info-circle';
      break;
  }
  return result;
}
</script>

<style scoped>
.chat-header {
  display: flex;
  align-items: center;
  padding: 10px;
  border-bottom: var(--border);
}

.chat-header-avatar {
  width: 40px;
  height: 40px;
  margin-right: 10px;
}

.chat-header-avatar img {
  width: 100%;
  height: 100%;
  border-radius: 10px;
  object-fit: cover;
}

.chat-header-title {
  margin-left: 10px;
  font-size: 18px;
  font-weight: bold;
}

.chat-header-status {
  font-size: 14px;
  color: #999;
  margin-left: auto;
}

.chat-body {
  padding: 10px;
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.chat-message {
  padding: 10px;
  width: fit-content;
  max-width: 75%;
  border-radius: 10px;
  margin: 5px;
  text-align: left;
  word-break: break-all;
}

.chat-message-system {
  display: flex;
  align-self: center;
  color: brown;
  background-color: transparent;
}

.chat-message-left {
  align-self: flex-start;
  background-color: #0c8eaf;
}

.chat-message-right {
  align-self: flex-end;
  background-color: #39a864;
}

.chat-message-avatar {
  width: 30px;
  height: 30px;
  margin-right: 10px;
}

.chat-message-avatar img {
  width: 100%;
  height: 100%;
  border-radius: 10px;
  object-fit: cover;
}

.chat-message-content {
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.chat-message-content p {
  font-size: 16px;
  margin: 0;
  font-family: Arial, sans-serif;
  letter-spacing: unset;
}

.chat-footer {
  border-top: var(--border);
}

.chat-footer label {
  display: flex;
  flex-direction: column;
  padding: 10px;
}

.footer-options {
  display: flex;
  flex-direction: row;
  gap: 10px;
}

.footer-options button {
  font-size: 1rem;
  margin: 0 0 5px 0;
}

.send-button {
  font-weight: bold;
  border-radius: 5px;
  padding: 5px;
  margin-left: auto !important;
  background-color: #4CAF50 !important;
}

.send-button::after {
  content: "发送";
  margin-left: 3px;
}

.chat-footer textarea {
  flex: 1;
  border: none;
  padding: 10px;
  resize: none;
  min-height: 133px;
  font-size: 16px;
  font-family: Arial, sans-serif;
}
</style>
