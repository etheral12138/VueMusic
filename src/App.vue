<script lang="ts" setup>
import LayOut from '@/components/Layout/Layout.vue';
import { zhCN } from 'naive-ui';
import { useMainStore } from './stores/main';
import { theme } from '@/main';

const mainStore = useMainStore();

// onMounted(() => {
//   mainStore.initDocumentTheme();
//   let media = window.matchMedia('(prefers-color-scheme: dark)');
//   media.addEventListener('change', (e) => {
//     mainStore.changeTheme(e.matches
//       ?'dark'
//       :'light');
//   });
// });
</script>
<template>
  <n-config-provider :locale="zhCN" :theme="theme as any">
    <n-loading-bar-provider>
      <div>
        <n-message-provider>
          <n-dialog-provider>
            <LayOut />
          </n-dialog-provider>
        </n-message-provider>
      </div>
    </n-loading-bar-provider>
  </n-config-provider>
</template>

<style>
:root {
    --theme-color: #ffffff;
}

::-webkit-scrollbar {
    width: 5px;
    height: 5px;
    background-color: transparent;
}

::-webkit-scrollbar-thumb {
    width: 5px;
    @apply bg-gray-400/40;
    border-radius: 10px;
}
</style>
