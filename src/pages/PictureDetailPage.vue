<template>
  <div id="pictureDetailPage">
    <a-row :gutter="[16, 16]">
      <!-- 图片预览 -->
      <a-col :sm="24" :md="16" :xl="18">
        <a-card title="图片预览">
          <a-image :src="picture.url" style="max-height: 600px; object-fit: contain" />
        </a-card>
      </a-col>
      <!-- 图片信息区域 -->
      <a-col :sm="24" :md="8" :xl="6">
        <a-card title="图片信息">
          <a-descriptions :column="1">
            <a-descriptions-item label="作者">
              <a-space>
                <a-avatar :size="24" :src="picture.user?.userAvatar" />
                <div>{{ picture.user?.userName }}</div>
              </a-space>
            </a-descriptions-item>
            <a-descriptions-item label="名称">
              {{ picture.name ?? '未命名' }}
            </a-descriptions-item>
            <a-descriptions-item label="简介">
              {{ picture.introduction ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="分类">
              {{ picture.category ?? '默认' }}
            </a-descriptions-item>
            <a-descriptions-item label="标签">
              <a-tag v-for="tag in picture.tags" :key="tag">
                {{ tag }}
              </a-tag>
            </a-descriptions-item>
            <a-descriptions-item label="格式">
              {{ picture.picFormat ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="宽度">
              {{ picture.picWidth ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="高度">
              {{ picture.picHeight ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="宽高比">
              {{ picture.picScale ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="大小">
              {{ formatSize(picture.picSize) }}
            </a-descriptions-item>

            <!-- 审核信息区域：仅管理员可见 -->
            <template v-if="isAdmin">
              <a-descriptions-item label="审核状态">
                <a-tag :color="getReviewStatusColor(picture.reviewStatus)">
                  {{ PIC_REVIEW_STATUS_MAP[picture.reviewStatus ?? 0] ?? '待审核' }}
                </a-tag>
              </a-descriptions-item>
              <a-descriptions-item v-if="picture.reviewMessage" label="审核信息">
                {{ picture.reviewMessage }}
              </a-descriptions-item>
              <a-descriptions-item v-if="picture.reviewerId" label="审核人ID">
                {{ picture.reviewerId }}
              </a-descriptions-item>
              <a-descriptions-item v-if="picture.reviewTime" label="审核时间">
                {{ dayjs(picture.reviewTime).format('YYYY-MM-DD HH:mm:ss') }}
              </a-descriptions-item>
            </template>
          </a-descriptions>
          <!-- 图片操作 -->
          <a-space wrap>
            <a-button type="primary" @click="doDownload">
              免费下载
              <template #icon>
                <DownloadOutlined />
              </template>
            </a-button>
            <a-button v-if="canEdit" :icon="h(EditOutlined)" type="default" @click="doEdit">
              编辑
            </a-button>
            <a-button v-if="canEdit" :icon="h(DeleteOutlined)" danger @click="doDelete">
              删除
            </a-button>

            <!-- 审核通过/拒绝按钮 -->
            <a-popconfirm
              v-if="isAdmin && picture.reviewStatus !== PIC_REVIEW_STATUS_ENUM.PASS"
              title="确认审核通过该图片吗？"
              @confirm="handleReview(PIC_REVIEW_STATUS_ENUM.PASS, '管理员操作通过')"
            >
              <a-button type="primary" style="background-color: #52c41a">审核通过</a-button>
            </a-popconfirm>
            <a-popconfirm
              v-if="isAdmin && picture.reviewStatus !== PIC_REVIEW_STATUS_ENUM.REJECT"
              title="确认审核拒绝该图片吗？"
              @confirm="handleReview(PIC_REVIEW_STATUS_ENUM.REJECT, '管理员操作拒绝')"
            >
              <a-button danger>审核拒绝</a-button>
            </a-popconfirm>
          </a-space>
        </a-card>
      </a-col>
    </a-row>
  </div>
</template>

<script setup lang="ts">
import { computed, h, onMounted, ref } from 'vue'
import {
  deletePictureUsingPost,
  getPictureVoByIdUsingGet,
  doPictureReviewUsingPost,
  listPictureByPageUsingPost,
} from '@/api/pictureController.ts'
import { message } from 'ant-design-vue'
import { DeleteOutlined, DownloadOutlined, EditOutlined } from '@ant-design/icons-vue'
import { useLoginUserStore } from '@/stores/useLoginUserStore.ts'
import { useRouter } from 'vue-router'
import { downloadImage, formatSize } from '@/utils'
import { PIC_REVIEW_STATUS_ENUM, PIC_REVIEW_STATUS_MAP } from '../constants/picture.ts'
import dayjs from 'dayjs'

interface Props {
  id: string | number
}

const props = defineProps<Props>()
const picture = ref<API.PictureVO & Partial<API.Picture>>({}) // 扩展类型，兼容审核字段

const loginUserStore = useLoginUserStore()

// 判断当前登录用户是否为管理员（新增）
const isAdmin = computed(() => {
  const loginUser = loginUserStore.loginUser
  return loginUser?.userRole === 'admin'
})
// 是否具有编辑权限
const canEdit = computed(() => {
  const loginUser = loginUserStore.loginUser
  // 未登录不可编辑
  if (!loginUser.id) {
    return false
  }
  // 仅本人或管理员可编辑
  const user = picture.value.user || {}
  return loginUser.id === user.id || loginUser.userRole === 'admin'
})
// 辅助函数：获取审核状态对应的标签颜色（新增）
const getReviewStatusColor = (status: number | undefined) => {
  const s = status ?? 0
  if (s === PIC_REVIEW_STATUS_ENUM.PASS) return 'green'
  if (s === PIC_REVIEW_STATUS_ENUM.REJECT) return 'red'
  return 'gold'
}
// 获取基础详情（VO，无审核字段）
const fetchPictureDetail = async () => {
  try {
    const res = await getPictureVoByIdUsingGet({
      id: props.id,
    })
    if (res.data.code === 0 && res.data.data) {
      picture.value = { ...picture.value, ...res.data.data }
    } else {
      message.error('获取图片详情失败，' + res.data.message)
    }
  } catch (e: any) {
    message.error('获取图片详情失败：' + e.message)
  }
}
// 从列表接口获取完整实体数据（包含审核字段）
const fetchFullEntity = async () => {
  try {
    // 注意：列表接口 listPictureByPageUsingPost 支持通过 id 精确查询
    const res = await listPictureByPageUsingPost({
      id: props.id,
      current: 1,
      pageSize: 1,
    })
    const entity = res.data.data?.records?.[0]
    if (entity) {
      // 合并审核相关字段到 picture.value
      picture.value.reviewStatus = entity.reviewStatus
      picture.value.reviewMessage = entity.reviewMessage
      picture.value.reviewerId = entity.reviewerId
      picture.value.reviewTime = entity.reviewTime
    } else {
      console.warn('无法从列表接口获取审核信息')
    }
  } catch (e: any) {
    console.error('获取审核信息失败：', e.message)
  }
}

onMounted(async () => {
  await fetchPictureDetail()
  if (isAdmin.value) {
    await fetchFullEntity() // 仅管理员需要审核信息
  }
})

const router = useRouter()

// 编辑
const doEdit = () => {
  router.push({
    path: '/add_picture',
    query: {
      id: picture.value.id,
      spaceId: picture.value.spaceId,
    },
  })
}

// 删除数据
const doDelete = async () => {
  const id = picture.value.id
  if (!id) {
    return
  }
  const res = await deletePictureUsingPost({ id })
  if (res.data.code === 0) {
    message.success('删除成功')
    router.push('/')
  } else {
    message.error('删除失败')
  }
}

// 下载图片
const doDownload = () => {
  downloadImage(picture.value.url)
}

// 审核图片（新增）
const handleReview = async (reviewStatus: number, reviewMessage: string) => {
  const id = picture.value.id
  if (!id) {
    message.error('图片ID不存在')
    return
  }
  try {
    const res = await doPictureReviewUsingPost({
      id,
      reviewStatus,
      reviewMessage,
    })
    if (res.data.code === 0) {
      message.success('审核操作成功')
      // 刷新数据：重新获取基础详情和审核信息
      await fetchPictureDetail()
      if (isAdmin.value) await fetchFullEntity()
    } else {
      message.error('审核操作失败，' + res.data.message)
    }
  } catch (e: any) {
    message.error('审核操作失败：' + e.message)
  }
}
</script>

<style scoped>
#pictureDetailPage {
  margin-bottom: 16px;
}
</style>
