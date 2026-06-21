<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import { Mail, Sparkles, User, Lock, ArrowRight } from 'lucide-vue-next'
import { userLoginUsingPost } from '@/api/userController'
import { useLoginUserStore } from '@/stores/useLoginUserStore'
import { message } from 'ant-design-vue'
import Button from '@/components/ui/Button.vue'
import Input from '@/components/ui/Input.vue'
import Label from '@/components/ui/Label.vue'
import EyeBall from '@/components/ui/EyeBall.vue'
import Pupil from '@/components/ui/Pupil.vue'

const REMEMBER_KEY = 'smartpic_remember'
const THIRTY_DAYS = 30 * 24 * 60 * 60 * 1000

const router = useRouter()
const loginUserStore = useLoginUserStore()

// 初始化时立即从 localStorage 恢复（不放在 onMounted，避免 Input 组件不响应）
let _initialAccount = ''
let _initialRememberMe = false
try {
  const saved = JSON.parse(localStorage.getItem(REMEMBER_KEY) || '{}')
  if (saved.account && saved.timestamp && (Date.now() - saved.timestamp < THIRTY_DAYS)) {
    _initialAccount = saved.account
    _initialRememberMe = true
  } else {
    localStorage.removeItem(REMEMBER_KEY)
  }
} catch {
  localStorage.removeItem(REMEMBER_KEY)
}

const showPassword = ref(false)
const userAccount = ref(_initialAccount)
const userPassword = ref('')
const error = ref('')
const isLoading = ref(false)

const mouseX = ref(0)
const mouseY = ref(0)

const isPurpleBlinking = ref(false)
const isBlackBlinking = ref(false)
const isTyping = ref(false)
const isLookingAtEachOther = ref(false)
const isPurplePeeking = ref(false)
const rememberMe = ref(_initialRememberMe)

const purpleRef = ref<HTMLDivElement | null>(null)
const blackRef = ref<HTMLDivElement | null>(null)
const yellowRef = ref<HTMLDivElement | null>(null)
const orangeRef = ref<HTMLDivElement | null>(null)

// 全局鼠标追踪
function handleMouseMove(e: MouseEvent) {
  mouseX.value = e.clientX
  mouseY.value = e.clientY
}

onMounted(() => {
  window.addEventListener('mousemove', handleMouseMove)

  // 紫色角色眨眼
  schedulePurpleBlink()
  // 黑色角色眨眼
  scheduleBlackBlink()
})

onUnmounted(() => {
  window.removeEventListener('mousemove', handleMouseMove)
  if (_purpleBlinkTimer) clearTimeout(_purpleBlinkTimer)
  if (_blackBlinkTimer) clearTimeout(_blackBlinkTimer)
  if (peekTimeout) clearTimeout(peekTimeout)
})

// 紫色角色随机眨眼
let _purpleBlinkTimer: ReturnType<typeof setTimeout> | null = null
function schedulePurpleBlink() {
  const interval = Math.random() * 4000 + 3000
  _purpleBlinkTimer = setTimeout(() => {
    isPurpleBlinking.value = true
    setTimeout(() => {
      isPurpleBlinking.value = false
      schedulePurpleBlink()
    }, 150)
  }, interval)
}

// 黑色角色随机眨眼
let _blackBlinkTimer: ReturnType<typeof setTimeout> | null = null
function scheduleBlackBlink() {
  const interval = Math.random() * 4000 + 3000
  _blackBlinkTimer = setTimeout(() => {
    isBlackBlinking.value = true
    setTimeout(() => {
      isBlackBlinking.value = false
      scheduleBlackBlink()
    }, 150)
  }, interval)
}

// 输入时角色互相看
watch(isTyping, (val) => {
  if (val) {
    isLookingAtEachOther.value = true
    setTimeout(() => {
      isLookingAtEachOther.value = false
    }, 800)
  } else {
    isLookingAtEachOther.value = false
  }
})

// 紫色角色偷看密码
let peekTimeout: ReturnType<typeof setTimeout> | null = null
watch([userPassword, showPassword], () => {
  if (userPassword.value.length > 0 && showPassword.value) {
    schedulePeek()
  } else {
    isPurplePeeking.value = false
    if (peekTimeout) clearTimeout(peekTimeout)
  }
})

function schedulePeek() {
  const interval = Math.random() * 3000 + 2000
  peekTimeout = setTimeout(() => {
    isPurplePeeking.value = true
    setTimeout(() => {
      isPurplePeeking.value = false
      if (userPassword.value.length > 0 && showPassword.value) {
        schedulePeek()
      }
    }, 800)
  }, interval)
}

// 计算角色位置
function calculatePosition(el: HTMLDivElement | null) {
  if (!el) return { faceX: 0, faceY: 0, bodySkew: 0 }

  const rect = el.getBoundingClientRect()
  const centerX = rect.left + rect.width / 2
  const centerY = rect.top + rect.height / 3

  const deltaX = mouseX.value - centerX
  const deltaY = mouseY.value - centerY

  const faceX = Math.max(-15, Math.min(15, deltaX / 20))
  const faceY = Math.max(-10, Math.min(10, deltaY / 30))
  const bodySkew = Math.max(-6, Math.min(6, -deltaX / 120))

  return { faceX, faceY, bodySkew }
}

// 登录提交
const handleSubmit = async () => {
  error.value = ''
  isLoading.value = true

  try {
    const res = await userLoginUsingPost({
      userAccount: userAccount.value,
      userPassword: userPassword.value,
    })

    if (res.data.code === 0 && res.data.data) {
      // 处理「记住我」
      if (rememberMe.value) {
        localStorage.setItem(REMEMBER_KEY, JSON.stringify({
          account: userAccount.value,
          timestamp: Date.now(),
        }))
      } else {
        localStorage.removeItem(REMEMBER_KEY)
      }

      await loginUserStore.fetchLoginUser()
      message.success('登录成功')
      router.push({ path: '/', replace: true })
    } else {
      error.value = '登录失败，' + (res.data.message || '请检查账号密码')
      message.error('登录失败')
    }
  } catch {
    error.value = '登录失败，请检查账号密码'
  } finally {
    isLoading.value = false
  }
}
</script>

<template>
  <div class="min-h-screen grid lg:grid-cols-2">
    <!-- 左侧动画区域 - 暗灰色高端风格 -->
    <div class="relative hidden lg:flex flex-col justify-between bg-gradient-to-br from-[#0f0f1a] via-[#1a1a2e] to-[#12121f] p-12 text-gray-200 overflow-hidden">
      <div class="relative z-20">
        <div class="flex items-center gap-3 text-lg font-semibold tracking-wide">
          <div class="size-9 rounded-xl bg-white/[0.08] backdrop-blur-sm flex items-center justify-center border border-white/[0.06]">
            <Sparkles class="size-4 text-violet-400" />
          </div>
          <span class="bg-gradient-to-r from-white to-gray-400 bg-clip-text text-transparent">智图云 SmartPic</span>
        </div>
      </div>

      <div class="relative z-20 flex items-end justify-center h-[500px]">
        <!-- 卡通角色 -->
        <div class="relative" style="width: 550px; height: 400px">
          <!-- 紫色高矩形角色 - 后层 -->
          <div
            ref="purpleRef"
            class="absolute bottom-0 transition-all duration-700 ease-in-out"
            :style="{
              left: '70px',
              width: '180px',
              height: (isTyping || (userPassword.length > 0 && !showPassword)) ? '440px' : '400px',
              backgroundColor: '#7C5CFC',
              borderRadius: '12px 12px 0 0',
              zIndex: 1,
              boxShadow: '0 0 60px rgba(124, 92, 252, 0.3)',
              transform: (userPassword.length > 0 && showPassword)
                ? 'skewX(0deg)'
                : (isTyping || (userPassword.length > 0 && !showPassword))
                  ? `skewX(${(calculatePosition(purpleRef).bodySkew || 0) - 12}deg) translateX(40px)`
                  : `skewX(${calculatePosition(purpleRef).bodySkew || 0}deg)`,
              transformOrigin: 'bottom center',
            }"
          >
            <!-- 眼睛 -->
            <div
              class="absolute flex gap-8 transition-all duration-700 ease-in-out"
              :style="{
                left: (userPassword.length > 0 && showPassword) ? '20px' : isLookingAtEachOther ? '55px' : `${45 + calculatePosition(purpleRef).faceX}px`,
                top: (userPassword.length > 0 && showPassword) ? '35px' : isLookingAtEachOther ? '65px' : `${40 + calculatePosition(purpleRef).faceY}px`,
              }"
            >
              <EyeBall
                :size="18"
                :pupilSize="7"
                :maxDistance="5"
                eyeColor="white"
                pupilColor="#2D2D2D"
                :isBlinking="isPurpleBlinking"
                :forceLookX="(userPassword.length > 0 && showPassword) ? (isPurplePeeking ? 4 : -4) : isLookingAtEachOther ? 3 : undefined"
                :forceLookY="(userPassword.length > 0 && showPassword) ? (isPurplePeeking ? 5 : -4) : isLookingAtEachOther ? 4 : undefined"
              />
              <EyeBall
                :size="18"
                :pupilSize="7"
                :maxDistance="5"
                eyeColor="white"
                pupilColor="#2D2D2D"
                :isBlinking="isPurpleBlinking"
                :forceLookX="(userPassword.length > 0 && showPassword) ? (isPurplePeeking ? 4 : -4) : isLookingAtEachOther ? 3 : undefined"
                :forceLookY="(userPassword.length > 0 && showPassword) ? (isPurplePeeking ? 5 : -4) : isLookingAtEachOther ? 4 : undefined"
              />
            </div>
          </div>

          <!-- 黑色高矩形角色 - 中层 -->
          <div
            ref="blackRef"
            class="absolute bottom-0 transition-all duration-700 ease-in-out"
            :style="{
              left: '240px',
              width: '120px',
              height: '310px',
              backgroundColor: '#3a3a4a',
              borderRadius: '8px 8px 0 0',
              zIndex: 2,
              boxShadow: '0 0 40px rgba(58, 58, 74, 0.5)',
              transform: (userPassword.length > 0 && showPassword)
                ? 'skewX(0deg)'
                : isLookingAtEachOther
                  ? `skewX(${(calculatePosition(blackRef).bodySkew || 0) * 1.5 + 10}deg) translateX(20px)`
                  : (isTyping || (userPassword.length > 0 && !showPassword))
                    ? `skewX(${(calculatePosition(blackRef).bodySkew || 0) * 1.5}deg)`
                    : `skewX(${calculatePosition(blackRef).bodySkew || 0}deg)`,
              transformOrigin: 'bottom center',
            }"
          >
            <!-- 眼睛 -->
            <div
              class="absolute flex gap-6 transition-all duration-700 ease-in-out"
              :style="{
                left: (userPassword.length > 0 && showPassword) ? '10px' : isLookingAtEachOther ? '32px' : `${26 + calculatePosition(blackRef).faceX}px`,
                top: (userPassword.length > 0 && showPassword) ? '28px' : isLookingAtEachOther ? '12px' : `${32 + calculatePosition(blackRef).faceY}px`,
              }"
            >
              <EyeBall
                :size="16"
                :pupilSize="6"
                :maxDistance="4"
                eyeColor="white"
                pupilColor="#2D2D2D"
                :isBlinking="isBlackBlinking"
                :forceLookX="(userPassword.length > 0 && showPassword) ? -4 : isLookingAtEachOther ? 0 : undefined"
                :forceLookY="(userPassword.length > 0 && showPassword) ? -4 : isLookingAtEachOther ? -4 : undefined"
              />
              <EyeBall
                :size="16"
                :pupilSize="6"
                :maxDistance="4"
                eyeColor="white"
                pupilColor="#2D2D2D"
                :isBlinking="isBlackBlinking"
                :forceLookX="(userPassword.length > 0 && showPassword) ? -4 : isLookingAtEachOther ? 0 : undefined"
                :forceLookY="(userPassword.length > 0 && showPassword) ? -4 : isLookingAtEachOther ? -4 : undefined"
              />
            </div>
          </div>

          <!-- 橙色半圆角色 - 前左 -->
          <div
            ref="orangeRef"
            class="absolute bottom-0 transition-all duration-700 ease-in-out"
            :style="{
              left: '0px',
              width: '240px',
              height: '200px',
              zIndex: 3,
              backgroundColor: '#FF8C5A',
              borderRadius: '120px 120px 0 0',
              boxShadow: '0 0 50px rgba(255, 140, 90, 0.25)',
              transform: (userPassword.length > 0 && showPassword) ? 'skewX(0deg)' : `skewX(${calculatePosition(orangeRef).bodySkew || 0}deg)`,
              transformOrigin: 'bottom center',
            }"
          >
            <!-- 眼睛 - 只有瞳孔 -->
            <div
              class="absolute flex gap-8 transition-all duration-200 ease-out"
              :style="{
                left: (userPassword.length > 0 && showPassword) ? '50px' : `${82 + (calculatePosition(orangeRef).faceX || 0)}px`,
                top: (userPassword.length > 0 && showPassword) ? '85px' : `${90 + (calculatePosition(orangeRef).faceY || 0)}px`,
              }"
            >
              <Pupil :size="12" :maxDistance="5" pupilColor="#2D2D2D" :forceLookX="(userPassword.length > 0 && showPassword) ? -5 : undefined" :forceLookY="(userPassword.length > 0 && showPassword) ? -4 : undefined" />
              <Pupil :size="12" :maxDistance="5" pupilColor="#2D2D2D" :forceLookX="(userPassword.length > 0 && showPassword) ? -5 : undefined" :forceLookY="(userPassword.length > 0 && showPassword) ? -4 : undefined" />
            </div>
          </div>

          <!-- 黄色高矩形角色 - 前右 -->
          <div
            ref="yellowRef"
            class="absolute bottom-0 transition-all duration-700 ease-in-out"
            :style="{
              left: '310px',
              width: '140px',
              height: '230px',
              backgroundColor: '#F0D94E',
              borderRadius: '70px 70px 0 0',
              zIndex: 4,
              boxShadow: '0 0 50px rgba(240, 217, 78, 0.2)',
              transform: (userPassword.length > 0 && showPassword) ? 'skewX(0deg)' : `skewX(${calculatePosition(yellowRef).bodySkew || 0}deg)`,
              transformOrigin: 'bottom center',
            }"
          >
            <!-- 眼睛 - 只有瞳孔 -->
            <div
              class="absolute flex gap-6 transition-all duration-200 ease-out"
              :style="{
                left: (userPassword.length > 0 && showPassword) ? '20px' : `${52 + (calculatePosition(yellowRef).faceX || 0)}px`,
                top: (userPassword.length > 0 && showPassword) ? '35px' : `${40 + (calculatePosition(yellowRef).faceY || 0)}px`,
              }"
            >
              <Pupil :size="12" :maxDistance="5" pupilColor="#2D2D2D" :forceLookX="(userPassword.length > 0 && showPassword) ? -5 : undefined" :forceLookY="(userPassword.length > 0 && showPassword) ? -4 : undefined" />
              <Pupil :size="12" :maxDistance="5" pupilColor="#2D2D2D" :forceLookX="(userPassword.length > 0 && showPassword) ? -5 : undefined" :forceLookY="(userPassword.length > 0 && showPassword) ? -4 : undefined" />
            </div>
            <!-- 嘴巴横线 -->
            <div
              class="absolute w-20 h-1 bg-[#2D2D2D] rounded-full transition-all duration-200 ease-out"
              :style="{
                left: (userPassword.length > 0 && showPassword) ? '10px' : `${40 + (calculatePosition(yellowRef).faceX || 0)}px`,
                top: (userPassword.length > 0 && showPassword) ? '88px' : `${88 + (calculatePosition(yellowRef).faceY || 0)}px`,
              }"
            />
          </div>
        </div>
      </div>

      <div class="relative z-20 flex items-center gap-8 text-sm text-gray-500">
        <a href="#" class="hover:text-gray-300 transition-colors duration-300">隐私政策</a>
        <a href="#" class="hover:text-gray-300 transition-colors duration-300">服务条款</a>
        <a href="#" class="hover:text-gray-300 transition-colors duration-300">联系我们</a>
      </div>

      <!-- 装饰元素 -->
      <div class="absolute inset-0 bg-grid-white/[0.03] bg-[size:32px_32px]" />
      <div class="absolute top-1/4 right-1/4 size-80 bg-violet-500/[0.07] rounded-full blur-[100px]" />
      <div class="absolute bottom-1/4 left-1/4 size-96 bg-blue-500/[0.05] rounded-full blur-[120px]" />
      <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 size-64 bg-orange-500/[0.04] rounded-full blur-[80px]" />
    </div>

    <!-- 右侧登录表单 - 高端丝滑风格 -->
    <div class="relative flex items-center justify-center p-8 overflow-hidden bg-[#f5f3ff]/40">
      <!-- 装饰背景 -->
      <div class="absolute top-[-20%] right-[-10%] size-[500px] bg-violet-200/30 rounded-full blur-[100px]" />
      <div class="absolute bottom-[-15%] left-[-5%] size-[400px] bg-indigo-200/25 rounded-full blur-[80px]" />
      <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 size-[300px] bg-pink-100/20 rounded-full blur-[60px]" />

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
            <h1 class="text-2xl font-bold tracking-tight mb-2 text-gray-900">欢迎回来</h1>
            <p class="text-gray-400 text-sm">请输入您的账号信息，开始智能图片管理之旅</p>
          </div>

          <!-- 登录表单 -->
          <form @submit.prevent="handleSubmit" class="space-y-5">
            <!-- 账号输入 -->
            <div class="space-y-1.5">
              <Label for="userAccount" class="text-xs font-semibold text-gray-500 uppercase tracking-wider">账号</Label>
              <div class="relative group">
                <div class="absolute left-3.5 top-1/2 -translate-y-1/2 text-gray-300 group-focus-within:text-violet-500 transition-colors duration-300">
                  <User class="size-[18px]" />
                </div>
                <Input
                  id="userAccount"
                  type="text"
                  placeholder="请输入账号"
                  v-model="userAccount"
                  required
                  class="h-12 pl-11 bg-gray-50/80 border-gray-100 rounded-xl text-gray-800 placeholder:text-gray-300 focus:bg-white focus:border-violet-400 focus:ring-2 focus:ring-violet-100 transition-all duration-300"
                  @focus="isTyping = true"
                  @blur="isTyping = false"
                />
              </div>
            </div>

            <!-- 密码输入 -->
            <div class="space-y-1.5">
              <Label for="userPassword" class="text-xs font-semibold text-gray-500 uppercase tracking-wider">密码</Label>
              <div class="relative group">
                <div class="absolute left-3.5 top-1/2 -translate-y-1/2 text-gray-300 group-focus-within:text-violet-500 transition-colors duration-300">
                  <Lock class="size-[18px]" />
                </div>
                <Input
                  id="userPassword"
                  :type="showPassword ? 'text' : 'password'"
                  placeholder="请输入密码"
                  v-model="userPassword"
                  required
                  class="h-12 pl-11 pr-11 bg-gray-50/80 border-gray-100 rounded-xl text-gray-800 placeholder:text-gray-300 focus:bg-white focus:border-violet-400 focus:ring-2 focus:ring-violet-100 transition-all duration-300"
                />
                <button
                  type="button"
                  @click="showPassword = !showPassword"
                  class="absolute right-3.5 top-1/2 -translate-y-1/2 p-1 text-gray-300 hover:text-violet-500 transition-colors duration-300"
                >
                  <!-- 自定义眼睛图标 - 可见状态 -->
                  <svg v-if="showPassword" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z" />
                    <circle cx="12" cy="12" r="3" fill="currentColor" stroke="none" />
                  </svg>
                  <!-- 自定义眼睛图标 - 隐藏状态（带斜线） -->
                  <svg v-else width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M17.94 17.94A10.07 10.07 0 0 1 12 20c-7 0-11-8-11-8a18.45 18.45 0 0 1 5.06-5.94" />
                    <path d="M9.9 4.24A9.12 9.12 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.16 3.19" />
                    <path d="M14.12 14.12a3 3 0 1 1-4.24-4.24" />
                    <line x1="1" y1="1" x2="23" y2="23" />
                  </svg>
                </button>
              </div>
            </div>

            <!-- 记住我 / 忘记密码 -->
            <div class="flex items-center justify-between pt-1">
              <label class="flex items-center gap-2.5 cursor-pointer group select-none">
                <button
                  type="button"
                  role="checkbox"
                  :aria-checked="rememberMe"
                  @click.stop="rememberMe = !rememberMe"
                  class="relative h-5 w-5 rounded-md border-2 transition-all duration-200 focus:outline-none focus:ring-2 focus:ring-violet-200 focus:ring-offset-1"
                  :class="rememberMe
                    ? 'bg-violet-500 border-violet-500'
                    : 'bg-white border-gray-200 group-hover:border-gray-300'"
                >
                  <svg
                    v-if="rememberMe"
                    class="absolute inset-0 m-auto size-3 text-white"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="3"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  >
                    <polyline points="20 6 9 17 4 12" />
                  </svg>
                </button>
                <span class="text-sm text-gray-400 group-hover:text-gray-600 transition-colors duration-200">记住我 30 天</span>
              </label>
              <a href="#" class="text-sm text-gray-400 hover:text-violet-500 transition-colors duration-300 font-medium">
                忘记密码？
              </a>
            </div>

            <!-- 错误提示 -->
            <div v-if="error" class="p-3 text-sm text-red-500 bg-red-50/80 border border-red-100 rounded-xl">
              {{ error }}
            </div>

            <!-- 登录按钮 -->
            <Button
              type="submit"
              class="w-full h-12 text-base font-semibold rounded-xl bg-gradient-to-r from-violet-600 to-indigo-600 hover:from-violet-500 hover:to-indigo-500 shadow-lg shadow-violet-500/25 hover:shadow-violet-500/40 hover:-translate-y-0.5 active:translate-y-0 transition-all duration-300"
              size="lg"
              :disabled="isLoading"
            >
              <span v-if="isLoading" class="flex items-center gap-2">
                <svg class="animate-spin size-4" viewBox="0 0 24 24" fill="none">
                  <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" />
                  <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z" />
                </svg>
                登录中...
              </span>
              <span v-else class="flex items-center gap-2">
                登 录
                <ArrowRight class="size-4" />
              </span>
            </Button>
          </form>

          <!-- 分隔线 -->
          <div class="relative my-7">
            <div class="absolute inset-0 flex items-center">
              <div class="w-full border-t border-gray-100"></div>
            </div>
            <div class="relative flex justify-center text-xs">
              <span class="bg-transparent px-4 text-gray-300">或使用以下方式</span>
            </div>
          </div>

          <!-- 社交登录 -->
          <Button
            variant="outline"
            class="w-full h-11 bg-white/60 border-gray-100 rounded-xl hover:bg-gray-50 hover:border-gray-200 text-gray-500 transition-all duration-300"
            type="button"
          >
            <Mail class="mr-2 size-4" />
            <span class="text-sm">使用 Google 登录</span>
          </Button>
        </div>

        <!-- 注册链接 -->
        <div class="text-center text-sm text-gray-400 mt-6">
          还没有账号？
          <RouterLink to="/user/register" class="text-violet-500 hover:text-violet-600 font-medium transition-colors duration-300">
            去注册
          </RouterLink>
        </div>
      </div>
    </div>
  </div>
</template>
