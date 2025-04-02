<template>
    <div>
        <h1>Đây là trang quản trị</h1>
    </div>
</template>

<script setup>
import { ref } from 'vue';

const user_id = ref()
const user_info = ref()
const checkSomeThings = async () => {
    user_id.value = sessionStorage.getItem('user_id');
    if (user_id.value == null) {
        window.location.href = '/'
    } else {
        const response = await fetch(`http://localhost:4999/user/${user_id.value}`)
        const data = await response.json()
        user_info.value = data.data
        if (user_info.value.role_id != 1) {
            window.location.href = '/'
        } else {

        }

    }


}
checkSomeThings()
</script>

<style lang="scss" scoped></style>