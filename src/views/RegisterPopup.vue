<template>
    <div class="modal d-block" tabindex="-1">
        <div class="modal-dialog modal-dialog-centered">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">Đăng ký</h5>
                    <button type="button" class="btn-close" @click="$emit('close')"></button>
                </div>
                <div class="modal-body">
                    <input v-model="user.email" class="form-control mb-2" placeholder="Email" />
                    <input v-model="user.password" type="password" class="form-control mb-2" placeholder="Mật khẩu" />
                    <input v-model="user.confirmPassword" type="password" class="form-control mb-2"
                        placeholder="Nhập lại mật khẩu" />
                    <p v-if="errorMessage" class="text-danger">{{ errorMessage }}</p>
                </div>
                <div class="modal-footer">
                    <button class="btn btn-success" @click="register">Đăng ký</button>
                    <button class="btn btn-danger" @click="$emit('close')">Đóng</button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import { ref } from 'vue';
import axios from 'axios';

export default {
    setup(_, { emit }) {
        const user = ref({
            email: "",
            password: "",
            confirmPassword: ""
        });
        const errorMessage = ref("");

        const register = async () => {
            if (user.value.password !== user.value.confirmPassword) {
                errorMessage.value = "Mật khẩu không khớp";
                return;
            }
            try {
                const response = await axios.post("http://localhost:4999/register", user.value);
                const data = response.data
                if (data.status) {
                    emit("close");
                } else {
                    alert(data.message);
                }

            } catch (error) {
                errorMessage.value = error.response?.data?.message || "Có lỗi xảy ra. Vui lòng thử lại.";
            }
        };

        return { user, errorMessage, register };
    }
};
</script>

<style lang="scss" scoped></style>
