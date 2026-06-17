<template>
  <div id="spaceDetailPage">
    <!-- 空间信息 -->
    <a-flex justify="space-between">
      <h2>{{ space.spaceName }}（私有空间）</h2>
      <a-space size="middle">
        <a-button type="primary" :href="`/add_picture?spaceId=${id}`" target="_blank">
          + 创建图片
        </a-button>
        <a-button
          type="primary"
          ghost
          :icon="h(BarChartOutlined)"
          :href="`/space_analyze?spaceId=${id}`"
          target="_blank"
        >
          空间分析
        </a-button>

        <a-button :icon="h(EditOutlined)" @click="doBatchEdit"> 批量编辑 </a-button>

        <a-tooltip
          :title="`占用空间 ${formatSize(space.totalSize)} / ${formatSize(space.maxSize)}`"
        >
          <a-progress
            type="circle"
            :size="42"
            :percent="((space.totalSize * 100) / space.maxSize).toFixed(1)"
          />
        </a-tooltip>
      </a-space>
    </a-flex>
    <div style="margin-bottom: 16px" />
    <!-- 搜索表单 -->
    <PictureSearchForm :onSearch="onSearch" />
    <div style="margin-bottom: 16px" />
    <!-- 按颜色搜索，跟其他搜索条件独立 -->
    <!-- <a-form-item label="按颜色搜索">
      <color-picker format="hex" @pureColorChange="onColorChange" />
    </a-form-item> -->

    <a-flex align="center" style="gap: 16px">
      <span style="color: rgba(0, 0, 0, 0.85); font-size: 14px; white-space: nowrap">
        按颜色搜索：
      </span>

      <div style="display: flex; align-items: center; height: 32px">
        <color-picker format="hex" @pureColorChange="onColorChange" />
      </div>

      <a-button
        v-if="selectedPictures.length > 0"
        type="dashed"
        danger
        size="small"
        style="height: 32px; border-radius: 4px"
        @click="clearSelection"
      >
        取消选中 (已选 {{ selectedPictures.length }} 张)
      </a-button>
    </a-flex>
    <div style="margin-bottom: 12px" />
    <!-- 图片列表 -->
    <!-- <PictureList :dataList="dataList" :loading="loading" :showOp="true" :onReload="fetchData" /> -->
    <PictureList
      :dataList="dataList"
      :loading="loading"
      :showOp="true"
      :onReload="fetchData"
      v-model:selectedRows="selectedPictures"
      :showSelect="true"
    />
    <!-- 分页 -->
    <a-pagination
      style="text-align: right"
      v-model:current="searchParams.current"
      v-model:pageSize="searchParams.pageSize"
      :total="total"
      @change="onPageChange"
    />
    <!-- 批量编辑图片 弹窗 -->
    <!-- <BatchEditPictureModal
      ref="batchEditPictureModalRef"
      :spaceId="id"
      :pictureList="dataList"
      :onSuccess="onBatchEditPictureSuccess"
    /> -->
    <BatchEditPictureModal
      ref="batchEditPictureModalRef"
      :spaceId="id"
      :pictureList="effectivePictureList"
      :onSuccess="onBatchEditPictureSuccess"
    />
  </div>
</template>

<script setup lang="ts">
import { h, onMounted, ref, computed } from 'vue'
import { getSpaceVoByIdUsingGet } from '@/api/spaceController.ts'
import { message } from 'ant-design-vue'
import {
  listPictureVoByPageUsingPost,
  searchPictureByColorUsingPost,
} from '@/api/pictureController.ts'
import { formatSize } from '@/utils'
import PictureList from '@/components/PictureList.vue'
import PictureSearchForm from '@/components/PictureSearchForm.vue'
import { ColorPicker } from 'vue3-colorpicker'
import 'vue3-colorpicker/style.css'
import BatchEditPictureModal from '@/components/BatchEditPictureModal.vue'
import { BarChartOutlined, EditOutlined } from '@ant-design/icons-vue'

interface Props {
  id: string | number
}

const props = defineProps<Props>()
const space = ref<API.SpaceVO>({})

// -------- 获取空间详情 --------
const fetchSpaceDetail = async () => {
  try {
    const res = await getSpaceVoByIdUsingGet({
      id: props.id,
    })
    if (res.data.code === 0 && res.data.data) {
      space.value = res.data.data
    } else {
      message.error('获取空间详情失败，' + res.data.message)
    }
  } catch (e: any) {
    message.error('获取空间详情失败：' + e.message)
  }
}

onMounted(() => {
  fetchSpaceDetail()
})

// --------- 获取图片列表 --------

// 定义数据
const dataList = ref<API.PictureVO[]>([])
const total = ref(0)
const loading = ref(true)

// 搜索条件
const searchParams = ref<API.PictureQueryRequest>({
  current: 1,
  pageSize: 12,
  sortField: 'createTime',
  sortOrder: 'descend',
})

// 获取数据
const fetchData = async () => {
  loading.value = true
  // 转换搜索参数
  const params = {
    spaceId: props.id,
    ...searchParams.value,
  }
  const res = await listPictureVoByPageUsingPost(params)
  if (res.data.code === 0 && res.data.data) {
    dataList.value = res.data.data.records ?? []
    total.value = res.data.data.total ?? 0
  } else {
    message.error('获取数据失败，' + res.data.message)
  }
  loading.value = false
}

// 页面加载时获取数据，请求一次
onMounted(() => {
  fetchData()
})

// 分页参数
const onPageChange = (page: number, pageSize: number) => {
  searchParams.value.current = page
  searchParams.value.pageSize = pageSize
  fetchData()
}

// 搜索
const onSearch = (newSearchParams: API.PictureQueryRequest) => {
  console.log('new', newSearchParams)

  searchParams.value = {
    ...searchParams.value,
    ...newSearchParams,
    current: 1,
  }
  console.log('searchparams', searchParams.value)
  fetchData()
}

// 按照颜色搜索
const onColorChange = async (color: string) => {
  loading.value = true
  const res = await searchPictureByColorUsingPost({
    picColor: color,
    spaceId: props.id,
  })
  if (res.data.code === 0 && res.data.data) {
    const data = res.data.data ?? []
    dataList.value = data
    total.value = data.length
  } else {
    message.error('获取数据失败，' + res.data.message)
  }
  loading.value = false
}

// ---- 批量编辑图片 -----
const batchEditPictureModalRef = ref()

// 批量编辑图片成功
// const onBatchEditPictureSuccess = () => {
//   fetchData()
// }

// 批量编辑图片成功后的回调
const onBatchEditPictureSuccess = () => {
  // 1. 刷新列表数据
  fetchData()
  // 2. 关键：成功后清空勾选状态，避免留在页面上干扰下次操作
  selectedPictures.value = []
}

// 打开批量编辑图片弹窗
const doBatchEdit = () => {
  if (batchEditPictureModalRef.value) {
    batchEditPictureModalRef.value.openModal()
  }
}

// ==================== 【新加的清空逻辑】====================
/**
 * 清空所有已选中的图片
 */
const clearSelection = () => {
  selectedPictures.value = []
  message.info('已取消所有选中图片')
}
// ==========================================================

// --------- 批量选择与编辑逻辑 --------

// 存储用户手动勾选的图片对象数组
const selectedPictures = ref<API.PictureVO[]>([])

/**
 * 核心计算属性：
 * 如果用户勾选了，就返回勾选的图片；
 * 如果没有勾选，默认返回当前页面的所有图片 dataList
 */
const effectivePictureList = computed(() => {
  if (selectedPictures.value && selectedPictures.value.length > 0) {
    return selectedPictures.value
  }
  return dataList.value
})
</script>

<style scoped>
#spaceDetailPage {
  margin-bottom: 16px;
}
</style>
