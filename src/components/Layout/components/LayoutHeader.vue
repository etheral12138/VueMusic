<script lang="ts" setup>
import {useMainStore} from '@/stores/main';
import {Moon, SunnySharp} from '@vicons/ionicons5';
import {ArrowForwardIosRound, ExitToAppRound} from '@vicons/material';
import {UserProfile} from '@vicons/carbon';
import {ref, watch} from 'vue';
import {onClickOutside} from '@vueuse/core';
import {getLoginStatus, getUserDetail, getUserInfo, logout, signIn} from '@/service';
import type {AnyObject} from 'env';
import {useRouter} from 'vue-router';
import LayoutHeaderSearch from '@/components/Layout/components/LayoutHeaderSearch.vue';
import {theme} from '@/main';
import {darkTheme} from 'naive-ui';

let mainStore = useMainStore();
const router = useRouter();
const popoverContainerRef = ref();
const userDetail = ref<AnyObject>();
const showUserPopover = ref(false);
const signBtnLoading = ref(false);

const changeTheme = () => {
  theme.value = theme.value == null
    ? darkTheme
    : null;
};
// 监听登录状态 获取用户信息
watch(() => mainStore.isLogin, (val) => {
  if (val) {
    getUserProfile();
  }
});
onClickOutside(popoverContainerRef, (event: MouseEvent) => {
  let target = event.target as HTMLElement;
  // 如果点击的不是不是触发弹出选择的元素
  if (!target.classList.contains('trigger')) {
    showUserPopover.value = false;
  }
});
// 获取用户账号数据
const getUserProfile = () => {
  // 如果已经设置过,则从store读取userId
  if (mainStore.userProfile?.userId) {
    getUserDetailInfo(mainStore.userProfile.profile?.userId);
  } else {
    getUserInfo().then(res => {
      getUserDetailInfo(res.data.profile?.userId);
    });
  }
};
// 获取用户详情数据
const getUserDetailInfo = (uid: string) => {
  getUserDetail(uid).then((res) => {
    if (res?.data?.code === 200) {
      mainStore.userProfile = res.data;
      localStorage.userProfile = JSON.stringify(res.data);
      userDetail.value = res.data;
    }
  });
};
const checkLoginStatus = () => {
  getLoginStatus().then(res => {
    if (res.data?.data?.code === 200) {
      if (!res.data.data.account) {
        window.$message.warning('登录状态过期,请重新登录');
        mainStore.userProfile = {};
        localStorage.clear();
        mainStore.isLogin = false;
      }
    }
  });
};
const handlePositiveClick = () => {
  window.$message.loading('退出登录中...', { duration: 0 });
  logout().then(res => {
    if (res.data.code === 200) {
      mainStore.isLogin = false;
      mainStore.userProfile = {};
      localStorage.isLogin = false;
      localStorage.clear();
      mainStore.removeAllLikeList();
      window.$message.destroyAll();
      window.$message.success('退出登录成功!');
    }
  });
};
const handleInfoEditClick = () => {
  showUserPopover.value = false;
  router.push('/userInfoEdit');
};
const handleSignInClick = () => {
  signBtnLoading.value = true;
  signIn().then(() => {
    if (userDetail.value) {
      signBtnLoading.value = false;
      userDetail.value.pcSign = true;
      window.$message.success('签到成功!');
    }
  });
};
// const handleThemeSwitchUpdateChange = () => {
//   mainStore.toggleTheme();
// };
const BackToDiscovery = () => {
  router.push('/discovery');
};
if (mainStore.isLogin) {
  getUserProfile();
  checkLoginStatus();
}
</script>
<template>
  <n-layout-header bordered class="flex justify-between items-center px-4 h-14 sm:px-3">
    <div class="flex leading-7">
      <img alt="" class="w-7 h-7" src="https://etheral.oss-cn-shanghai.aliyuncs.com/images/mao.png">
      <span class="ml-2 text-xl italic truncate cursor-pointer" @click="BackToDiscovery">EMusic</span>
      <layout-header-search />
    </div>
    <div class="flex items-center">
      <!-- 用户信息入口 -->
      <div v-if="mainStore.isLogin">
        <div v-if="mainStore.userProfile" class="flex items-center mr-2">
          <n-avatar :size="30" :src="mainStore.userProfile?.profile?.avatarUrl" round />
          <n-popover
            :show="showUserPopover"
            :style="{'background-color':theme!==null?'rgb(16,16,20)':'white'}"
            animated
            display-directive="show"
            trigger="click"
          >
            <template #trigger>
              <p
                class="pl-2 text-xs truncate opacity-80 hover:opacity-100 cursor-pointer w-30 trigger"
                @click="() => (userDetail && (showUserPopover = !showUserPopover ))
                "
              >
                {{ mainStore.userProfile?.profile?.nickname }}
              </p>
            </template>
            <div ref="popoverContainerRef" style="width:200px">
              <div class="flex justify-evenly py-4">
                <div class="flex flex-col items-center">
                  <p class="text-base font-bold">
                    {{ mainStore.userProfile?.profile?.eventCount }}
                  </p>
                  动态
                </div>
                <div>
                  <p class="text-base font-bold">
                    {{ mainStore.userProfile?.profile?.follows }}
                  </p>
                  关注
                </div>
                <div>
                  <p class="text-base font-bold">
                    {{ mainStore.userProfile?.profile?.followeds }}
                  </p>
                  粉丝
                </div>
                <n-button
                  :disabled="mainStore.userProfile.pcSign" :loading="signBtnLoading" round
                  @click="handleSignInClick"
                >
                  {{ mainStore.userProfile.pcSign ? '已签到' : ' 签到' }}
                </n-button>
              </div>
              <!--              <div class="flex justify-center">-->

              <!--              </div>-->
              <div
                class="mt-2 hover:bg-neutral-200/20 border-0 border-b border-gray-200  dark:border-gray-200/20 border-solid"
              >
                <!-- 个人信息设置 -->
                <div
                  class="flex justify-between items-center py-2 px-4 cursor-pointer"
                  @click="handleInfoEditClick"
                >
                  <div class="flex items-center text-base">
                    <n-icon :component="UserProfile" :size="20" />
                    <span class="ml-2">个人信息设置</span>
                  </div>
                  <n-icon :component="ArrowForwardIosRound" />
                </div>
              </div>
              <div
                class="hover:bg-neutral-200/20 border-0 border-b border-gray-200  dark:border-gray-200/20 border-solid"
              >
                <n-popconfirm
                  @positive-click="handlePositiveClick"
                >
                  <template #trigger>
                    <!-- 个人信息设置 -->
                    <div class="flex justify-between items-center py-2 px-4 cursor-pointer">
                      <div class="flex items-center text-base">
                        <n-icon :component="ExitToAppRound" :size="20" />
                        <span class="ml-2">退出登录</span>
                      </div>
                      <n-icon :component="ArrowForwardIosRound" />
                    </div>
                  </template>
                  确认退出登录吗?
                </n-popconfirm>
              </div>
            </div>
          </n-popover>
        </div>
        <!-- 用户详情加载骨架屏 -->
        <div v-else class="flex items-center mr-2">
          <n-skeleton height="30px" round width="30px" />
          <n-skeleton style="width:100px;margin-left: 8px;" text />
        </div>
      </div>
      <div class="flex items-center ml-2">
        <!--        <n-switch :on-update:value='mainStore.toggleTheme' :value='mainStore.isActiveDarkTheme' size='large'>-->
        <!--          <template #checked-icon>-->
        <!--            <n-icon :component='Moon' />-->
        <!--          </template>-->
        <!--          <template #unchecked-icon>-->
        <!--            <n-icon :component='SunnySharp' />-->
        <!--          </template>-->
        <!--        </n-switch>-->
        <n-config-provider :theme="theme as any">
          <n-switch :on-update:value="changeTheme" :value="theme!==null" size="large">
            <template #checked-icon>
              <n-icon :component="Moon" />
            </template>
            <template #unchecked-icon>
              <n-icon :component="SunnySharp" />
            </template>
          </n-switch>
          <!--                        <n-button @click="changetheme">&ndash;&gt;-->
          <!--                             {{ themebtntit }}&ndash;&gt;-->
          <!--                         </n-button>&ndash;&gt;-->
          <n-global-style />
        </n-config-provider>
      </div>
    </div>
  </n-layout-header>
</template>
<style scoped>
:deep(.n-popover:not(.n-popover--raw):not(.n-popover--show-header)) {
    padding: 0 !important;
}
</style>
