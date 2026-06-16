<template>
  <a-modal
    class="image-out-painting"
    v-model:visible="visible"
    title="AI 扩图"
    :footer="false"
    @cancel="closeModal"
  >
    <a-row gutter="16">
      <a-col span="12">
        <h4>原始图片</h4>
        <img :src="picture?.url" :alt="picture?.name" style="max-width: 100%" />
      </a-col>
      <a-col span="12">
        <h4>扩图结果</h4>
        <img
          v-if="resultImageUrl"
          :src="resultImageUrl"
          :alt="picture?.name"
          style="max-width: 100%"
        />
      </a-col>
    </a-row>
    <div style="margin-bottom: 16px" />
    <a-flex justify="center" gap="16">
      <a-button type="primary" :loading="!!taskId" ghost @click="createTask">生成图片</a-button>
      <a-button v-if="resultImageUrl" type="primary" :loading="uploadLoading" @click="handleUpload">
        应用结果
      </a-button>
    </a-flex>
  </a-modal>
</template>

<script lang="ts" setup>
import { ref } from 'vue'
import {
  createPictureOutPaintingTaskUsingPost,
  getPictureOutPaintingTaskUsingGet,
  uploadPictureByUrlUsingPost,
} from '@/api/pictureController.ts'
import { message, notification } from 'ant-design-vue'

interface Props {
  picture?: API.PictureVO
  spaceId?: number
  onSuccess?: (newPicture: API.PictureVO) => void
}

const props = defineProps<Props>()

const resultImageUrl = ref<string>('')

// 任务 id
const taskId = ref<string>()

/**
 * 创建任务
 */
// const createTask = async () => {
//   if (!props.picture?.id) {
//     message.warning('图片数据不完整')
//     return
//   }
//   const res = await createPictureOutPaintingTaskUsingPost({
//     pictureId: props.picture.id,
//     // 根据需要设置扩图参数
//     parameters: {
//       xScale: 2,
//       yScale: 2,
//     },
//   })
//   if (res.data.code === 0 && res.data.data) {
//     message.success('创建任务成功，请耐心等待，不要退出界面')
//     console.log(res.data.data.output.taskId)
//     taskId.value = res.data.data.output.taskId
//     // 开启轮询
//     startPolling()
//   } else {
//     message.error('图片任务失败，' + res.data.message)
//   }
// }

/**
 * 创建扩图任务（阿里云百炼智能防爆版最优解）
 */
const createTask = async () => {
  if (!props.picture?.id || !props.picture?.url) {
    message.warning('图片数据不完整')
    return
  }

  // 1. 利用浏览器 Image 对象，抓取网络图片的绝对真实物理像素
  const img = new Image()
  img.crossOrigin = 'anonymous' // 防止跨域导致无法读取
  img.src = props.picture.url

  // 开启加载提示
  taskId.value = 'loading_detector'

  img.onload = async () => {
    const realWidth = img.width
    const realHeight = img.height

    console.log(`[AI扩图探测] 当前原图绝对尺寸: ${realWidth} × ${realHeight}`)

    // 2. 前置卡点：如果原图本身就超标，直接拦截，省去后端排队时间
    if (realWidth < 512 || realWidth > 4096 || realHeight < 512 || realHeight > 4096) {
      alert(`${realWidth}×${realHeight}`)
      notification.warning({
        message: '图片无法执行扩图',
        description: `当前图片物理尺寸为 ${realWidth}×${realHeight}。\n阿里云模型要求原图长宽单边必须在 512px 到 4096px 之间。`,
        style: { whiteSpace: 'pre-line' },
      })
      taskId.value = null // 释放按钮 loading
      return
    }

    // 3. 🎯 核心计算：动态计算最高安全放大倍数（确保放大后的单边绝对不超过 4096）
    const maxWScale = 4096 / realWidth
    const maxHScale = 4096 / realHeight

    // 阿里云允许的倍数范围是 [1.0, 2.0]，所以上限卡在 2.0
    let safeScale = Math.min(maxWScale, maxHScale, 2.0)
    // 保留两位小数，防止出现无限循环小数导致接口报错
    safeScale = Math.floor(safeScale * 100) / 100

    // 4. 如果图片太大（比如单边 3900px），计算出安全倍数可能逼近 1.0，说明没有扩图空间了
    if (safeScale <= 1.0) {
      notification.info({
        message: '图片已达上限',
        description: `当前图片尺寸已达 ${realWidth}×${realHeight}，已经接近阿里云 4096px 的上限值，无法进一步向外扩图。`,
      })
      taskId.value = null
      return
    }

    console.log(`[AI扩图智能推荐] 已为您匹配最安全、清晰度最高的扩图倍数: ${safeScale} 倍`)

    // 5. 将安全倍数发送给后端
    try {
      const res = await createPictureOutPaintingTaskUsingPost({
        pictureId: props.picture.id,
        parameters: {
          xScale: safeScale, // 🎯 智能自适应倍数
          yScale: safeScale, // 🎯 智能自适应倍数
        },
      })

      if (res.data.code === 0 && res.data.data) {
        message.success('创建任务成功，请耐心等待，不要退出界面')
        taskId.value = res.data.data.output.taskId
        startPolling()
      } else {
        taskId.value = null
        message.error('图片任务失败，' + res.data.message)
      }
    } catch (apiError) {
      taskId.value = null
      message.error('发送扩图请求失败')
    }
  }

  img.onerror = () => {
    taskId.value = null
    message.error('解析原图失败，请检查图片网络地址是否有效')
  }
}

// 轮询定时器
let pollingTimer: NodeJS.Timeout = null

// 开始轮询
const startPolling = () => {
  if (!taskId.value) {
    return
  }

  pollingTimer = setInterval(async () => {
    try {
      const res = await getPictureOutPaintingTaskUsingGet({
        taskId: taskId.value,
      })
      if (res.data.code === 0 && res.data.data) {
        const taskResult = res.data.data.output
        if (taskResult.taskStatus === 'SUCCEEDED') {
          message.success('扩图任务执行成功')
          resultImageUrl.value = taskResult.outputImageUrl
          // 清理轮询
          clearPolling()
        } else if (taskResult.taskStatus === 'FAILED') {
          // message.error('扩图任务执行失败')
          // // 清理轮询
          // clearPolling()
          // ==================== 【修改后的 FAILED 拦截分支】====================
          // 提取后端透传的第三方模型核心错误码与具体信息
          const errCode = taskResult.code
          const errMsg = taskResult.message

          // 精准捕获 COS 模型的尺寸/分辨率不合规错误
          if (errCode === 'InvalidParameter.ImageResolution') {
            notification.error({
              message: 'AI 扩图失败：图片尺寸不合规',
              description: `系统检测到您的图片无法执行扩图，请参考以下 阿里云百炼 模型限制：

                1. 📐 分辨率范围：不低于 512 × 512 像素且不超过 4096 × 4096 像素。
                2. 📏 单边长度：图像的宽、高单边长度都必须在 [512, 4096] 像素范围内。
                3. 💾 图像大小：图片文件体积不能超过 10MB。
                4. 🖼️ 支持格式：JPG、JPEG、PNG、HEIF、WEBP。`,
              style: {
                whiteSpace: 'pre-line', // 🎯 关键：保证里面的换行符能被解析出来
              },
              duration: 8, // 保持 8 秒让用户看清技术限制
            })
          } else {
            // 其他未定义的 FAILED 错误，走通用报错提示
            message.error('扩图任务执行失败：' + (errMsg || '未知系统异常'))
          }

          // 清理轮询状态
          clearPolling()
        }
      }
    } catch (error) {
      console.error('扩图任务轮询失败', error)
      message.error('扩图任务轮询失败，' + error.message)
      // 清理轮询
      clearPolling()
    }
  }, 3000) // 每 3 秒轮询一次
}

// 清理轮询
const clearPolling = () => {
  if (pollingTimer) {
    clearInterval(pollingTimer)
    pollingTimer = null
    taskId.value = null
  }
}

// 是否正在上传
const uploadLoading = ref(false)

/**
 * 上传图片
 * @param file
 */
const handleUpload = async () => {
  uploadLoading.value = true
  try {
    const params: API.PictureUploadRequest = {
      fileUrl: resultImageUrl.value,
      spaceId: props.spaceId,
    }
    if (props.picture) {
      params.id = props.picture.id
    }
    const res = await uploadPictureByUrlUsingPost(params)
    if (res.data.code === 0 && res.data.data) {
      message.success('图片上传成功')
      // 将上传成功的图片信息传递给父组件
      props.onSuccess?.(res.data.data)
      // 关闭弹窗
      closeModal()
    } else {
      message.error('图片上传失败，' + res.data.message)
    }
  } catch (error) {
    console.error('图片上传失败', error)
    message.error('图片上传失败，' + error.message)
  }
  uploadLoading.value = false
}

// 是否可见
const visible = ref(false)

// 打开弹窗
const openModal = () => {
  visible.value = true
}

// 关闭弹窗
const closeModal = () => {
  visible.value = false
  clearPolling()
}

// 暴露函数给父组件
defineExpose({
  openModal,
})
</script>

<style>
.image-out-painting {
  text-align: center;
}
</style>
