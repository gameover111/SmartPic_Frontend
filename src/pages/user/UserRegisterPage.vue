<script lang="ts" setup>
import { reactive } from 'vue'
import { userRegisterUsingPost } from '@/api/userController.ts'
import { message } from 'ant-design-vue'
import router from '@/router'
import { Sparkles, User, Lock, ShieldCheck, ArrowRight } from 'lucide-vue-next'
import Button from '@/components/ui/Button.vue'
import Input from '@/components/ui/Input.vue'
import Label from '@/components/ui/Label.vue'

// 用于接受表单输入的值
const formState = reactive<API.UserRegisterRequest>({
  userAccount: '',
  userPassword: '',
  checkPassword: '',
})

/**
 * 提交表单
 */
const handleSubmit = async () => {
  // 校验两次输入的密码是否一致
  if (formState.userPassword !== formState.checkPassword) {
    message.error('两次输入的密码不一致')
    return
  }
  const res = await userRegisterUsingPost(formState)
  // 注册成功，跳转到登录页面
  if (res.data.code === 0 && res.data.data) {
    message.success('注册成功')
    router.push({
      path: '/user/login',
      replace: true,
    })
  } else {
    message.error('注册失败，' + res.data.message)
  }
}
</script>

<template>
  <div class="min-h-screen grid lg:grid-cols-2">
    <!-- 左侧装饰区域 - 暗灰色高端风格 -->
    <div
      class="relative hidden lg:flex flex-col justify-between bg-gradient-to-br from-[#0f0f1a] via-[#1a1a2e] to-[#12121f] p-12 text-gray-200 overflow-hidden"
    >
      <div class="relative z-20">
        <div class="flex items-center gap-3 text-lg font-semibold tracking-wide">
          <div
            class="size-9 rounded-xl bg-white/[0.08] backdrop-blur-sm flex items-center justify-center border border-white/[0.06]"
          >
            <Sparkles class="size-4 text-violet-400" />
          </div>
          <span class="bg-gradient-to-r from-white to-gray-400 bg-clip-text text-transparent">智图云 SmartPic</span>
        </div>
      </div>

      <div class="relative z-20 flex flex-col items-center justify-center h-[400px]">
        <h2 class="text-4xl font-bold mb-4 bg-gradient-to-r from-white to-gray-300 bg-clip-text text-transparent">加入我们</h2>
        <p class="text-gray-400 text-lg text-center max-w-xs leading-relaxed">
          企业级智能协同云图库，开启您的智能图片管理之旅
        </p>
      </div>

      <div class="relative z-20 flex items-center gap-8 text-sm text-gray-500">
        <a href="#" class="hover:text-gray-300 transition-colors duration-300">隐私政策</a>
        <a href="#" class="hover:text-gray-300 transition-colors duration-300">服务条款</a>
        <a href="#" class="hover:text-gray-300 transition-colors duration-300">联系我们</a>
      </div>

      <!-- 装饰元素 -->
      <div class="absolute inset-0 bg-grid-white/[0.03] bg-[size:32px_32px]" />
      <div
        class="absolute top-1/4 right-1/4 size-80 bg-violet-500/[0.07] rounded-full blur-[100px]"
      />
      <div
        class="absolute bottom-1/4 left-1/4 size-96 bg-blue-500/[0.05] rounded-full blur-[120px]"
      />
    </div>

    <!-- 右侧注册表单 - 高端丝滑风格 -->
    <div class="relative flex items-center justify-center p-8 overflow-hidden bg-[#f5f3ff]/40">
      <!-- 装饰背景 -->
      <div class="absolute top-[-20%] right-[-10%] size-[500px] bg-violet-200/30 rounded-full blur-[100px]" />
      <div class="absolute bottom-[-15%] left-[-5%] size-[400px] bg-indigo-200/25 rounded-full blur-[80px]" />

      <div class="relative w-full max-w-[440px] z-10">
        <!-- 表单卡片 -->
        <div class="bg-white/70 backdrop-blur-xl rounded-2xl shadow-[0_8px_32px_rgba(0,0,0,0.06)] border border-white/60 p-10">
          <!-- 移动端 Logo -->
          <div class="lg:hidden flex items-center justify-center gap-2 text-lg font-semibold mb-12">
            <div class="size-8 rounded-lg bg-primary/10 flex items-center justify-center">
              <Sparkles class="size-4 text-primary" />
            </div>
            <span>智图云 SmartPic</span>
          </div>

          <!-- 标题 -->
          <div class="mb-8">
            <h1 class="text-2xl font-bold tracking-tight mb-2 text-gray-900">创建账号</h1>
            <p class="text-gray-400 text-sm">请填写注册信息，开始智能图片管理之旅</p>
          </div>

          <!-- 注册表单 -->
          <form @submit.prevent="handleSubmit" class="space-y-5">
            <div class="space-y-1.5">
              <Label for="regAccount" class="text-xs font-semibold text-gray-500 uppercase tracking-wider">账号</Label>
              <div class="relative group">
                <div class="absolute left-3.5 top-1/2 -translate-y-1/2 text-gray-300 group-focus-within:text-violet-500 transition-colors duration-300">
                  <User class="size-[18px]" />
                </div>
                <Input
                  id="regAccount"
                  v-model="formState.userAccount"
                  placeholder="请输入账号"
                  required
                  class="h-12 pl-11 bg-gray-50/80 border-gray-100 rounded-xl text-gray-800 placeholder:text-gray-300 focus:bg-white focus:border-violet-400 focus:ring-2 focus:ring-violet-100 transition-all duration-300"
                />
              </div>
            </div>

            <div class="space-y-1.5">
              <Label for="regPassword" class="text-xs font-semibold text-gray-500 uppercase tracking-wider">密码</Label>
              <div class="relative group">
                <div class="absolute left-3.5 top-1/2 -translate-y-1/2 text-gray-300 group-focus-within:text-violet-500 transition-colors duration-300">
                  <Lock class="size-[18px]" />
                </div>
                <Input
                  id="regPassword"
                  type="password"
                  v-model="formState.userPassword"
                  placeholder="请输入密码（至少8位）"
                  required
                  class="h-12 pl-11 bg-gray-50/80 border-gray-100 rounded-xl text-gray-800 placeholder:text-gray-300 focus:bg-white focus:border-violet-400 focus:ring-2 focus:ring-violet-100 transition-all duration-300"
                />
              </div>
            </div>

            <div class="space-y-1.5">
              <Label for="regCheckPassword" class="text-xs font-semibold text-gray-500 uppercase tracking-wider">确认密码</Label>
              <div class="relative group">
                <div class="absolute left-3.5 top-1/2 -translate-y-1/2 text-gray-300 group-focus-within:text-violet-500 transition-colors duration-300">
                  <ShieldCheck class="size-[18px]" />
                </div>
                <Input
                  id="regCheckPassword"
                  type="password"
                  v-model="formState.checkPassword"
                  placeholder="请再次输入密码"
                  required
                  class="h-12 pl-11 bg-gray-50/80 border-gray-100 rounded-xl text-gray-800 placeholder:text-gray-300 focus:bg-white focus:border-violet-400 focus:ring-2 focus:ring-violet-100 transition-all duration-300"
                />
              </div>
            </div>

            <Button type="submit" class="w-full h-12 text-base font-semibold rounded-xl bg-gradient-to-r from-violet-600 to-indigo-600 hover:from-violet-500 hover:to-indigo-500 shadow-lg shadow-violet-500/25 hover:shadow-violet-500/40 hover:-translate-y-0.5 active:translate-y-0 transition-all duration-300" size="lg">
              <span class="flex items-center gap-2">
                注 册
                <ArrowRight class="size-4" />
              </span>
            </Button>
          </form>
        </div>

        <!-- 登录链接 -->
        <div class="text-center text-sm text-gray-400 mt-6">
          已有账号？
          <RouterLink to="/user/login" class="text-violet-500 hover:text-violet-600 font-medium transition-colors duration-300">
            去登录
          </RouterLink>
        </div>
      </div>
    </div>
  </div>
</template>
