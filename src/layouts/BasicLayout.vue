<template>
  <div id="basicLayout">
    <a-layout style="min-height: 100vh">
      <a-layout-header class="header">
        <GlobalHeader />
      </a-layout-header>

      <a-layout>
        <GlobalSider class="sider" />
        <a-layout-content class="content">
          <router-view />
        </a-layout-content>
      </a-layout>
      <a-layout-footer class="footer">
        <a href="https://github.com/gameover111/SmartPic#" target="_blank">
          智图云 (SmartPic),基于 Vue 3+Spring Boot+COS+WebSocket
          的企业级智能协同云图库平台。————韩淑成
        </a>
      </a-layout-footer>
    </a-layout>
  </div>
</template>

<script setup lang="ts">
import GlobalHeader from '@/components/GlobalHeader.vue'
import GlobalSider from '@/components/GlobalSider.vue'
</script>

<style scoped>
#basicLayout .header {
  padding-inline: 20px;
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  color: unset;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.04);
  position: sticky;
  top: 0;
  z-index: 100;
}

#basicLayout .sider {
  background: linear-gradient(180deg, #fafbfc 0%, #f5f5f7 100%);
  border-right: none;
  box-shadow: 1px 0 4px rgba(0, 0, 0, 0.03);
  padding-top: 20px;
}

#basicLayout :deep(.ant-menu-root) {
  border-bottom: none !important;
  border-inline-end: none !important;
  background: transparent;
}

#basicLayout .content {
  padding: 24px;
  background: linear-gradient(135deg, #fafbfc 0%, #f5f5f7 50%, #fafbfc 100%);
  min-height: calc(100vh - 64px - 69px);
}

#basicLayout .footer {
  background: linear-gradient(90deg, #f5f5f7, #fafbfc, #f5f5f7);
  border-top: 1px solid rgba(0, 0, 0, 0.06);
  padding: 16px;
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  text-align: center;
  color: #999;
  font-size: 13px;
}

#basicLayout .footer a {
  color: #999;
  transition: color 0.3s;
}

#basicLayout .footer a:hover {
  color: #555;
}
</style>
