<template>
  <div id="globalSider">
    <!-- <a-layout-sider
      v-if="loginUserStore.loginUser.id"
      width="200"
      breakpoint="lg"
      collapsed-width="0"
    >
      <a-menu
        v-model:selectedKeys="current"
        mode="inline"
        :items="menuItems"
        @click="doMenuClick"
      />
    </a-layout-sider> -->
    <a-layout-sider
      v-if="loginUserStore.loginUser.id"
      v-model:collapsed="collapsed"
      width="180"
      :collapsible="true"
      :trigger="null"
      collapsed-width="64"
      theme="light"
      style="position: relative"
    >
      <div class="sider-trigger-wrapper">
        <a-button type="text" size="small" @click="() => (collapsed = !collapsed)">
          <template #icon>
            <menu-fold-outlined v-if="!collapsed" />
            <menu-unfold-outlined v-else />
          </template>
        </a-button>
      </div>

      <a-menu
        v-model:selectedKeys="current"
        mode="inline"
        :items="menuItems"
        @click="doMenuClick"
      />
    </a-layout-sider>
  </div>
</template>
<script lang="ts" setup>
import { computed, h, ref, watchEffect } from 'vue'
import { PictureOutlined, TeamOutlined, UserOutlined } from '@ant-design/icons-vue'
import { useRouter } from 'vue-router'
import { useLoginUserStore } from '@/stores/useLoginUserStore.ts'
import { SPACE_TYPE_ENUM } from '@/constants/space.ts'
import { listMyTeamSpaceUsingPost } from '@/api/spaceUserController.ts'
import { message } from 'ant-design-vue'
// 如果需要手动引入图标组件（按需引入项目适用，全局引入可忽略这两行）
import { MenuFoldOutlined, MenuUnfoldOutlined } from '@ant-design/icons-vue'

// ✨ 定义控制侧边栏折叠的响应式变量
const collapsed = ref<boolean>(false)

const loginUserStore = useLoginUserStore()

// 固定的菜单列表
const fixedMenuItems = [
  {
    key: '/',
    icon: () => h(PictureOutlined),
    label: '公共图库',
  },
  {
    key: '/my_space',
    label: '我的空间',
    icon: () => h(UserOutlined),
  },
  {
    key: '/add_space?type=' + SPACE_TYPE_ENUM.TEAM,
    label: '创建团队',
    icon: () => h(TeamOutlined),
  },
]

const teamSpaceList = ref<API.SpaceUserVO[]>([])
const menuItems = computed(() => {
  // 如果用户没有团队空间，则只展示固定菜单
  if (teamSpaceList.value.length < 1) {
    return fixedMenuItems
  }
  // 如果用户有团队空间，则展示固定菜单和团队空间菜单
  // 展示团队空间分组
  const teamSpaceSubMenus = teamSpaceList.value.map((spaceUser) => {
    const space = spaceUser.space
    return {
      key: '/space/' + spaceUser.spaceId,
      label: space?.spaceName,
    }
  })
  const teamSpaceMenuGroup = {
    type: 'group',
    label: '我的团队',
    key: 'teamSpace',
    children: teamSpaceSubMenus,
  }
  return [...fixedMenuItems, teamSpaceMenuGroup]
})

// 加载团队空间列表
const fetchTeamSpaceList = async () => {
  const res = await listMyTeamSpaceUsingPost()
  if (res.data.code === 0 && res.data.data) {
    teamSpaceList.value = res.data.data
  } else {
    message.error('加载我的团队空间失败，' + res.data.message)
  }
}

/**
 * 监听变量，改变时触发数据的重新加载
 */
watchEffect(() => {
  // 登录才加载
  if (loginUserStore.loginUser.id) {
    fetchTeamSpaceList()
  }
})

const router = useRouter()
// 当前要高亮的菜单项
const current = ref<string[]>([])
// // 监听路由变化，更新高亮菜单项
// router.afterEach((to, from, next) => {
//   current.value = [to.path]
// })

// 监听路由变化，更新高亮菜单项
router.afterEach((to, from, next) => {
  // 🎯 核心修复：如果进入了具体的空间详情页（/space/xxx），依然高亮“我的空间”
  if (to.path.startsWith('/space/')) {
    current.value = ['/my_space']
  } else {
    current.value = [to.path]
  }
})

// 路由跳转事件
const doMenuClick = ({ key }) => {
  router.push(key)
}
</script>

<style scoped>
#globalSider .ant-layout-sider {
  background: none;
}

/* 让侧边栏作为绝对定位的参照物 */
:deep(.ant-layout-sider) {
  position: relative;
}

/* 控制折叠按钮悬浮在侧边栏右边缘外侧 */
.sider-trigger-wrapper {
  position: absolute;
  top: -20px;
  right: -24px;
  z-index: 1001;
}

/* 折叠按钮 - 灰色风格 */
.sider-trigger-wrapper .ant-btn {
  background-color: rgba(0, 0, 0, 0.03);
  color: #666;
  border: 1px solid rgba(0, 0, 0, 0.08);
  border-left: none;
  border-radius: 0 6px 6px 0;
  box-shadow: 2px 0 8px rgba(0, 0, 0, 0.06);

  height: 24px;
  width: 24px;
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.3s;
}

.sider-trigger-wrapper .ant-btn:hover {
  background-color: rgba(0, 0, 0, 0.06);
  color: #333;
  border-color: rgba(0, 0, 0, 0.15);
}

/* 侧边栏菜单样式优化 */
:deep(.ant-menu-inline) {
  padding-top: 16px;
  border-inline-end: none !important;
}

:deep(.ant-menu-item) {
  border-radius: 8px;
  margin: 2px 8px;
  transition: all 0.25s ease;
}

:deep(.ant-menu-item:hover) {
  background: rgba(0, 0, 0, 0.04) !important;
  color: #333 !important;
}

:deep(.ant-menu-item-selected) {
  background: rgba(0, 0, 0, 0.06) !important;
  color: #333 !important;
  font-weight: 500;
}

:deep(.ant-menu-item-group-title) {
  font-size: 12px;
  color: #aaa;
  padding: 12px 16px 4px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
</style>
