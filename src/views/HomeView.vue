<template>
  <div class="container-fluid d-flex vh-100">
    <aside class="col-3 bg-light p-3">
      <h2 class="h5">Lịch sử Chat</h2>
      <ul class="list-group">
        <li class="list-group-item list-group-item-action" v-for="(chat, index) in chatHistory" :key="index"
          @click="loadChat(index)">
          {{ chat.title }}
        </li>
      </ul>
    </aside>
    <main class="col d-flex flex-column">
      <header class="d-flex justify-content-end bg-secondary p-2">
        <RouterLink v-if="checkLogin() !== null && userRoleId === 2" :to="{ name: 'dashboard' }" class="btn btn-danger">
          Trang quản trị
        </RouterLink>

        <button v-if="checkLogin() !== null" @click="handleLogout" class="btn btn-success">Đăng xuất</button>

        <button v-if="checkLogin() === null" @click="toggleLogin" class="btn btn-primary me-2">Đăng nhập</button>
      </header>
      <div class="flex-grow-1 p-3 overflow-auto">
        <div v-for="(msg, index) in messages" :key="index" v-html="convertMarkdown(msg.content)" class="chat_template"
          :class="['alert', { 'alert-primary text-end': msg.role === 'user', 'alert-success': msg.role !== 'user' }]">
        </div>
      </div>
      <footer class="d-flex p-3 bg-light">
        <input v-model="userInput" @keyup.enter="sendMessage" class="form-control me-2"
          placeholder="Nhập tin nhắn..." />
        <button @click="sendMessage" class="btn btn-success">Gửi</button>
      </footer>
    </main>
  </div>
  <LoginPopup v-if="showLogin" @close="toggleLogin" @login-success="handleLoginSuccess" />
</template>
<script>
import { onMounted, ref, watchEffect } from "vue";
import LoginPopup from "./LoginPopup.vue";
import RegisterPopup from "./RegisterPopup.vue";
import { marked } from "marked";

export default {
  components: { LoginPopup, RegisterPopup },
  setup() {
    const chatHistory = ref([]);
    const messages = ref([]);
    const userInfo = ref(null);
    const userRoleId = ref(null); // Lưu role_id của user
    const userInput = ref("");
    const showLogin = ref(false);
    const showRegister = ref(false);
    const assistantMessage = ref();
    const apiToken = import.meta.env.VITE_API_TOKEN;
    const userId = ref(sessionStorage.getItem("user_id"));

    const checkLogin = () => {
      return userId.value;
    };

    const getInformationOfUser = async (id) => {
      try {
        const response = await fetch(`http://localhost:4999/user/${id}`);
        const data = await response.json();
        if (data.status) {
          userInfo.value = data.data;
          userRoleId.value = data.data.role_id; // Cập nhật role_id
        }
      } catch (error) {
        console.error("Lỗi khi lấy thông tin user:", error);
      }
    };

    const toggleLogin = () => {
      showLogin.value = !showLogin.value;
    };

    const handleLoginSuccess = async (id) => {
      sessionStorage.setItem("user_id", id);
      userId.value = id;
      showLogin.value = false;
      await getInformationOfUser(id); // Gọi API lấy thông tin user sau khi đăng nhập
    };

    const handleLogout = () => {
      sessionStorage.removeItem("user_id");
      userId.value = null;
      userInfo.value = null;
      userRoleId.value = null; // Reset role_id khi đăng xuất
      alert("Bạn đã đăng xuất thành công!");
    };
    const convertMarkdown = (text) => {
      return marked(text);
    };
    const sendMessage = async () => {

      messages.value.push({
        "role": "user",
        'content': userInput.value
      })
      userInput.value = ""
      let data = { messages: messages.value }
      const response = await fetch("http://localhost:5000/assistant/chat", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": "Bearer AKFN64dfAE45dfsdfs1dfsDFSDfDFFSDF"
        },
        body: JSON.stringify(data)
      })
      if (!response.ok) {
        throw new Error('Server error: ${response.status}');
      }
      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let res = "";
      assistantMessage.value = {
        "role": "assistant",
        "content": res
      }
      messages.value.push(assistantMessage.value)
      let generated_text = "";
      while (true) {
        // console.log(generated_text)
        const { value, done } = await reader.read();
        if (done) break; // Dừng vòng lặp nếu stream kết thúc
        const text = decoder.decode(value); // Giữ lại dữ liệu không đầy đủ
        console.log(text);

        if (text === "[END]") break;
        if (text !== "[START]") {
          generated_text = JSON.parse(text);
          // console.log(generated_text.messages[generated_text.messages.length - 1].content);

          const i = messages.value.length - 1;
          if (i >= 0) {
            messages.value[i].content = generated_text.messages[generated_text.messages.length - 1].content;
          }

          window.scrollBy(0, 100);
        }

      }
    }
    // Khi người dùng đã đăng nhập, tự động lấy thông tin user
    watchEffect(() => {
      userId.value = sessionStorage.getItem("user_id");
      if (userId.value) {
        getInformationOfUser(userId.value);
      }
    });

    return {
      chatHistory,
      messages,
      userInput,
      showLogin,
      showRegister,
      apiToken,
      assistantMessage,
      toggleLogin,
      handleLoginSuccess,
      handleLogout,
      checkLogin,
      convertMarkdown,
      sendMessage,
      userRoleId, // Trả về role_id để kiểm tra
    };
  },
};
</script>
<style>
/* .chat_template>table {} */

.chat_template table {
  border: 1px solid black !important;
  background: none !important;
}

.chat_template td,
.chat_template th {
  border: 1px solid black !important;
}

.chat_template tr:nth-child(odd) {
  background-color: #f7f7f8 !important;
  /* Màu nền cho hàng lẻ */
}

.chat_template tr:nth-child(even) {
  background-color: white !important;
  /* Màu nền cho hàng chẵn */
}
</style>