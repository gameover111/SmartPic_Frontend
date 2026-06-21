<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { message } from 'ant-design-vue'
import { useLoginUserStore } from '@/stores/useLoginUserStore'
import { editUserUsingPost } from '@/api/userController'
import { testUploadFileUsingPost } from '@/api/fileController'
import { User, Mail, Camera, Save, ArrowLeft, Upload } from 'lucide-vue-next'
import Button from '@/components/ui/Button.vue'
import Input from '@/components/ui/Input.vue'
import Label from '@/components/ui/Label.vue'

const router = useRouter()
const loginUserStore = useLoginUserStore()

const userName = ref('')
const userProfile = ref('')
const userAvatar = ref('')
const isLoading = ref(false)
const isUploadingAvatar = ref(false)

// 隐藏的文件输入
const fileInputRef = ref<HTMLInputElement | null>(null)

// 初始化表单数据
onMounted(() => {
  const loginUser = loginUserStore.loginUser
  if (loginUser) {
    userName.value = loginUser.userName || ''
    userProfile.value = loginUser.userProfile || ''
    userAvatar.value = loginUser.userAvatar || ''
  }
})

// 计算属性：用户头像
const avatarDisplay = computed(() => {
  return userAvatar.value || ''
})

// 点击头像触发文件选择
const triggerAvatarUpload = () => {
  fileInputRef.value?.click()
}

// 头像文件上传处理
const handleAvatarFileChange = async (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (!file) return

  // 校验文件类型
  if (!file.type.startsWith('image/')) {
    message.error('请选择图片文件')
    return
  }

  // 校验文件大小（最大 5MB）
  if (file.size > 5 * 1024 * 1024) {
    message.error('图片大小不能超过 5MB')
    return
  }

  isUploadingAvatar.value = true
  try {
    const res = await testUploadFileUsingPost({}, file)
    if (res.data.code === 0 && res.data.data) {
      // 上传成功，将返回的 URL 设置到头像链接
      userAvatar.value = res.data.data
      message.success('头像上传成功')
    } else {
      message.error('头像上传失败：' + (res.data.message || '未知错误'))
    }
  } catch {
    message.error('头像上传失败，请稍后重试')
  } finally {
    isUploadingAvatar.value = false
    // 清空 input，以便再次选择同一文件
    target.value = ''
  }
}

// 保存个人信息
const handleSave = async () => {
  if (!userName.value.trim()) {
    message.warning('用户名不能为空')
    return
  }

  isLoading.value = true
  try {
    const res = await editUserUsingPost({
      userName: userName.value,
      userProfile: userProfile.value,
      userAvatar: userAvatar.value,
    })

    if (res.data.code === 0) {
      message.success('个人信息更新成功')
      // 重新获取用户信息
      await loginUserStore.fetchLoginUser()
      // 返回主页
      router.push({ path: '/', replace: true })
    } else {
      message.error('更新失败：' + (res.data.message || '未知错误'))
    }
  } catch {
    message.error('更新失败，请稍后重试')
  } finally {
    isLoading.value = false
  }
}

// 返回主页
const goBack = () => {
  router.push({ path: '/', replace: true })
}
</script>

<template>
  <div class="min-h-screen bg-background">
    <div class="max-w-2xl mx-auto p-6 lg:p-10">
      <!-- 页头 -->
      <div class="flex items-center gap-4 mb-8">
        <button
          @click="goBack"
          class="p-2 rounded-lg hover:bg-accent transition-colors text-muted-foreground hover:text-foreground"
        >
          <ArrowLeft class="size-5" />
        </button>
        <div>
          <h1 class="text-2xl font-bold tracking-tight">个人信息</h1>
          <p class="text-muted-foreground text-sm mt-1">管理您的账号信息</p>
        </div>
      </div>

      <!-- 头像区域 -->
      <div class="flex items-center gap-6 mb-10 p-6 rounded-xl border border-border/60 bg-background">
        <div class="relative group">
          <div
            class="size-24 rounded-full overflow-hidden border-2 border-border bg-muted flex items-center justify-center"
          >
            <img
              v-if="avatarDisplay"
              :src="avatarDisplay"
              alt="用户头像"
              class="w-full h-full object-cover"
            />
            <User v-else class="size-10 text-muted-foreground" />
            <!-- 上传中遮罩 -->
            <div
              v-if="isUploadingAvatar"
              class="absolute inset-0 bg-black/50 rounded-full flex items-center justify-center"
            >
              <span class="text-white text-xs">上传中...</span>
            </div>
          </div>
          <!-- 点击上传头像 -->
          <div
            class="absolute inset-0 rounded-full bg-black/40 opacity-0 group-hover:opacity-100 transition-opacity flex items-center justify-center cursor-pointer"
            @click="triggerAvatarUpload"
          >
            <Camera class="size-6 text-white" />
          </div>
        </div>
        <!-- 隐藏的文件输入 -->
        <input
          ref="fileInputRef"
          type="file"
          accept="image/*"
          class="hidden"
          @change="handleAvatarFileChange"
        />
        <div class="flex-1">
          <h3 class="font-semibold text-lg">{{ userName || '未设置昵称' }}</h3>
          <p class="text-muted-foreground text-sm">
            {{ loginUserStore.loginUser.userAccount || '' }}
          </p>
          <p class="text-xs text-muted-foreground mt-1">
            点击头像可上传图片，或在下方输入图片链接
          </p>
        </div>
      </div>

      <!-- 表单区域 -->
      <form @submit.prevent="handleSave" class="space-y-6">
        <!-- 用户名 -->
        <div class="space-y-2">
          <Label for="userName" class="text-sm font-medium">用户名</Label>
          <div class="relative">
            <User class="absolute left-3 top-1/2 -translate-y-1/2 size-4 text-muted-foreground" />
            <Input
              id="userName"
              v-model="userName"
              placeholder="请输入用户名"
              class="h-12 pl-10 bg-background border-border/60 focus:border-primary"
            />
          </div>
        </div>

        <!-- 头像 -->
        <div class="space-y-2">
          <Label for="userAvatar" class="text-sm font-medium">头像</Label>
          <div class="flex gap-2">
            <div class="relative flex-1">
              <Camera class="absolute left-3 top-1/2 -translate-y-1/2 size-4 text-muted-foreground" />
              <Input
                id="userAvatar"
                v-model="userAvatar"
                placeholder="输入头像图片 URL 或点击右侧上传"
                class="h-12 pl-10 bg-background border-border/60 focus:border-primary"
              />
            </div>
            <Button
              type="button"
              variant="outline"
              class="h-12 px-4 shrink-0"
              :disabled="isUploadingAvatar"
              @click="triggerAvatarUpload"
            >
              <Upload class="size-4 mr-1" />
              {{ isUploadingAvatar ? '上传中' : '上传' }}
            </Button>
          </div>
          <p class="text-xs text-muted-foreground">支持上传本地图片或输入图片 URL 地址</p>
        </div>

        <!-- 个人简介 -->
        <div class="space-y-2">
          <Label for="userProfile" class="text-sm font-medium">个人简介</Label>
          <textarea
            id="userProfile"
            v-model="userProfile"
            placeholder="介绍一下自己吧..."
            rows="4"
            class="flex w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50 border-border/60 focus:border-primary resize-none"
          />
        </div>

        <!-- 账号信息（只读） -->
        <div class="space-y-2">
          <Label class="text-sm font-medium">账号</Label>
          <div class="relative">
            <Mail class="absolute left-3 top-1/2 -translate-y-1/2 size-4 text-muted-foreground" />
            <Input
              :modelValue="loginUserStore.loginUser.userAccount || ''"
              disabled
              class="h-12 pl-10 bg-muted border-border/60"
            />
          </div>
          <p class="text-xs text-muted-foreground">账号信息不可修改</p>
        </div>

        <!-- 角色信息（只读） -->
        <div class="space-y-2">
          <Label class="text-sm font-medium">角色</Label>
          <Input
            :modelValue="loginUserStore.loginUser.userRole === 'admin' ? '管理员' : '普通用户'"
            disabled
            class="h-12 bg-muted border-border/60"
          />
        </div>

        <!-- 保存按钮 -->
        <div class="flex gap-4 pt-4">
          <Button
            type="submit"
            class="flex-1 h-12 text-base font-medium"
            :disabled="isLoading"
          >
            <Save class="mr-2 size-4" />
            {{ isLoading ? '保存中...' : '保存修改' }}
          </Button>
          <Button
            type="button"
            variant="outline"
            class="h-12 px-6"
            @click="goBack"
          >
            取消
          </Button>
        </div>
      </form>
    </div>
  </div>
</template>