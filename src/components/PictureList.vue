<template>
  <div class="picture-list">
    <!-- 图片列表 -->
    <a-list
      :grid="{ gutter: 16, xs: 1, sm: 2, md: 3, lg: 4, xl: 5, xxl: 6 }"
      :data-source="dataList"
      :loading="loading"
    >
      <template #renderItem="{ item: picture }">
        <a-list-item style="padding: 0">
          <!-- 单张图片 -->
          <!-- <a-card hoverable @click="doClickPicture(picture)"> -->
          <!-- // ==================== 【改动 1】==================== -->
          <a-card
            hoverable
            class="picture-card"
            :class="{ 'is-selected': showSelect && isChecked(picture.id) }"
            @click="doClickPicture(picture)"
          >
            <!-- // =================================================== -->
            <!--  -->
            <!-- // ==================== 【改动 2】==================== -->
            <div v-if="showSelect" class="card-checkbox" @click.stop>
              <a-checkbox
                :checked="isChecked(picture.id)"
                @change="(e) => handleCheckChange(e.target.checked, picture)"
              />
            </div>
            <!-- // =================================================== -->
            <template #cover>
              <img
                :alt="picture.name"
                :src="picture.thumbnailUrl ?? picture.url"
                style="height: 180px; object-fit: cover"
              />
            </template>
            <a-card-meta :title="picture.name">
              <template #description>
                <a-flex>
                  <a-tag color="green">
                    {{ picture.category ?? '默认' }}
                  </a-tag>
                  <a-tag v-for="tag in picture.tags" :key="tag">
                    {{ tag }}
                  </a-tag>
                </a-flex>
              </template>
            </a-card-meta>
            <template v-if="showOp" #actions>
              <ShareAltOutlined @click="(e) => doShare(picture, e)" />
              <SearchOutlined @click="(e) => doSearch(picture, e)" />
              <EditOutlined @click="(e) => doEdit(picture, e)" />
              <DeleteOutlined @click="(e) => doDelete(picture, e)" />
            </template>
          </a-card>
        </a-list-item>
      </template>
    </a-list>
    <ShareModal ref="shareModalRef" :link="shareLink" />
  </div>
</template>

<script setup lang="ts">
import { useRouter } from 'vue-router'
import {
  DeleteOutlined,
  EditOutlined,
  SearchOutlined,
  ShareAltOutlined,
} from '@ant-design/icons-vue'
import { deletePictureUsingPost } from '@/api/pictureController.ts'
import { message } from 'ant-design-vue'
import ShareModal from '@/components/ShareModal.vue'
import { ref } from 'vue'

interface Props {
  dataList?: API.PictureVO[]
  loading?: boolean
  showOp?: boolean
  onReload?: () => void

  // ==================== 【改动 3】====================
  // 在 Props 接口中，增加接收父组件传过来的“已选择图片对象数组”
  selectedRows?: API.PictureVO[]
  // ===================================================
  // ====== 【新加的控制开关】======
  showSelect?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  dataList: () => [],
  loading: false,
  showOp: false,
  // ==================== 【改动 4】====================
  // 为新加的 selectedRows 赋予默认空数组，防止未传参时报错
  selectedRows: () => [],
  // ===================================================
  // ====== 【默认不开启多选】======
  showSelect: false,
})

// ==================== 【改动 5】====================
// 声明自定义事件，update:selectedRows 是 Vue3 v-model 语法的标准命名
const emit = defineEmits(['update:selectedRows'])

/**
 * 判断当前图片卡片是否已被勾选
 */
const isChecked = (id: string | number) => {
  return props.selectedRows.some((item) => item.id === id)
}

/**
 * 处理多选框的勾选与取消勾选状态
 */
const handleCheckChange = (checked: boolean, picture: API.PictureVO) => {
  // 浅拷贝一份当前选中的数据，避免直接修改 props 报错
  const nextSelectedRows = [...props.selectedRows]

  if (checked) {
    // 勾选：把完整的图片对象追加进去
    nextSelectedRows.push(picture)
  } else {
    // 取消勾选：从现有已选数组中找到对应的图片并剔除
    const index = nextSelectedRows.findIndex((row) => row.id === picture.id)
    if (index > -1) {
      nextSelectedRows.splice(index, 1)
    }
  }
  // 将最新的已选数组通知给父组件
  emit('update:selectedRows', nextSelectedRows)
}
// ===================================================

const router = useRouter()
// 跳转至图片详情页
const doClickPicture = (picture: API.PictureVO) => {
  router.push({
    path: `/picture/${picture.id}`,
  })
}

// 搜索
const doSearch = (picture, e) => {
  // 阻止冒泡
  e.stopPropagation()
  // 打开新的页面
  window.open(`/search_picture?pictureId=${picture.id}`)
}

// 编辑
const doEdit = (picture, e) => {
  // 阻止冒泡
  e.stopPropagation()
  // 跳转时一定要携带 spaceId
  router.push({
    path: '/add_picture',
    query: {
      id: picture.id,
      spaceId: picture.spaceId,
    },
  })
}

// 删除数据
const doDelete = async (picture, e) => {
  // 阻止冒泡
  e.stopPropagation()
  const id = picture.id
  if (!id) {
    return
  }
  const res = await deletePictureUsingPost({ id })
  if (res.data.code === 0) {
    message.success('删除成功')
    props.onReload?.()
  } else {
    message.error('删除失败')
  }
}

// ----- 分享操作 ----
const shareModalRef = ref()
// 分享链接
const shareLink = ref<string>()
// 分享
const doShare = (picture, e) => {
  // 阻止冒泡
  e.stopPropagation()
  shareLink.value = `${window.location.protocol}//${window.location.host}/picture/${picture.id}`
  if (shareModalRef.value) {
    shareModalRef.value.openModal()
  }
}
</script>

<style scoped>
/* ==================== 【改动 6】==================== */
/* 为卡片加上相对定位，因为里面的复选框要用绝对定位悬浮 */
.picture-card {
  position: relative;
  transition: all 0.3s;
}

/* 漂浮定位在卡片的右上方，并美化一下复选框的半透明背景层 */
.card-checkbox {
  position: absolute;
  top: 12px;
  right: 12px;
  z-index: 10;
  background: rgba(255, 255, 255, 0.85);
  padding: 4px 6px;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* 被选中后的高亮边框样式 */
.is-selected {
  border-color: #1890ff !important;
  box-shadow: 0 0 10px rgba(24, 144, 255, 0.25) !important;
}
/* =================================================== */
</style>
