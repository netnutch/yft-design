<template>
  <div>
    <div class="category-container" ref="categoryRef" @scroll="onCategoryScroll" v-if="typeRef === 'all'">
      <div v-for="(item, index) in illustrationCategoryData" :key="index">
        <el-row class="col-tip mt-5">
          <el-col :span="5" class="col-name">
            <el-tag>{{ item.name }}</el-tag>
          </el-col>
          <el-col :span="7" class="col-name">
            <el-button text @click="showTotal(item.type)">{{ $t("message.all") }}<IconRight/></el-button>
          </el-col>
        </el-row>
        <el-row class="category-box mt-5" v-loading="item.category.length === 0">
          <el-col :span="8" class="box-image" v-for="(img, index) in item.category" :key="index">
            <img :src="img.previewURL" :alt="img.tags" @click="createImage(img)"/>
          </el-col>
        </el-row>
      </div>
    </div>
    <div class="category-container" ref="totalRef" @scroll="onTotalScroll" v-else>
      <el-row class="col-tip mt-5">
        <el-col :span="7" class="col-name">
          <el-button text @click="hideTotal()"><IconLeft />{{ categoryData.name }}</el-button>
        </el-col>
      </el-row>
      <el-row class="total-box mt-5" v-loading="categoryData.total.length === 0">
        <div class="box-image" v-for="(img, index) in categoryData.total" :key="index" :style="{ justifyContent: index % 2 === 0 ? 'flex-start' : 'flex-end'}">
          <img :src="img.previewURL" :alt="img.tags" @click="createImage(img)"/>
        </div>
      </el-row>
    </div>
    <el-row class="image-bottom">{{ $t("message.endOfContent") }}</el-row>
  </div>
</template>

<script lang="ts" setup>
import { onMounted, ref, computed } from "vue";
import { debounce, throttle } from "lodash-es";
import { getIllustrationPages, getIllustrationCategory } from "@/api/image";
import { ImageHit } from "@/api/image/types";
import { useMainStore } from "@/store";
import { storeToRefs } from "pinia";
import { util } from "fabric";
import { GifImage } from '@/extension/object/GifImage'
import useHandleCreate from "@/hooks/useHandleCreate";
import useCanvas from "@/views/Canvas/useCanvas";
import { Image } from "fabric";
import useCenter from "@/views/Canvas/useCenter";
const mainStore = useMainStore();
const { illustrationCategoryType, illustrationCategoryData } = storeToRefs(mainStore);
const { createImageElement } = useHandleCreate();

const categoryRef = ref<HTMLDivElement>();
const totalRef = ref<HTMLDivElement>();
const categoryTop = ref(0);
const typeRef = ref("all");
const categoryData = computed(() => {
  return illustrationCategoryData.value.filter(
    (ele) => ele.type === typeRef.value
  )[0];
});
const getImageCategoryData = throttle(
  async (t: string) => {
    const res = await getIllustrationCategory({ t });
    if (res && res.data.code === 200) {
      illustrationCategoryData.value
        .filter((item) => item.type === t)
        .map((ele) => (ele.category = res.data.data));
    }
  },
  100,
  { leading: true, trailing: false }
);

const getImagePageData = throttle(
  async (t: string, page: 1) => {
    const res = await getIllustrationPages({ t, page });
    if (res && res.data.code === 200) {
      illustrationCategoryData.value
        .filter((item) => item.type === t)
        .map((ele) => (ele.total = res.data.data));
    }
  },
  100,
  { leading: true, trailing: false }
);

const getContainScroll = () => {
  let startIndex = 0,
    endIndex = 2;
  if (!categoryRef.value)
    return {
      startIndex,
      endIndex,
    };
  const scrollTop = categoryRef.value.scrollTop;
  const containerHeight = categoryRef.value.clientHeight;
  const itemHeight = 132;
  startIndex = Math.floor(scrollTop / itemHeight);
  endIndex = Math.ceil((scrollTop + containerHeight) / itemHeight);
  return {
    startIndex,
    endIndex,
  };
};

const onCategoryScroll = async () => {
  const { startIndex, endIndex } = getContainScroll();
  for (let i = startIndex; i < endIndex; i++) {
    const item = illustrationCategoryData.value[i];
    if (!item) return;
    if (!illustrationCategoryType.value.includes(item.type)) {
      illustrationCategoryType.value.push(item.type);
      await getImageCategoryData(item.type);
    }
    // if (item.category.length === 0) {
    //   await getImageCategoryData(item.type)
    // }
  }
};

const onTotalScroll = async () => {
  const { startIndex, endIndex } = getContainScroll();
  for (let i = startIndex; i < endIndex; i++) {
    const item = illustrationCategoryData.value[i];
    if (!item) return;
    if (!illustrationCategoryType.value.includes(item.type)) {
      illustrationCategoryType.value.push(item.type);
      await getImageCategoryData(item.type);
    }
    // if (item.category.length === 0) {
    //   await getImageCategoryData(item.type)
    // }
  }
};

const showTotal = async (type: string) => {
  if (!categoryRef.value) return;
  categoryTop.value = categoryRef.value.scrollTop;
  typeRef.value = type;
  await getImagePageData(type, 1);
};

const hideTotal = () => {
  typeRef.value = "all";
  if (!categoryRef.value) return;
  categoryRef.value.scrollTo({ top: categoryTop.value, behavior: "smooth" });
};

const hideLoading = async (item: ImageHit, loading: Image) => {
  const [ canvas ] = useCanvas()
  await util.loadImage(item.largeImageURL)
  loading.set({visible: false})
  canvas.renderAll()
}

const createImage = async (item: ImageHit) => {
  const [ canvas ] = useCanvas()
  const { centerPoint } = useCenter()
  let loading = canvas.loading
  if (!loading) {
    loading = await GifImage.fromURL(new URL(`/src/assets/images/loading.gif`, import.meta.url).href)
    loading.set({left: centerPoint.x - loading.width / 2, top: centerPoint.y - loading.height / 2})
    canvas.add(loading);
    canvas.renderAll()
    canvas.loading = loading
  }
  else {
    loading.set({visible: true})
    canvas.bringObjectToFront(loading)
    canvas.renderAll()
  }
  await hideLoading(item, loading)
  createImageElement(item.largeImageURL);
};

onMounted(() => {
  if (!categoryRef.value) return;
  onCategoryScroll();
});
</script>

<style lang="scss" scoped>
.col-tip {
  justify-content: space-between;
  align-items: center;
}
.col-name {
  text-align: center;
}
.mt-5 {
  margin-top: 5px;
}
.category-box {
  align-items: center;
  flex-wrap: nowrap;
  overflow: hidden;
  height: 100px;
  .box-image {
    display: flex;
    align-items: center;
    padding: 0 2px;
    &:first-child {
      justify-content: flex-start;
    }
    &:last-child {
      justify-content: flex-end;
    }
    img {
      max-width: 100%;
      cursor: pointer;
    }
  }
}

.category-container {
  overflow-y: scroll;
  height: 100vh;
  align-items: center;
}
.total-box {
  .box-image {
    padding: 2px;
    width: 48%;
    height: 120px;
    overflow: hidden;
    display: flex;
    img {
      max-width: 100%;
    }
  }
}
.image-bottom {
  justify-content: center;
  padding-top: 20px;
  margin-bottom: 130px;
}
</style>