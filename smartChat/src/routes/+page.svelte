<script>
// @ts-nocheck
import { onMount } from 'svelte';
import { BaseDirectory, exists, readFile, create, remove, readTextFile, writeTextFile } from '@tauri-apps/plugin-fs';
let currentMessage = $state("");
let messageHistory = $state([]);
let isLoading = $state(false);
let allChats = $state([]);
let currentChatId = $state(null);
let availableModels = $state([]);
let currentModel = $state("llama3.1:latest");
let showModelDropdown = $state(false);
let editingChatId = $state(null);
const MAIN_PROMPT = '';

onMount(async () => {
  await loadAllChats();
  await fetchAvailableModels();
  if (allChats.length === 0) {
    createNewChat();
  } else {
    currentChatId = allChats[0].id;
    messageHistory = allChats[0].messages;
  }
  scrollToBottom();
});

async function fetchAvailableModels() {
  try {
    const response = await fetch('http://localhost:11434/api/tags');
    const data = await response.json();
    availableModels = data.models || [];
  } catch (error) {
    console.error('Error fetching available models:', error);
    availableModels = [{ name: currentModel }];
  }
}

function selectModel(modelName) {
  currentModel = modelName;
  showModelDropdown = false;
}

function saveChatName(chatId, newName) {
  const chatIndex = allChats.findIndex(chat => chat.id === chatId);
  if (chatIndex >= 0) {
    allChats[chatIndex].name = newName || "Unnamed Chat";
    saveAllChats();
  }
  editingChatId = null;
}

function handleEditKeydown(event, chatId, currentName) {
  if (event.key === 'Enter') {
    saveChatName(chatId, event.target.value);
  } else if (event.key === 'Escape') {
    editingChatId = null;
  }
}

async function saveAllChats() {
  try {
    if(!await exists('chatHistory.json', { baseDir: BaseDirectory.AppData })){
      await create('chatHistory.json', { baseDir: BaseDirectory.AppData });
    }

    if (currentChatId) {
      const chatIndex = allChats.findIndex(chat => chat.id === currentChatId);
      if (chatIndex >= 0) {
        allChats[chatIndex].messages = messageHistory;
      }
    }
    
    await writeTextFile('chatHistory.json', JSON.stringify(allChats), { 
      create: true, 
      baseDir: BaseDirectory.AppData,
    });
  } catch (error) {
    console.error('Error saving chat history:', error);
  }
}

async function loadAllChats() {
  try {
    if(!await exists('chatHistory.json', { baseDir: BaseDirectory.AppData })){
      await create('chatHistory.json', { baseDir: BaseDirectory.AppData });
    }
    const saved = await readTextFile('chatHistory.json', { baseDir: BaseDirectory.AppData });
    allChats = saved && saved.trim() ? JSON.parse(saved) : [];
  } catch (error) {
    console.error('Error loading chat history:', error);
    allChats = [];
  }
}

function createNewChat() {
  const newChat = {
    id: Date.now().toString(),
    name: "New Chat",
    messages: [{ content: MAIN_PROMPT, role: 'system' }]
  };
  
  allChats = [...allChats, newChat];
  currentChatId = newChat.id;
  messageHistory = newChat.messages;
  saveAllChats();
}

function selectChat(chatId) {
  if (currentChatId) {
    const currentChatIndex = allChats.findIndex(chat => chat.id === currentChatId);
    if (currentChatIndex >= 0) {
      allChats[currentChatIndex].messages = messageHistory;
    }
  }
  
  currentChatId = chatId;
  const selectedChat = allChats.find(chat => chat.id === chatId);
  if (selectedChat) {
    messageHistory = selectedChat.messages;
  }
  scrollToBottom();
}

async function deleteChat(chatId) {

  if (allChats.length <= 1) {
    createNewChat();
  }

  allChats = allChats.filter(chat => chat.id !== chatId);

  if (chatId === currentChatId) {
    currentChatId = allChats[0].id;
    messageHistory = allChats[0].messages;
  }

  await saveAllChats();
}

async function clearHistory() {
  if (currentChatId) {
    allChats = allChats.filter(chat => chat.id !== currentChatId);
    
    if (allChats.length === 0) {
      createNewChat();
    } else {
      currentChatId = allChats[0].id;
      messageHistory = allChats[0].messages;
    }
    
    await saveAllChats();
  }
}

async function getOllamaResponse(message) {
  try {
    const megaPrompt = messageHistory.map(msg => {
      if (msg.role === "system") return msg.content;
      if (msg.role === "user") return `User: ${msg.content}`;
      if (msg.role === "assistant") return `AI: ${msg.content}`;
      return "";
    }).join("\n\n") + "\n\nUser: " + message + "\n\nAI: ";
    
    const response = await fetch('http://localhost:11434/api/generate', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: currentModel,
        prompt: megaPrompt,
        stream: false
      }),
    });
    const data = await response.json();
    return data.response;
  } catch (error) {
    console.error('Error calling Ollama API:', error);
    return "Sorry, I encountered an error while processing your request.";
  }
}

async function handleSubmit(event) {
  event.preventDefault();
  if (currentMessage.trim() !== "") {
    const userMessage = { content: currentMessage, role: 'user' };
    messageHistory = [...messageHistory, userMessage]; 
    const userInput = currentMessage;
    currentMessage = "";

    isLoading = true;
    const aiResponse = await getOllamaResponse(userInput);

    const assistantMessage = { content: aiResponse, role: 'assistant' };
    messageHistory = [...messageHistory, assistantMessage];

    await saveAllChats();

    isLoading = false;
    scrollToBottom();
  }
}

function scrollToBottom() {
  setTimeout(() => {
    const chatContainer = document.querySelector(".chat-messages");
    if (chatContainer) {
      chatContainer.scrollTo({
        top: chatContainer.scrollHeight,
        behavior: 'smooth'
      });
    }
  }, 50);
}
</script>

<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@20..48,100..700,0..1,-50..200&icon_names=delete,smart_toy" />

<main class="container">
  <div class="sidebar">
    <div class="chats-header">
      <h3>Chats</h3>
      <button type="button" class="new-chat-btn" onclick={createNewChat}>+ New</button>
    </div>
    <div class="chats-list">
      {#each allChats as chat}
        <div class="chat-item-container">
          {#if editingChatId === chat.id}
            <!-- Edit mode: show text input -->
            <input
              type="text"
              class="chat-name-input {chat.id === currentChatId ? 'active' : ''}"
              value={chat.name}
              onblur={e => saveChatName(chat.id, e.target.value)}
              onkeydown={e => handleEditKeydown(e, chat.id, chat.name)}
            />
          {:else}
            <!-- Normal mode: show button that can be double-clicked -->
            <button 
              type="button"
              class="chat-item {chat.id === currentChatId ? 'active' : ''}" 
              onclick={() => selectChat(chat.id)}
              ondblclick={() => editingChatId = chat.id}>
              {chat.name}
            </button>
          {/if}
          
          <button 
            type="button"
            class="delete-chat-btn"
            onclick={(e) => {
              e.stopPropagation();
              deleteChat(chat.id);
            }}
            title="Delete chat">
            <span class="material-symbols-outlined">delete</span>
          </button>
        </div>
      {/each}
    </div>
  </div>

  <div class="chat-area">
    <div class="chat-container">
      <div class="chat-messages">
        {#each messageHistory.filter(msg => msg.role !== 'system') as message}
          <pre class="message {message.role}-message">{message.content}</pre>
        {/each}
        {#if isLoading}
          <pre class="message assistant-message loading">Thinking<span>.</span><span>.</span><span>.</span></pre>
        {/if}
        <div class="message-spacer"></div>
      </div>
    </div>

    <form class="row" onsubmit={handleSubmit}>
      <textarea 
        id="message-input" 
        placeholder="Type a massage" 
        bind:value={currentMessage}
        rows="2"
        onkeydown={(e) => {
          if (e.key === 'Enter' && !e.shiftKey) {
            e.preventDefault();
            handleSubmit(e);
          }
        }}
      ></textarea>
      
      <div class="buttons-column">
        <div class="model-selector">
          <button 
            type="button" 
            class="model-btn" 
            onclick={() => showModelDropdown = !showModelDropdown}>
            AI Models
          </button>
          
          {#if showModelDropdown}
            <div class="model-dropdown">
              {#each availableModels as model}
                <button 
                  type="button" 
                  class="model-option {currentModel === model.name ? 'active' : ''}"
                  onclick={() => selectModel(model.name)}>
                  {model.name}
                </button>
              {/each}
            </div>
          {/if}
        </div>
        
        <button type="submit" disabled={isLoading} class="send-btn">Send</button>
      </div>
    </form>
  </div>
</main>

<style>
  /* ==================== */
  /* BASE & GLOBAL STYLES */
  /* ==================== */
  
  :global(body, html) {
    margin: 0;
    padding: 0;
    overflow: hidden;
    height: 100%;
    width: 100%;
  }

  :global(.material-symbols-outlined) {
    font-variation-settings:
    'FILL' 0,
    'wght' 400,
    'GRAD' 0,
    'opsz' 24
  }

  :root {
    font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
    font-size: 16px;
    line-height: 24px;
    font-weight: 400;
    color: #f6f6f6;
    background-color: #2f2f2f;
    font-synthesis: none;
    text-rendering: optimizeLegibility;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    -webkit-text-size-adjust: 100%;
  }

  /* ==================== */
  /* LAYOUT COMPONENTS    */
  /* ==================== */

  .container {
    display: flex;
    height: 100vh;
    width: 100%;
    overflow: hidden;
  }

  /* ==================== */
  /* SIDEBAR COMPONENTS   */
  /* ==================== */

  .sidebar {
    width: 30%;
    background-color: #0d1526;
    height: 100vh;
    border-right: 1px solid #2a3a5e;
    display: flex;
    flex-direction: column;
  }

  .chats-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;
    border-bottom: 1px solid #2a3a5e;
  }

  .chats-header h3 {
    margin: 0;
    color: #ffffff;
  }

  .new-chat-btn {
    padding: 0.4em 0.8em;
    background-color: #2a3a5e;
    color: white;
  }

  .chats-list {
    flex: 1;
    overflow-y: auto;
    padding: 0.5rem;
    display: flex;
    flex-direction: column;
  }

  .chat-item {
    padding: 0.75rem;
    margin: 0.25rem 0;
    border-radius: 6px;
    cursor: pointer;
    color: #ffffff;
    transition: background-color 0.2s;
    width: 100%;
    display: block;
    text-align: left;
    border: none;
    background-color: transparent;
    box-shadow: none;
    font-family: inherit;
    font-size: inherit;
  }

  .chat-item:hover {
    background-color: #1c2a45;
  }

  .chat-item.active {
    background-color: #2a3a5e;
  }

  /* ==================== */
  /* CHAT COMPONENTS      */
  /* ==================== */

  .chat-area {
    width: 70%;
    display: flex;
    flex-direction: column;
    height: 100vh;
  }

  .chat-container {
    border: 1px solid #2a3a5e;
    border-radius: 8px;
    background-color: #0d1526;
    margin: 1rem;
    height: 80%;
    overflow: hidden;
  }

  .chat-messages {
    width: 100%;
    height: 100%;
    overflow-y: auto;
    padding: 1rem;
    padding-bottom: 2rem;
    display: flex;
    flex-direction: column;
    color: #ffffff;
    box-sizing: border-box;
    scroll-behavior: smooth;
    scrollbar-width: thin;
    scrollbar-color: #3a5a8e #0d1526;
  }

  .chat-messages::-webkit-scrollbar {
    width: 8px;
  }

  .chat-messages::-webkit-scrollbar-track {
    background: #0d1526;
    margin: 4px 0;
  }

  .chat-messages::-webkit-scrollbar-thumb {
    background-color: #3a5a8e;
    border-radius: 20px;
    border: 2px solid #0d1526;
    background-clip: content-box;
  }

  .chat-messages::-webkit-scrollbar-thumb:hover {
    background-color: #5a7ab0;
  }

  .message-spacer {
    height: 10px;
    width: 100%;
  }

  /* ==================== */
  /* MESSAGE STYLES       */
  /* ==================== */

  .message {
    padding: 8px 12px;
    border-radius: 8px;
    margin: 4px 0;
    max-width: 80%;
    word-break: break-word;
    text-align: left;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
    color: #ffffff;
    font-family: inherit;
    white-space: pre-wrap;
  }

  .user-message {
    background-color: #2c3e5d;
    align-self: flex-end;
  }

  .assistant-message {
    background-color: #1c2a45;
    align-self: flex-start;
  }

  .loading span {
    animation: dots 1.5s infinite;
    opacity: 0;
  }

  .loading span:nth-child(1) {
    animation-delay: 0s;
  }

  .loading span:nth-child(2) {
    animation-delay: 0.5s;
  }

  .loading span:nth-child(3) {
    animation-delay: 1s;
  }

  @keyframes dots {
    0%, 100% { opacity: 0; }
    50% { opacity: 1; }
  }

  .chat-name-input {
    width: 100%;
    padding: 0.75rem;
    margin: 0;
    border-radius: 6px;
    background-color: #1c2a45;
    color: #ffffff;
    font-family: inherit;
    font-size: inherit;
    border: 1px solid #3a5a8e;
  }
  
  .chat-name-input.active {
    background-color: #2a3a5e;
  }

  /* ==================== */
  /* FORM COMPONENTS      */
  /* ==================== */

  .row {
    display: flex;
    padding: 1rem;
    height: 20%;
    width: calc(100% - 2rem);
    margin: 0 1rem;
    box-sizing: border-box;
    background-color: rgba(47, 47, 47, 0.8);
    align-items: center;
  }

  button, textarea {
    border-radius: 8px;
    border: 1px solid transparent;
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    color: #ffffff;
    background-color: #0f0f0f98;
    transition: border-color 0.25s;
    box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
    outline: none;
  }

  button {
    cursor: pointer;
  }

  button:hover {
    border-color: #396cd8;
  }

  button:active {
    background-color: #0f0f0f69;
  }

  textarea {
    flex: 1;
    margin: 0 5px;
    resize: none;
    line-height: 1.5;
  }

  .chat-item-container {
    display: flex;
    align-items: center;
    margin: 0.25rem 0;
    width: 100%;
  }
  
  .chat-item {
    flex: 1;
    margin: 0;
  }
  
  .delete-chat-btn {
    padding: 0.4em;
    background: transparent;
    color: #ffffff;
    border: none;
    margin-left: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0.6;
    transition: opacity 0.2s, background-color 0.2s;
    box-shadow: none;
  }
  
  .delete-chat-btn:hover {
    opacity: 1;
    background-color: rgba(183, 38, 29, 0.3);
    border-color: transparent;
  }
  
  .delete-chat-btn .material-symbols-outlined {
    font-size: 18px;
  }

  /* Model selector styles */
  .buttons-column {
    display: flex;
    flex-direction: column;
    height: 100%;
    width: 120px;
    margin-left: 5px;
  }

  .model-selector {
    position: relative;
    height: 50%;
    margin-bottom: 5px;
  }

  .model-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100%;
    width: 100%;
    padding: 0.6em;
    font-size: 0.9em;
    white-space: nowrap;
  }

  .model-dropdown {
    position: absolute;
    bottom: 100%;
    right: 0;
    background-color: #0f0f0f;
    border-radius: 8px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
    overflow: hidden;
    z-index: 10;
    margin-bottom: 8px;
    max-height: 300px;
    overflow-y: auto;
    min-width: 180px;
  }

  .model-option {
    display: block;
    width: 100%;
    text-align: left;
    border: none;
    border-radius: 0;
    padding: 0.6em 1em;
    background-color: transparent;
    cursor: pointer;
    margin: 0;
    box-shadow: none;
  }

  .model-option:hover {
    background-color: #1a1a1a;
  }

  .model-option.active {
    background-color: #2a2a4a;
  }

  .send-btn {
    height: 50%;
    width: 100%;
  }

  /* ==================== */
  /* RESPONSIVE LAYOUT    */
  /* ==================== */

  @media (max-width: 768px) {
    .container {
      flex-direction: column;
    }
    
    .sidebar {
      width: 100%;
      height: 30vh;
      border-right: none;
      border-bottom: 1px solid #2a3a5e;
    }
    
    .chat-area {
      width: 100%;
      height: 70vh;
    }
    
    .chat-container {
      height: 75%;
      margin: 0.5rem;
    }
    
    .row {
      height: 25%;
      padding: 0.5rem;
      margin: 0 0.5rem;
    }
    
    .buttons-column {
      width: 100px;
    }
  }
</style>
