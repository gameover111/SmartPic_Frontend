<template>
  <div id="addSpacePage">
    <h2 style="margin-bottom: 16px">
      {{ route.query?.id ? '修改' : '创建' }} {{ SPACE_TYPE_MAP[spaceType] }}
    </h2>
    <!-- 空间信息表单 -->
    <a-form name="spaceForm" layout="vertical" :model="spaceForm" @finish="handleSubmit">
      <a-form-item name="spaceName" label="空间名称">
        <a-input v-model:value="spaceForm.spaceName" placeholder="请输入空间" allow-clear />
      </a-form-item>
      <a-form-item name="spaceLevel" label="空间级别">
        <a-select
          v-model:value="spaceForm.spaceLevel"
          style="min-width: 180px"
          placeholder="请选择空间级别"
          :options="SPACE_LEVEL_OPTIONS"
          allow-clear
        />
      </a-form-item>

      <a-form-item v-if="isAdmin" name="maxSize" label="空间最大容量 (单位: MB)">
        <a-input-number
          v-model:value="inputSize"
          style="width: 100%"
          placeholder="请输入空间大小"
          :min="1"
          @change="handleSizeChange"
        />
        <span style="font-size: 12px; color: #8c8c8c" v-if="spaceForm.maxSize">
          转换后提交为：{{ spaceForm.maxSize }} Bytes
        </span>
      </a-form-item>

      <a-form-item v-if="isAdmin" name="maxCount" label="最大图片数量">
        <a-input-number
          v-model:value="spaceForm.maxCount"
          style="width: 100%"
          placeholder="不填则使用级别默认数量"
          :min="1"
        />
      </a-form-item>

      <a-form-item>
        <a-button type="primary" html-type="submit" :loading="loading" style="width: 100%">
          提交
        </a-button>
      </a-form-item>
    </a-form>
    <!-- 空间级别介绍 -->
    <a-card title="空间级别介绍">
      <a-typography-paragraph>
        * 目前仅支持开通普通版，如需升级空间，请
        <a href="https://github.com/gameover111/SmartPic_Frontend" target="_blank">联系作者:hsc</a>
      </a-typography-paragraph>
      <a-typography-paragraph v-for="spaceLevel in spaceLevelList" :key="spaceLevel.text">
        <span style="margin-right: 6px">
          <span v-if="spaceLevel.text.includes('普通')">⚡</span>
          <span v-else-if="spaceLevel.text.includes('专业')">💎</span>
          <span v-else-if="spaceLevel.text.includes('旗舰')">👑</span>
        </span>
        {{ spaceLevel.text }}：大小 {{ formatSize(spaceLevel.maxSize) }}，数量
        {{ spaceLevel.maxCount }}
      </a-typography-paragraph>
    </a-card>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'
import { message } from 'ant-design-vue'
import {
  addSpaceUsingPost,
  getSpaceVoByIdUsingGet,
  listSpaceLevelUsingGet,
  updateSpaceUsingPost,
} from '@/api/spaceController.ts'
import { useRoute, useRouter } from 'vue-router'
import {
  SPACE_LEVEL_MAP,
  SPACE_LEVEL_OPTIONS,
  SPACE_TYPE_ENUM,
  SPACE_TYPE_MAP,
} from '@/constants/space.ts'
import { formatSize } from '../utils'
import { useLoginUserStore } from '@/stores/useLoginUserStore.ts'

const loginUserStore = useLoginUserStore()
const isAdmin = computed(() => {
  const loginUser = loginUserStore.loginUser
  return loginUser?.userRole === 'admin'
})

const space = ref<API.SpaceVO>()
const spaceForm = reactive<API.SpaceAddRequest | API.SpaceEditRequest>({})
const loading = ref(false)

const route = useRoute()
// 空间类别，默认为私有空间
const spaceType = computed(() => {
  if (route.query?.type) {
    return Number(route.query.type)
  } else {
    return SPACE_TYPE_ENUM.PRIVATE
  }
})

const spaceLevelList = ref<API.SpaceLevel[]>([])

// 获取空间级别
const fetchSpaceLevelList = async () => {
  const res = await listSpaceLevelUsingGet()
  if (res.data.code === 0 && res.data.data) {
    spaceLevelList.value = res.data.data
  } else {
    message.error('获取空间级别失败，' + res.data.message)
  }
}

onMounted(() => {
  fetchSpaceLevelList()
})

const router = useRouter()

/**
 * 提交表单
 * @param values
 */
const handleSubmit = async (values: any) => {
  const spaceId = space.value?.id
  loading.value = true
  let res
  if (spaceId) {
    // 更新
    res = await updateSpaceUsingPost({
      id: spaceId,
      ...spaceForm,
    })
  } else {
    // 创建
    res = await addSpaceUsingPost({
      ...spaceForm,
      spaceType: spaceType.value,
    })
  }
  // 操作成功
  if (res.data.code === 0 && res.data.data) {
    message.success('操作成功')

    // ====== 🎯 合理跳转的核心修复点 ======
    let targetId
    if (spaceId) {
      // 1. 如果是更新，后端返回的是 true，我们不用它，直接用原本就已经拿到的 spaceId 数字ID
      targetId = spaceId
    } else {
      // 2. 如果是创建，后端返回的是新空间的 ID，我们直接读取 res.data.data
      targetId = res.data.data
    }

    // 顺滑跳转，再也不会出现 /space/true 了
    router.push({
      path: `/space/${targetId}`,
    })
    // ===================================

    // // 跳转到空间详情页
    // router.push({
    //   path: `/space/${res.data.data}`,
    // })
  } else {
    message.error('操作失败，' + res.data.message)
  }
  loading.value = false
}

// 获取老数据
const getOldSpace = async () => {
  // 获取到 id
  const id = route.query?.id
  if (id) {
    const res = await getSpaceVoByIdUsingGet({
      id,
    })
    if (res.data.code === 0 && res.data.data) {
      const data = res.data.data
      space.value = data
      // 填充表单
      spaceForm.spaceName = data.spaceName
      spaceForm.spaceLevel = data.spaceLevel
      const sizeInBytes = Number(data.maxSize)
      inputSize.value = Number((sizeInBytes / (1024 * 1024)).toFixed(2))
      spaceForm.maxSize = sizeInBytes
      spaceForm.maxCount = data.maxCount
    }
  }
}

onMounted(() => {
  getOldSpace()
})

// ====== ✨ 1. 定义一个单纯的输入框数值绑定 ======
const inputSize = ref<number | undefined>(undefined)

// ====== ✨ 2. 用户修改 MB 数字时，自动乘以 1024 * 1024 换算成 Bytes ======
const handleSizeChange = () => {
  if (inputSize.value) {
    spaceForm.maxSize = inputSize.value * 1024 * 1024
  } else {
    spaceForm.maxSize = undefined
  }
}
</script>

<style scoped>
#addSpacePage {
  max-width: 720px;
  margin: 0 auto;
}
</style>
