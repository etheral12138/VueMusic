<template>
  <n-modal v-model:show="showModal" :mask-closable="false" transform-origin="center">
    <n-card
      :bordered="false"
      aria-modal="true"
      preset="dialog"
      size="small"
      style="width: 350px;height: 420px;"
    >
      <template #header>
        <div class="flex justify-end ">
          <n-icon
            :component="CloseOutline" class="cursor-pointer" size="25"
            @click="showModal = false"
          />
        </div>
      </template>
      <div v-if="status !== 802" class="flex flex-col justify-between items-center">
        <p class="mt-5 text-3xl">
          扫码登录
        </p>
        <!-- 二维码图片 -->
        <n-spin :show="isLoadingQrCodeImg">
          <div class="relative mt-5">
            <img
              v-show="!isLoadingQrCodeImg"
              :src="qrCodeImg" height="200"
              width="200"
            >
            <!-- 图片加载时占位符 -->
            <div v-show="isLoadingQrCodeImg" style="width:200px;height:200px" />

            <!-- 二维码过期蒙层 -->
            <div
              v-if="status === 800"
              class="flex absolute inset-0 justify-center items-center bg-black/90"
            >
              <div class="text-white">
                <p>
                  二维码已失效
                </p>
                <n-button
                  class="mt-4" size="small" type="primary"
                  @click="handleRefreshClick"
                >
                  点击刷新
                </n-button>
              </div>
            </div>
          </div>
        </n-spin>
        <p class="mt-6">
          <span>使用</span>
          <span class="text-sky-500">网易云音乐APP</span>
          <span>扫码登录</span>
        </p>
      </div>
      <!-- 扫码待确认登录 -->
      <div v-if="status === 802" class="mt-20">
        <n-result
          description="请在手机上确认登录" size="small" status="success"
          title="扫码成功"
        />
      </div>
    </n-card>
  </n-modal>
</template>

<script lang="ts" setup>
import { onUnmounted, ref, watch } from 'vue';
import { CloseOutline } from '@vicons/ionicons5';
import { getQrCode, getQrCodeImg, getQrCodeStatus } from '@/service';
import { useMainStore } from '@/stores/main';

export interface LoginModalExpose {
    show: () => void;
    close: () => void;
}

type qrCodeStatus = 800 | 801 | 802 | 803;
let qrCodeKey = '';
let timer: number | undefined;
const showModal = ref(false);
const status = ref<qrCodeStatus | ''>();
let qrCodeImg = ref('');
const isLoadingQrCodeImg = ref(true);
const mainStore = useMainStore();
defineExpose({
  show() {
    showModal.value = true;
  },
  close() {
    showModal.value = false;
  }
});
const getQsCodeKey = () => {
  isLoadingQrCodeImg.value = true;
  getQrCode().then(res => {
    qrCodeKey = res.data.data.unikey;
    getQrCodePic();
  });
};
const getQrCodePic = () => {
  getQrCodeImg(qrCodeKey).then(res => {
    qrCodeImg.value = res.data.data.qrimg;
    isLoadingQrCodeImg.value = false;
  });
  loopGetQrCodeStatus();
};
const loopGetQrCodeStatus = () => {
  timer = setInterval(async () => {
    const res = await getQrCodeStatus(qrCodeKey);
    // 避免当定时器清空时,多余的回调执行
    if (timer === undefined) return;
    status.value = res.data.code;
    if (res.data.code === 803) {
      clearInterval(timer);
      window.$message.success('授权登录成功');
      showModal.value = false;
      status.value = '';
      localStorage.isLogin = true;
      mainStore.isLogin = true;
      timer = undefined;
      return;
    }
    if (res.data.code === 800) {
      clearInterval(timer);
      window.$message.warning('二维码已过期,请重新获取');
      timer = undefined;
    }
  }, 1000);
};
const handleRefreshClick = () => {
  qrCodeImg.value = '';
  status.value = '';
  getQsCodeKey();
};
watch(showModal, (val) => {
  if (val) {
    getQsCodeKey();
  } else {
    clearInterval(timer);
  }
});
onUnmounted(() => {
  clearInterval(timer);
});
</script>
<style scoped>

</style>