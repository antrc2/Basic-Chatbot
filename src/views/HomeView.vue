<template>
  <div class="container-fluid d-flex vh-100">
    <aside class="col-3 bg-light p-3 border-end overflow-auto">
      <h2 class="h6 text-uppercase mb-3">Lịch sử Chat</h2>
      <ul class="list-group small">
        <li
          class="list-group-item list-group-item-action"
          v-for="(chat, index) in chatHistory"
          :key="index"
          @click="loadChat(index)"
        >
          {{ chat.title || `Phiên #${index + 1}` }}
        </li>
      </ul>
    </aside>

    <main class="col d-flex flex-column">
      <header class="d-flex justify-content-end align-items-center gap-2 bg-secondary p-2">
        <RouterLink
          v-if="checkLogin() !== null && userRoleId === 2"
          :to="{ name: 'dashboard' }"
          class="btn btn-danger btn-sm"
        >
          Trang quản trị
        </RouterLink>

        <button v-if="checkLogin() !== null" @click="handleLogout" class="btn btn-success btn-sm">
          Đăng xuất
        </button>

        <button v-if="checkLogin() === null" @click="toggleLogin" class="btn btn-primary btn-sm me-2">
          Đăng nhập
        </button>
      </header>

      <div class="flex-grow-1 p-3 overflow-auto bg-body-tertiary">
        <!-- Mỗi message là một thẻ card, hiển thị đầy đủ role, content và tool (nếu có) -->
        <div v-for="(msg, index) in messages" :key="index" class="mb-3">
          <div class="card shadow-sm chat-card" :class="roleCardClass(msg.role)">
            <div class="card-header d-flex justify-content-between align-items-center py-2">
              <div class="d-flex align-items-center gap-2">
                <span class="badge text-uppercase" :class="roleBadgeClass(msg.role)">
                  {{ (msg.role || 'unknown') }}
                </span>
                <small v-if="msg.name" class="text-muted">{{ msg.name }}</small>
                <small class="text-muted">#{{ index + 1 }}</small>
              </div>
              <small v-if="msg.created_at" class="text-muted">{{ formatTime(msg.created_at) }}</small>
            </div>

            <div class="card-body">
              <!-- Nội dung thường (markdown) -->
              <div v-if="!isToolMessage(msg)" class="chat-markdown" v-html="convertMarkdown(msg.content || '')"></div>

              <!-- Chi tiết tool call theo nhiều chuẩn dữ liệu khác nhau -->
              <div v-else class="tool-block">
                <!-- OpenAI-style: assistant chứa tool_calls -->
                <div v-if="Array.isArray(msg.tool_calls) && msg.tool_calls.length">
                  <h6 class="mb-2">Yêu cầu gọi tool</h6>
                  <div v-for="(tc, i) in msg.tool_calls" :key="i" class="border rounded p-2 mb-2 bg-light-subtle">
                    <div class="d-flex flex-wrap gap-2 align-items-center mb-1">
                      <span class="badge bg-dark-subtle text-dark-emphasis">Tool</span>
                      <code class="small">{{ tc.function?.name || tc.name || tc.type || '—' }}</code>
                    </div>
                    <div>
                      <div class="fw-medium small mb-1">Arguments</div>
                      <pre class="mb-0 small">{{ pretty(safeJson(tc.function?.arguments ?? tc.arguments)) }}</pre>
                    </div>
                  </div>
                </div>

                <!-- Kiểu tuỳ chỉnh: có tool_name/tool_args/tool_response trong cùng 1 message -->
                <div v-if="msg.tool_name || msg.tool_args">
                  <h6 class="mb-2">Yêu cầu gọi tool</h6>
                  <div class="border rounded p-2 mb-2 bg-light-subtle">
                    <div class="d-flex flex-wrap gap-2 align-items-center mb-1">
                      <span class="badge bg-dark-subtle text-dark-emphasis">Tool</span>
                      <code class="small">{{ msg.tool_name || '—' }}</code>
                    </div>
                    <div>
                      <div class="fw-medium small mb-1">Arguments</div>
                      <pre class="mb-0 small">{{ pretty(safeJson(msg.tool_args)) }}</pre>
                    </div>
                  </div>
                </div>

                <!-- Phản hồi từ tool: role === 'tool' hoặc có tool_response -->
                <div v-if="msg.role === 'tool' || msg.tool_response">
                  <h6 class="mb-2">Phản hồi từ tool</h6>
                  <div class="border rounded p-2 bg-body">
                    <div class="d-flex flex-wrap gap-2 align-items-center mb-1">
                      <span class="badge bg-info-subtle text-info-emphasis">Tool</span>
                      <code class="small">{{ msg.tool_name || msg.name || '—' }}</code>
                    </div>
                    <pre class="mb-0 small">{{ typeof msg.tool_response === 'string' ? msg.tool_response : pretty(msg.tool_response ?? msg.content) }}</pre>
                  </div>
                </div>

                <!-- Nếu message còn có content dạng markdown đi kèm -->
                <div v-if="msg.content && msg.role !== 'tool'" class="mt-3">
                  <div class="fw-medium small mb-1">Ghi chú</div>
                  <div class="chat-markdown" v-html="convertMarkdown(msg.content)"></div>
                </div>
              </div>
            </div>

            <div class="card-footer py-2 bg-transparent">
              <details>
                <summary class="small">Xem JSON gốc</summary>
                <pre class="small mb-0">{{ pretty(msg) }}</pre>
              </details>
            </div>
          </div>
        </div>
      </div>

      <footer class="d-flex p-3 bg-light border-top">
        <input
          v-model="userInput"
          @keyup.enter="sendMessage"
          class="form-control me-2"
          placeholder="Nhập tin nhắn..."
        />
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
import DOMPurify from "dompurify";

export default {
  components: { LoginPopup, RegisterPopup },
  setup() {
    const chatHistory = ref([]);
    const messages = ref([]);
    const userInfo = ref(null);
    const userRoleId = ref(null);
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
          userRoleId.value = data.data.role_id;
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
      await getInformationOfUser(id);
    };

    const handleLogout = () => {
      sessionStorage.removeItem("user_id");
      userId.value = null;
      userInfo.value = null;
      userRoleId.value = null;
      alert("Bạn đã đăng xuất thành công!");
    };

    // Markdown -> HTML (đã sanitize để tránh XSS)
    const convertMarkdown = (text) => {
      const html = marked.parse(text || "");
      return DOMPurify.sanitize(html);
    };

    // Phân loại hiển thị tool
    const isToolMessage = (msg) => {
      return (
        msg?.role === "tool" ||
        !!msg?.tool_name ||
        !!msg?.tool_args ||
        (Array.isArray(msg?.tool_calls) && msg.tool_calls.length > 0)
      );
    };

    // Pretty JSON helpers
    const pretty = (obj) => {
      try {
        return JSON.stringify(obj, null, 2);
      } catch (e) {
        return String(obj);
      }
    };
    // Parse nếu arguments là chuỗi JSON, nếu không trả lại nguyên văn
    const safeJson = (maybeJson) => {
      if (typeof maybeJson !== "string") return maybeJson;
      try {
        return JSON.parse(maybeJson);
      } catch {
        return maybeJson; // không phải JSON, giữ nguyên
      }
    };

    // Role -> class cho card & badge
    const roleCardClass = (role) => {
      switch (role) {
        case "user":
          return "border-primary";
        case "assistant":
          return "border-success";
        case "system":
          return "border-warning";
        case "tool":
          return "border-info";
        default:
          return "border-secondary";
      }
    };
    const roleBadgeClass = (role) => {
      switch (role) {
        case "user":
          return "bg-primary";
        case "assistant":
          return "bg-success";
        case "system":
          return "bg-warning text-dark";
        case "tool":
          return "bg-info text-dark";
        default:
          return "bg-secondary";
      }
    };

    const formatTime = (ts) => {
      try {
        const d = new Date(ts);
        if (Number.isNaN(d.getTime())) return ts;
        return d.toLocaleString();
      } catch {
        return ts;
      }
    };

    // Gửi tin nhắn + stream kết quả
    const sendMessage = async () => {
      if (!userInput.value?.trim()) return;

      messages.value.push({ role: "user", content: userInput.value });
      const data = { messages: messages.value };
      userInput.value = "";

      const response = await fetch("http://localhost:5000/assistant/chat", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: "Bearer AKFN64dfAE45dfsdfs1dfsDFSDfDFFSDF",
        },
        body: JSON.stringify(data),
      });

      if (!response.ok) {
        throw new Error(`Server error: ${response.status}`);
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder();

      // Tạo khung message trống để stream đổ vào
      assistantMessage.value = { role: "assistant", content: "" };
      messages.value.push(assistantMessage.value);

      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        const chunk = decoder.decode(value);
        if (chunk === "[END]") break;
        if (chunk === "[START]") continue;

        // Backend của bạn trả về JSON cho toàn bộ phiên chat
        // { messages: [...] }
        try {
          const parsed = JSON.parse(chunk);
          const latest = parsed.messages?.[parsed.messages.length - 1];
          if (latest) {
            // Nếu backend stream cả tool messages, ta có thể push đúng loại thay vì luôn cập nhật assistant cuối cùng
            // Ở đây để tương thích tối đa, nếu latest.role === 'tool' thì thay card cuối nếu cùng role, ngược lại tạo mới
            const lastIdx = messages.value.length - 1;
            const last = messages.value[lastIdx];
            const shouldReplace = last && last.role === latest.role && (latest.role === 'assistant' || latest.role === 'tool');

            if (shouldReplace) {
              messages.value[lastIdx] = latest;
            } else {
              messages.value.push(latest);
            }
          } else {
            // fallback: chỉ cập nhật nội dung assistant đang stream
            const i = messages.value.length - 1;
            if (i >= 0) messages.value[i].content = parsed.messages?.[i]?.content ?? messages.value[i].content;
          }
        } catch (e) {
          // Nếu không phải JSON hoàn chỉnh, bỏ qua chunk (tuỳ backend có thể gửi keep-alive)
          console.debug("Non-JSON chunk:", chunk);
        }

        // Auto scroll nhẹ
        window.scrollBy({ top: 120, behavior: "smooth" });
      }
    };

    // Khi người dùng đã đăng nhập, tự động lấy thông tin user
    watchEffect(() => {
      userId.value = sessionStorage.getItem("user_id");
      if (userId.value) {
        getInformationOfUser(userId.value);
      }
    });

    // Demo: nạp mảng mẫu nếu cần (bạn có thể xoá phần này)
    onMounted(() => {
      // messages.value = [
      //   { role: 'user', content: 'giới thiệu về danh mục con 1' },
      //   { role: 'assistant', content: 'Chào bạn, 13Bee đây ạ!\n\nHiện tại...' },
      //   { role: 'user', content: 'có' }
      // ]
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
      userRoleId,
      isToolMessage,
      pretty,
      safeJson,
      roleCardClass,
      roleBadgeClass,
      formatTime,
      loadChat: (i) => console.log("load chat", i),
    };
  },
};
</script>

<style>
.chat-markdown table {
  border: 1px solid var(--bs-border-color) !important;
  background: none !important;
}
.chat-markdown td,
.chat-markdown th {
  border: 1px solid var(--bs-border-color) !important;
  padding: .25rem .5rem;
}
.chat-markdown tr:nth-child(odd) {
  background-color: #f7f7f8 !important;
}
.chat-markdown tr:nth-child(even) {
  background-color: #fff !important;
}

.chat-card pre {
  background: #0d1117;
  color: #c9d1d9;
  border-radius: .5rem;
  padding: .5rem .75rem;
  overflow: auto;
}

.tool-block h6 { font-weight: 600; }
</style>
