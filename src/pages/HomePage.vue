<template>
  <div id="homePage">
    <!-- 搜索框 -->
    <div class="search-bar">
      <a-input-search
        v-model:value="searchParams.searchText"
        placeholder="从海量图片中搜索"
        enter-button="搜索"
        size="large"
        @search="doSearch"
      />
    </div>
    <!-- 分类和标签筛选 -->
    <div class="filter-section">
      <a-tabs v-model:active-key="selectedCategory" @change="doSearch">
        <a-tab-pane key="all" tab="全部" />
        <a-tab-pane v-for="category in categoryList" :tab="category" :key="category" />
      </a-tabs>
      <div class="tag-bar">
        <span class="tag-label">标签：</span>
        <a-space :size="[0, 8]" wrap>
          <a-checkable-tag
            v-for="(tag, index) in tagList"
            :key="tag"
            v-model:checked="selectedTagList[index]"
            @change="doSearch"
          >
            {{ tag }}
          </a-checkable-tag>
        </a-space>
      </div>
    </div>
    <!-- 图片列表 -->
    <PictureList :dataList="dataList" :loading="loading" />
    <!-- 分页 -->
    <div class="pagination-wrapper">
      <a-pagination
        v-model:current="searchParams.current"
        v-model:pageSize="searchParams.pageSize"
        :total="total"
        @change="onPageChange"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import {
  listPictureTagCategoryUsingGet,
  listPictureVoByPageUsingPost,
  listPictureVoByPageWithMultiCacheUsingPost,
} from '@/api/pictureController.ts'
import { message } from 'ant-design-vue'
import PictureList from '@/components/PictureList.vue' // 定义数据

// 定义数据
const dataList = ref<API.PictureVO[]>([])
const total = ref(0)
const loading = ref(true)

// 搜索条件
const searchParams = reactive<API.PictureQueryRequest>({
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
    ...searchParams,
    tags: [] as string[],
  }
  if (selectedCategory.value !== 'all') {
    params.category = selectedCategory.value
  }
  // [true, false, false] => ['java']
  selectedTagList.value.forEach((useTag, index) => {
    if (useTag) {
      params.tags.push(tagList.value[index])
    }
  })
  const res = await listPictureVoByPageWithMultiCacheUsingPost(params)
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

const onPageChange = (page: number, pageSize: number) => {
  searchParams.current = page
  searchParams.pageSize = pageSize
  fetchData()
}
// 搜索
const doSearch = () => {
  // 重置搜索条件
  searchParams.current = 1
  fetchData()
}

// 标签和分类列表
const categoryList = ref<string[]>([])
const selectedCategory = ref<string>('all')
const tagList = ref<string[]>([])
const selectedTagList = ref<boolean[]>([])

/**
 * 获取标签和分类选项
 * @param values
 */
const getTagCategoryOptions = async () => {
  const res = await listPictureTagCategoryUsingGet()
  if (res.data.code === 0 && res.data.data) {
    tagList.value = res.data.data.tagList ?? []
    categoryList.value = res.data.data.categoryList ?? []
  } else {
    message.error('获取标签分类列表失败，' + res.data.message)
  }
}

onMounted(() => {
  getTagCategoryOptions()
})
</script>

<style scoped>
#homePage {
  margin-bottom: 16px;
}

#homePage .search-bar {
  max-width: 520px;
  margin: 0 auto 20px;
}

#homePage .search-bar :deep(.ant-input-search) {
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
}

#homePage .search-bar :deep(.ant-input-search .ant-input) {
  border-radius: 12px 0 0 12px;
  border-color: rgba(0, 0, 0, 0.1);
}

#homePage .search-bar :deep(.ant-input-search .ant-input:focus) {
  border-color: #555;
  box-shadow: 0 0 0 2px rgba(0, 0, 0, 0.06);
}

#homePage .search-bar :deep(.ant-input-search .ant-btn) {
  background: linear-gradient(135deg, #555, #333);
  border-color: #555;
  border-radius: 0 12px 12px 0;
}

#homePage .search-bar :deep(.ant-input-search .ant-btn:hover) {
  background: linear-gradient(135deg, #666, #444);
}

#homePage .filter-section {
  background: white;
  border-radius: 12px;
  padding: 4px 20px 12px;
  margin-bottom: 20px;
  box-shadow: 0 1px 6px rgba(0, 0, 0, 0.04);
}

#homePage .filter-section :deep(.ant-tabs-nav::before) {
  border-bottom-color: rgba(0, 0, 0, 0.04);
}

#homePage .filter-section :deep(.ant-tabs-tab.ant-tabs-tab-active .ant-tabs-tab-btn) {
  color: #555;
}

#homePage .filter-section :deep(.ant-tabs-ink-bar) {
  background: #555;
}

#homePage .tag-bar {
  margin-bottom: 8px;
}

#homePage .tag-label {
  margin-right: 8px;
  color: #888;
  font-size: 13px;
}

#homePage :deep(.ant-checkable-tag) {
  border-radius: 6px;
  border: 1px solid #e8e8e8;
  transition: all 0.25s;
}

#homePage :deep(.ant-checkable-tag-checked) {
  background: rgba(0, 0, 0, 0.06) !important;
  color: #555 !important;
  border-color: rgba(0, 0, 0, 0.12);
}

#homePage .pagination-wrapper {
  display: flex;
  justify-content: flex-end;
  margin-top: 20px;
  padding: 12px 0;
}

#homePage .pagination-wrapper :deep(.ant-pagination-item-active) {
  border-color: #555;
}

#homePage .pagination-wrapper :deep(.ant-pagination-item-active a) {
  color: #555;
}
</style>
