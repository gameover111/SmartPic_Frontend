<template>
  <div class="batch-edit-picture-modal">
    <a-modal v-model:visible="visible" title="批量编辑图片" :footer="false" @cancel="closeModal">
      <!-- <a-typography-paragraph type="secondary"
        >* 若未选择图片，则默认当前页面的所有图片生效</a-typography-paragraph
      > -->
      <a-typography-paragraph type="secondary" v-html="tipMessage"> </a-typography-paragraph>
      <!-- 批量创建表单 -->
      <a-form name="formData" layout="vertical" :model="formData" @finish="handleSubmit">
        <a-form-item name="category" label="分类">
          <a-auto-complete
            v-model:value="formData.category"
            placeholder="请输入分类"
            :options="categoryOptions"
            allow-clear
          />
        </a-form-item>
        <a-form-item name="tags" label="标签">
          <a-select
            v-model:value="formData.tags"
            mode="tags"
            placeholder="请输入标签"
            :options="tagOptions"
            allow-clear
          />
        </a-form-item>
        <!-- <a-form-item name="nameRule" label="命名规则">
          <a-input
            v-model:value="formData.nameRule"
            placeholder="请输入命名规则，输入 {序号} 可动态生成"
            allow-clear
          />
        </a-form-item> -->

        <a-form-item name="nameRule" label="命名规则">
          <div style="margin-bottom: 8px">
            <span style="font-size: 12px; color: #8c8c8c; margin-right: 8px">点击插入变量：</span>
            <a-space wrap>
              <a-tag color="blue" style="cursor: pointer" @click="insertVariable('{序号}')">
                + 序号
              </a-tag>
              <a-tag color="orange" style="cursor: pointer" @click="insertVariable('{当前日期}')">
                + 当前日期
              </a-tag>
            </a-space>
          </div>

          <a-input
            v-model:value="formData.nameRule"
            placeholder="例如：智能素材_{当前日期}_{序号}"
            allow-clear
          />
          <div style="font-size: 12px; color: #bfbfbf; margin-top: 4px">
            示例：“壁纸_{当前日期}_{序号}” ➡ “壁纸_20260615_1”
          </div>
        </a-form-item>

        <a-form-item>
          <a-button type="primary" html-type="submit" style="width: 100%">提交</a-button>
        </a-form-item>
      </a-form>
    </a-modal>
  </div>
</template>
<script lang="ts" setup>
import { onMounted, reactive, ref, computed } from 'vue'
import {
  editPictureByBatchUsingPost,
  listPictureTagCategoryUsingGet,
} from '@/api/pictureController.ts'
import { message } from 'ant-design-vue'

// ==================== 【新加：动态提示文字计算属性】====================
const tipMessage = computed(() => {
  const count = props.pictureList?.length ?? 0
  // 1. 如果有 spaceId (说明是私有图库)
  if (props.spaceId !== null) {
    return `* 若未手动选择图片，则默认对当前页面的所有图片生效。<br /> * 已选择 ${count} 张图片。`
  }

  // 2. 如果 spaceId 为 null (说明是公共图库管理)
  return `* 已选中 ${count} 张图片，将对这些图片进行批量操作。`
})
// ====================================================================

interface Props {
  pictureList: API.PictureVO[]
  spaceId: number | null
  onSuccess: () => void
}

const props = withDefaults(defineProps<Props>(), {})

// 是否可见
const visible = ref(false)

// 打开弹窗
const openModal = () => {
  visible.value = true
}

// 关闭弹窗
const closeModal = () => {
  visible.value = false
}

// 暴露函数给父组件
defineExpose({
  openModal,
})

const formData = reactive<API.PictureEditByBatchRequest>({
  category: '',
  tags: [],
  nameRule: '',
})

/**
 * 提交表单
 * @param values
 */
const handleSubmit = async (values: any) => {
  if (!props.pictureList) {
    return
  }
  const res = await editPictureByBatchUsingPost({
    pictureIdList: props.pictureList.map((picture) => picture.id),
    spaceId: props.spaceId,
    ...values,
  })
  // 操作成功
  if (res.data.code === 0 && res.data.data) {
    message.success('操作成功')
    closeModal()
    props.onSuccess?.()
  } else {
    message.error('操作失败，' + res.data.message)
  }
}

const categoryOptions = ref<string[]>([])
const tagOptions = ref<string[]>([])

/**
 * 获取标签和分类选项
 * @param values
 */
const getTagCategoryOptions = async () => {
  const res = await listPictureTagCategoryUsingGet()
  if (res.data.code === 0 && res.data.data) {
    tagOptions.value = (res.data.data.tagList ?? []).map((data: string) => {
      return {
        value: data,
        label: data,
      }
    })
    categoryOptions.value = (res.data.data.categoryList ?? []).map((data: string) => {
      return {
        value: data,
        label: data,
      }
    })
  } else {
    message.error('获取标签分类列表失败，' + res.data.message)
  }
}

// ==================== 【新加位置：点击快速插入变量】====================
const insertVariable = (variableStr: string) => {
  // 1. 如果本来为空，直接赋值，不需要加下划线
  if (!formData.nameRule) {
    formData.nameRule = variableStr
    return
  }

  // 2. 如果不为空，检查最后一个字符是不是已经是下划线 '_'
  // 如果不是，就先补一个下划线，再拼上新的变量
  if (!formData.nameRule.endsWith('_')) {
    formData.nameRule += '_'
  }

  // 3. 追加变量内容
  formData.nameRule += variableStr
}
// ====================================================================

onMounted(() => {
  getTagCategoryOptions()
})
</script>
