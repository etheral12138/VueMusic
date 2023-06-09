<script lang="ts" setup>
import StopIcon from '@/components/Icon/StopIcon.vue';
import OrderPlay from '@/components/Icon/OrderPlay.vue';
import RandomIcon from '@/components/Icon/RandomIcon.vue';
import SingleLoop from '@/components/Icon/SingleLoop.vue';
import { formatSongsAuthor } from '@/utils';
import { List } from '@vicons/ionicons5';
import {
  AddBoxOutlined,
  KeyboardArrowDownOutlined,
  KeyboardArrowUpOutlined,
  PlayArrowSharp,
  SkipNextSharp,
  SkipPreviousSharp,
  VolumeOffRound,
  VolumeUpRound
} from '@vicons/material';
import { useThemeVars } from 'naive-ui';
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue';
import { useMainStore } from '@/stores/main';
import dayjs from 'dayjs';
import type { PlayListExpose } from './PlayList.vue';
import { useElementHover } from '@vueuse/core';
import observer from '@/utils/observer';
import type { HeartIconExpose } from '../common/HeartIcon.vue';
import { useAudioLoadProgress } from './hook/useAudioLoadProgress';
import HeartIcon from '@/components/common/HeartIcon.vue';
import PlayList from '@/components/Player/PlayList.vue';
import MusicDetail from '@/components/Player/MusicDetail.vue';
import SubscribePlayListModal from '@/components/common/SubscribePlayListModal.vue';
import SliderBar from '@/components/Base/SliderBar.vue';

let slideValueChange = false;// 记录slider值是否手动发生了改变
let triggerOriginalAudioTimeUpdate = true;
const progressWidth = 500;
const themeVars = useThemeVars();
const mainStore = useMainStore();
// 触发显示打开歌曲详情交互元素
const triggerRef = ref();
const isHover = useElementHover(triggerRef);
const heardLikeRef = ref<HeartIconExpose>();
const subscribeModalRef = ref<{ show: () => void }>();
let isLoad = false;
// audio元素
const audioRef = ref<HTMLAudioElement>();
const { updateBuffer, progressValue } = useAudioLoadProgress(audioRef, mainStore.currentPlaySong?.dt / 1000);
// 进度条百分比
const percentage = ref(0);
// 当前播放时间
const currentPlayTime = ref('00:00');
// 音量大小
const volume = ref(+localStorage.volume || 100);
// 播放列表组件ref
const playListRef = ref<PlayListExpose>();
//computed
const primaryColor = computed(() => themeVars.value.primaryColor);
const currentSong = computed(() => mainStore.currentPlaySong);
const isShow = computed(() => !!mainStore.playList.length);
const currentPlayModeIcon = computed(() => {
  if (mainStore.playMode === 'order') {
    return OrderPlay;
  } else if (mainStore.playMode === 'random') {
    return RandomIcon;
  } else {
    return SingleLoop;
  }
});
const activeStyle = computed(() => {
  let transformStyle;
  if (!mainStore.showMusicDetail) {
    transformStyle = 'translateY(0)';
  } else {
    transformStyle = 'translateY(-54px)';
  }
  return { transform: transformStyle };
});
const loadCurrentPrevAndNext = async (val: any) => {
  if (!val) return;
  // 加载上一首和下一首的歌曲url
  let next = mainStore.playList[val.nextIndex] as any;
  let prev = mainStore.playList[val.prevIndex] as any;
  if (!next.url) {
    await mainStore.setMusicData({ data: mainStore.playList, id: next.id, index: val.nextIndex, showMessage: false });
  }
  if (!prev.url) {
    await mainStore.setMusicData({ data: mainStore.playList, id: next.id, index: val.prevIndex, showMessage: false });
  }
  localStorage.playList = JSON.stringify(mainStore.playList);
};
watch(
  () => mainStore.currentPlaySong, (val, oldVal) => {
    loadCurrentPrevAndNext(val);
    if (oldVal && val.id !== oldVal.id) {
      // 重新加载媒体资源
      audioRef.value?.load();
      resetState();
      tryPlay();
    }
  }, { immediate: true }
);
watch(() => mainStore.playList, (val) => {
  if (val.length === 0) {
    percentage.value = 0;
  }
});
watch(() => mainStore.playing, (val) => {
  if (val) {
    tryPlay();
  } else {
    audioRef.value?.pause();
  }
});
// 点击切换上一首
const handlePrevClick = async () => {
  if (isLoad) return;
  isLoad = true;
  await mainStore.togglePrev();
  isLoad = false;
};
// 点击切换下一首
const handleNextClick = async () => {
  if (isLoad) return;
  isLoad = true;
  await mainStore.toggleNext();
  isLoad = false;
};
const resetState = () => {
  currentPlayTime.value = '00:00';
  percentage.value = 0;
  progressValue.value = 0;
    audioRef.value!.currentTime = 0;
};
// 切换播放状态
const togglePlayStatus = async () => {
  if (audioRef.value?.paused) {
    tryPlay();
  } else {
    audioRef.value?.pause();
    mainStore.changePlaying(false);
  }
};
const tryPlay = () => {
  if (audioRef.value && audioRef.value!.readyState >= 2 && audioRef.value?.paused) {
    audioRef.value?.play();
  }
  mainStore.changePlaying(true);
};
const handleEnded = () => {
  // 如果为单曲循环模式,则重新播放
  if (mainStore.playMode === 'singleLoop') {
        audioRef.value!.currentTime = 0;
        audioRef.value?.play();
  } else {
    mainStore.toggleNext();
  }
  observer.emit('ended');
};
// 播放进度变化
const handleTimeupdate = (event: Event) => {
  if (triggerOriginalAudioTimeUpdate) {
    const target = event.target as HTMLAudioElement;
    updatePlayTime(target.currentTime);
  }
};

const updatePlayTime = async (time: number, triggerPlay = false) => {
  // 如果当前滑动条正在改变,则不设置对应的值, 避免冲突
  if (!slideValueChange) {
    currentPlayTime.value = dayjs(time * 1000).format('mm:ss');
    percentage.value = Math.round(((time * 1000) / currentSong.value?.dt) * 100);
  }
  if (triggerPlay) {
    tryPlay();
    await nextTick();
    if (audioRef.value) {
            audioRef.value!.currentTime = time;
    }
  }
  observer.emit('timeUpdate', Math.round(time));
};
// 媒体的第一帧加载完成
const handleLoadeddata = () => {
  if (mainStore.playing && audioRef.value?.paused) {
    audioRef.value?.play();
  }
};
//处理数据还未加载完成时,播放暂停
const handleWaiting = () => {
  mainStore.playWaiting = true;
};
// 因为缺少数据而暂停或延迟的状态结束，播放准备开始
const handlePlaying = () => {
  mainStore.playWaiting = false;
};
const handleError = async () => {
  // 媒体资源过期,重新获取数据
  if (audioRef.value?.error!.code === 4 || audioRef.value?.error!.code === 2) {
    window.$message.warning('歌曲资源过期,准备尝试重新获取');
    if (isLoad) return;
    isLoad = true;
    const res = await mainStore.setMusicData({
      data: mainStore.playList,
      id: mainStore.currentPlaySong.id,
      index: mainStore.currentPlayIndex
    });
    localStorage.playList = JSON.stringify(mainStore.playList);
    isLoad = false;
    // 重新加载资源
    if (res.success) {
      audioRef.value?.load();
      resetState();
      if (mainStore.playing) {
        audioRef.value?.play();
      }
    }
  }
};
// 处理鼠标在进度条上抬起或者按下操作
const handleSliderDone = () => {
  triggerOriginalAudioTimeUpdate = false;
  let currentTime = (currentSong.value?.dt * percentage.value) / 100;
  currentPlayTime.value = dayjs(currentTime).format('mm:ss');
    audioRef.value!.currentTime = currentTime / 1000;
    slideValueChange = false;
    observer.emit(
      'slideValueChange', Math.round(currentTime / 1000), true
    );
};
const handleSliderChange = () => {
  slideValueChange = true;
  let currentTime = (currentSong.value?.dt * percentage.value) / 100;
  currentPlayTime.value = dayjs(currentTime).format('mm:ss');
};
// 音量滑动选择器监听回调
const handleVolumeChange = (value: number) => {
  localStorage.volume = value;
  volume.value = value;
    audioRef.value!.volume = volume.value / 100;
};
// 点击音量选择,切换静音
const handleVolumeClick = () => {
  if (volume.value === 100) {
    volume.value = 0;
    localStorage.volume = 0;
  } else {
    volume.value = 100;
    localStorage.volume = 100;
  }
    audioRef.value!.volume = volume.value / 100;
};
// 点击切换播放模式
const handlePlayModeClick = () => {
  const playMode = mainStore.playMode;
  if (playMode === 'order') {
    mainStore.changePlayMode('random');
    window.$message.info('随机播放');
  } else if (playMode === 'random') {
    mainStore.changePlayMode('singleLoop');
    window.$message.info('单曲循环');
  } else {
    mainStore.changePlayMode('order');
    window.$message.info('顺序播放');
  }
};
// 点击空格播放
const handlePressSpace = (e: KeyboardEvent) => {
  // e.preventDefault();
  if (e.code === 'Space' && mainStore.currentPlaySong) {
    togglePlayStatus();
  }
};
// 点击箭头打开歌曲详情
const handleArrowClick = () => {
  mainStore.toggleShowMusicDetail();
};
const likeSuccess = (like: boolean) => {
  mainStore.updatePlayListLike(like);
};
const handleLikeHeartClick = () => {
  heardLikeRef.value?.triggerLike();
};
onMounted(() => {
  document.body.addEventListener('keypress', handlePressSpace);
  observer.on('selectLyricPlay', (time) => {
    updatePlayTime(time, true);
  });
  observer.on('scrollComplete', () => {
    triggerOriginalAudioTimeUpdate = true;
  });
});
onUnmounted(() => {
  document.body.removeEventListener('keypress', handlePressSpace);
});
</script>
<template>
  <div class="flex relative z-30 items-center p-2 footer-player">
    <div v-if="isShow" class="overflow-hidden w-60 h-12">
      <div :style="activeStyle" class="open-detail-control-wrap">
        <div class="flex items-center w-40 h-full">
          <div ref="triggerRef" class="relative" @click="handleArrowClick">
            <n-image
              :preview-disabled="true"
              :src="currentSong?.al?.picUrl"
              :style="{filter:isHover ? 'blur(1px)' : 'none'}"
              class="w-12 h-12"
            />
            <transition v-show="isHover" name="fade">
              <div class="absolute top-0 left-0 z-10 w-12  h-12 bg-black/60 flex-items-justify-center">
                <n-icon :component="KeyboardArrowUpOutlined" color="white" size="35" />
              </div>
            </transition>
          </div>
          <div class="ml-4">
            <p class="flex items-center text-base">
              <n-ellipsis style="max-width: 150px">
                {{ currentSong?.name }}
              </n-ellipsis>
              <heart-icon
                :id="mainStore.currentPlaySong.id" :like="mainStore.currentPlaySong.like"
                class="ml-2"
                @like-success="likeSuccess"
              />
            </p>
            <n-ellipsis>
              <p>{{ formatSongsAuthor(currentSong?.ar || []) }}</p>
            </n-ellipsis>
          </div>
        </div>
        <div class="flex items-center ml-1 h-12">
          <n-icon
            :component="KeyboardArrowDownOutlined" size="45"
            @click="mainStore.setShowMusicDetail(false)"
          />
          <div class="ml-2">
            <div class="circleContainer" @click="handleLikeHeartClick">
              <heart-icon
                :id="mainStore.currentPlaySong.id"
                ref="heardLikeRef" :like="mainStore.currentPlaySong.like"
                :size="25" :trigger-click="true" @like-success="likeSuccess"
              />
            </div>
          </div>
          <div class="ml-2 circleContainer" @click="subscribeModalRef?.show()">
            <n-icon :component="AddBoxOutlined" :size="20" />
          </div>
        </div>
      </div>
    </div>
    <div :style="{opacity:isShow ? '1' : '0.6'}" class="flex flex-col flex-1 items-center control">
      <div v-if="!isShow" class="absolute bottom-0 z-50 w-full footer-player" />
      <div class="flex justify-between items-center" style="width:300px">
        <n-icon
          :component="currentPlayModeIcon" :size="22" class="custom-icon"
          @click="handlePlayModeClick"
        />
        <n-icon
          :component="SkipPreviousSharp" :size="22" class="prev custom-icon"
          @click="handlePrevClick"
        />
        <div
          class="flex justify-center items-center w-8 h-8  bg-neutral-200/60 hover:bg-neutral-200 dark:bg-slate-100/20 dark:hover:bg-slate-100/40 rounded-full"
          @click="togglePlayStatus"
        >
          <n-icon
            :component="mainStore.playing ? StopIcon :PlayArrowSharp"
            :size="mainStore.playing ? 14 : 20"
          />
        </div>
        <n-icon
          :component="SkipNextSharp" :size="22" class="next custom-icon"
          @click="handleNextClick"
        />
      </div>
      <div class="flex items-center mt-1">
        <span v-if="isShow" class="mr-2 text-xs opacity-50">{{ currentPlayTime }}</span>
        <div :style="{width:progressWidth+'px'}" class="flex flex-1 items-center">
          <slider-bar
            v-model="percentage"
            :load-value="progressValue"
            @change="handleSliderChange"
            @on-done="handleSliderDone"
          />
        </div>
        <span v-if="isShow" class="ml-2 text-xs opacity-50">
          <n-time :time="currentSong?.dt" format="mm:ss" />
        </span>
      </div>
    </div>
    <div v-if="isShow" class="flex justify-end items-center w-60">
      <n-popover
        placement="bottom"
        trigger="hover"
      >
        <template #trigger>
          <n-icon
            :component="volume === 0 ? VolumeOffRound : VolumeUpRound" :size="25"
            class="mr-2 custom-icon"
            @click="handleVolumeClick"
          />
        </template>
        <n-slider
          :value="volume" style="height:100px" vertical
          @update-value="handleVolumeChange"
        />
      </n-popover>
      <n-icon
        :component="List" :size="25" class="mr-2 custom-icon"
        @click="playListRef?.show()"
      />
    </div>
    <audio
      ref="audioRef"
      :src="currentSong?.url"
      preload="auto" @ended="handleEnded" @error="handleError"
      @loadeddata="handleLoadeddata" @playing="handlePlaying" @progress="updateBuffer"
      @timeupdate="handleTimeupdate" @waiting="handleWaiting"
    />
    <play-list ref="playListRef" />
    <music-detail v-if="mainStore.currentPlaySong?.id" ref="musicDetailRef" />
    <subscribe-play-list-modal
      v-if="mainStore.currentPlaySong?.id" ref="subscribeModalRef"
      :tracks="mainStore.currentPlaySong?.id"
    />
  </div>
</template>
<style scoped>
.footer-player {
    height: calc(100vh - 858px);
    box-sizing: border-box;
}

:deep(.custom-icon:hover) {
    color: v-bind(primaryColor);
}

:deep(.n-icon) {
    cursor: pointer;
}

.open-detail-control-wrap {
    transition: transform .6s ease;
}

.circleContainer {
    @apply w-10 h-10 rounded-full border
    border-gray-200
    dark:border-gray-200/30 border-solid
    hover:bg-gray-100
    dark:hover:bg-gray-100/20
    flex-items-justify-center;
}
</style>
