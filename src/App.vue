<template>
  <div class="chat-header">
    <div class="chat-header-avatar">
      <img src="/favicon.ico" alt="Chat Avatar">
    </div>
    <div class="chat-header-title">YIIY</div>
    <div class="chat-header-status">
      <span class="fa fa-circle" style="color: lightgreen">与你同在</span>
    </div>
  </div>

  <div class="chat-body scrollable">
    <div v-for="message in chatHistory[activeChatIndex]?.messages.filter(m => m.role !== 'system')" :key="message.id"
      class="chat-message" :class="getMessageClass(message.role)">
      <div class="chat-message-content">
        <p v-html="message.content"></p>
      </div>
    </div>
  </div>

  <div class="chat-footer">
    <label for="message-input">
      <div class="footer-options">
        <button class="emoji-button"><span class="fa fa-book"></span></button>
        <button class="file-button"><span class="fa fa-folder"></span></button>
        <button class="photo-button"><span class="fa fa-image"></span></button>
        <button class="send-button" @click="handleSendMessage" :disabled="isSending">
          <span class="fa fa-send"></span>
        </button>
      </div>
      <textarea id="message-input" rows="1" v-model="messageInput" placeholder="Type a message..." class="scrollable"
        v-on:input="autoHeight($event)"></textarea>
    </label>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue';
import { marked } from 'marked';
import Prism from 'prismjs';
import { app as config } from '@/config/config.json';

/** 
 * 自动调整输入框高度
 */
const autoHeight = (event) => {
  event.target.style.height = 'auto';
  event.target.style.height = (event.target.scrollHeight > (16 * 8) ? (16 * 8) : event.target.scrollHeight) + 'px';
};

/**
 * 初始化系统提示信息
 * @param {Array} customPrompts - 自定义提示数组
 * @returns {Array} 组合后的系统提示
 */
const initializeSystemPrompts = (customPrompts = []) => {
  const systemPrompts = [];

  // 开发者信息提示
  if (config.author?.able) {
    systemPrompts.push({
      role: 'system',
      content: `
        你的开发者的信息：
        姓名：${config.author.name || "NAME"}(${config.author.email || "EMAIL"})
        简介：${config.author.desc || "DESC"}
        链接：${config.author.link || "LINK"}
        (在被问到作者信息时，请用简介且优美的语句作答)
      `.replace(/\s+/g, ' ')
    });
  }

  // AI 系统提示
  if (config.ai.prompt) {
    systemPrompts.push({
      role: 'system',
      content: config.ai.prompt
    });
  }

  return [...systemPrompts, ...customPrompts];
};

/**
 * 获取消息样式类
 * @param {string} role - 消息角色
 * @returns {string} 对应的CSS类名
 */
const getMessageClass = (role) => {
  const roleClassMap = {
    user: 'chat-message-right',
    assistant: 'chat-message-left',
    default: 'chat-message-system fa fa-info-circle'
  };
  return roleClassMap[role] || roleClassMap.default;
};

/**
 * 处理消息发送
 */
const handleSendMessage = async () => {
  if (!messageInput.value.trim() || isSending.value) return;

  isSending.value = true;

  // 添加用户消息
  chatHistory.value[activeChatIndex.value].messages.push({
    role: "user",
    content: messageInput
      .value.trim()
      // 转义HTML特殊字符
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#039;')
  });

  // 清空输入框 && 重置高度
  messageInput.value = '';
  document.getElementById('message-input').style.height = 'auto';

  // 当前消息索引
  const currentMessageIndex = JSON.parse(JSON.stringify(chatHistory.value[activeChatIndex.value].messages));

  // 添加思考中状态
  const thinkingMessageIndex = chatHistory.value[activeChatIndex.value].messages.push({
    role: "assistant",
    content: "YIIY正在思考..."
  }) - 1;

  await scrollToBottom();

  try {
    // console.log(currentMessageIndex);
    const response = await fetch(config.ai.url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${config.ai.key}`
      },
      body: JSON.stringify({
        ...config.ai.params,
        messages: currentMessageIndex
      })
    });

    const data = await response.json();
    const replyContent = data.choices[0].message.content;

    // 处理Markdown内容
    const formattedContent = formatMarkdownContent(replyContent);
    displayTypingEffect(formattedContent, thinkingMessageIndex);
  } catch (error) {
    console.error('消息发送失败:', error);
    handleResponseError(thinkingMessageIndex);
  } finally {
    isSending.value = false;
    await scrollToBottom();
  }
};
window.addEventListener('keydown', (event) => {
  if (event.code === 'Enter' && event.ctrlKey) {
    event.preventDefault();
    handleSendMessage();
  }
});

/**
 * 格式化Markdown内容
 * @param {string} content - 原始内容
 * @returns {string} 格式化后的HTML
 */
const formatMarkdownContent = (content) => {
  const tempDiv = document.createElement('div');
  tempDiv.innerHTML = marked(content);
  Prism.highlightAllUnder(tempDiv);
  return tempDiv.innerHTML.match(/<[^>]+>|[\s\S]/g).filter(Boolean);
};

/**
 * 显示打字机效果
 * @param {Array} contentArray - 内容数组
 * @param {number} messageIndex - 消息索引
 */
const displayTypingEffect = (contentArray, messageIndex) => {
  chatHistory.value[activeChatIndex.value]
    .messages[messageIndex].content = '';
  let currentIndex = 0;
  const typingInterval = setInterval(() => {
    if (currentIndex < contentArray.length) {
      chatHistory.value[activeChatIndex.value].messages[messageIndex].content += contentArray[currentIndex];
      currentIndex++;
    } else {
      scrollToBottom();
      clearInterval(typingInterval);
    }
  }, 11);
};

/**
 * 处理响应错误
 * @param {number} messageIndex - 消息索引
 */
const handleResponseError = (messageIndex) => {
  chatHistory.value[activeChatIndex.value].messages[messageIndex].content =
    "YIIY无法生成回复，请稍后再试。";
};

/**
 * 滚动到底部
 */
const scrollToBottom = async () => {
  await nextTick();
  const chatBody = document.querySelector('.chat-body');
  if (chatBody) {
    chatBody.scrollTop = chatBody.scrollHeight;
  }
};


// 响应式数据
const chatHistory = ref([{ messages: initializeSystemPrompts() }]);
const activeChatIndex = ref(0);
const messageInput = ref('');
const isSending = ref(false);
</script>

<style>
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
  max-width: 95%;
  border-radius: 10px;
  margin: 5px;
  text-align: left;
  word-break: break-all;
}


.chat-message * {
  user-select: text !important;
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

.chat-message-content a {
  color: cornsilk;
  font-family: 'Fira Code VF';
  text-decoration: underline;
}

.chat-message-content img {
  width: 100%;
  height: auto;
}

.chat-message-content table {
  overflow: scroll;
  background-color: #00000033;
  border-radius: 5px;
  text-align: center;
  padding: 5px;
}

.chat-message-content table * {
  padding: 3px;
  border-radius: 5px;
  background-color: #ffffff11;
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
  border: none;
  padding: 10px;
  resize: none;
  font-size: 16px;
  font-family: Arial, sans-serif;
}
</style>
