<template>
  <a-config-provider :locale="zhCN">
    <div id="app">
<!--      <BasicLayout />-->
      <router-view />
    </div>
  </a-config-provider>
</template>

<script setup lang="ts">
import { healthUsingGet } from '@/api/mainController.ts'
import { useLoginUserStore } from '@/stores/useLoginUserStore.ts'
import zhCN from 'ant-design-vue/es/locale/zh_CN'
const loginUserStore = useLoginUserStore()
loginUserStore.fetchLoginUser()
// 已经改为在权限校验文件中获取
// const loginUserStore = useLoginUserStore()
// loginUserStore.fetchLoginUser()

healthUsingGet().then((res) => {
  console.log(res)
})
</script>

<style scoped></style>
