<script lang="ts" setup>
import {useMemorizeRequest} from '@/hook/useMemorizeRequest';
import {getMvList} from '@/service';
import {useThemeVars} from 'naive-ui';
import {nextTick, reactive, ref, toRaw, watch} from 'vue';
import CategoryTab from '@/components/Base/CategoryTab.vue';
import MvListSkeleton from '@/components/MvList/MvListSkeleton.vue';
import MvList from '@/components/MvList/MvList.vue';

const areaList = [
  '全部', '内地', '港台', '欧美', '日本', '韩国'
];
const typeList = [
  '全部', '官方版', '现场版', '原生', '网易出品'
];
const orderList = ['最热', '最新', '上升最快'];


const params = reactive({
  area: areaList[0],
  type: typeList[0],
  order: orderList[0]
});
const list = ref([]);
const firstFetchDataSuccess = ref(false);

const pageParams = reactive({
  pageCount: 10,
  page: 1,
  pageSize: 50
});
const { wrapRequest, loadSuccess, requestLoading: listIsLoading } = useMemorizeRequest(getMvList, 'getMvList');
const themeVars = useThemeVars();
const fetchList = (setPageCount = true) => {
  let data = getParams();
  list.value = [];
  wrapRequest(data).then((res: { data: { data: never[]; count: number; }; }) => {
    list.value = res.data.data;
    if (!firstFetchDataSuccess.value) {
      firstFetchDataSuccess.value = true;
    }
    if (setPageCount) {
      if (res.data.count) {
        pageParams.pageCount = Math.round(res.data.count / pageParams.pageSize);
      } else {
        pageParams.pageCount = 50;
      }
    }
    loadSuccess();
  });
};

const getParams = () => {
  return {
    ...toRaw(params),
    limit: pageParams.pageSize,
    offset: ((pageParams.page) - 1) * pageParams.pageSize
  };
};

fetchList();

watch(params, () => {
  fetchList();
});

watch([() => pageParams.page, () => pageParams.pageSize], async () => {
  fetchList(false);
  // 回到顶部
  await nextTick();
  (document.querySelector('.n-back-top') as HTMLElement).click();
});

</script>

<template>
  <div :style="{background:themeVars.bodyColor}" class="overflow-hidden p-6 musicList">
    <n-space
      :size="20" :style="{background:themeVars.bodyColor}" class="sticky top-0  py-4"
      vertical
    >
      <category-tab v-model="params.area" :list="areaList" name="地区" />
      <category-tab v-model="params.type" :list="typeList" name="类型" />
      <category-tab v-model="params.order" :list="orderList" name="排序" />
    </n-space>
    <div class="py-4">
      <MvListSkeleton v-if="listIsLoading" :count="pageParams.pageSize" />
      <mv-list v-if="!listIsLoading" :list="list" />
      <div v-if="firstFetchDataSuccess" class="flex justify-end mt-4">
        <n-pagination
          v-model:page="pageParams.page"
          v-model:page-size="pageParams.pageSize"
          :page-count="pageParams.pageCount"
          :page-sizes="[10, 20, 30, 40,50]"
          show-size-picker
        />
      </div>
    </div>
  </div>
</template>
<style scoped>
.n-layout .n-layout-scroll-container::-webkit-scrollbar-thumb {
    background-color: transparent;
}

:deep(.n-layout .n-layout-scroll-container) {
    overflow: hidden;
}
</style>
