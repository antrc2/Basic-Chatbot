<template>
  <div class="modal d-block" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title">Đăng nhập</h5>
          <button type="button" class="btn-close" @click="$emit('close')"></button>
        </div>
        <form action="" @submit.prevent="login" method="POST">
          <div class="modal-body">
            <input v-model="user.email" name="email" class="form-control mb-2" placeholder="Email" />
            <input v-model="user.password" name="password" type="password" class="form-control mb-2"
              placeholder="Mật khẩu" />
          </div>
          <div class="modal-footer">
            <button class="btn btn-primary">Đăng nhập</button>
            <button class="btn btn-danger" @click="$emit('close')">Đóng</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script>
import { ref } from "vue";

export default {
  setup(_, { emit }) {
    const user = ref({
      email: "",
      password: ""
    });
    const login = async () => {

      try {
        const response = await fetch("http://localhost:4999/login", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify(user.value)
        });
        console.log(await response)
        if (!response.ok) {
          throw new Error("Đăng nhập thất bại");
        }

        const data = await response.json();
        console.log(data)
        if (data.status) {

          sessionStorage.setItem("user_id", data.id);
          emit("login-success", data.id);
          emit("close");
          alert(data.message)
        } else {
          alert(data.message)
        }

      } catch (error) {
        alert(error.message);
      }
    };

    return { user, login };
  }
};
</script>

<style lang="scss" scoped></style>
