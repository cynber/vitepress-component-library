<template>
  <div class="image-gallery-container">
    <div v-if="format === 'debug'">
      <pre>{{ galleryData }}</pre>
    </div>

    <component :is="containerComponent" class="gallery-container">
      <TitleCard :title="title" :date="date" :title-lines="titleLines" :link="link" />
      <div v-if="sortedImageUrls.length === 0" class="no-images">
        No images found for this gallery.
      </div>
      <div class="image-grid" :id="galleryId" v-else>
        <ImageCardSquare
          v-for="(img, index) in sortedImageUrls"
          :key="index"
          :image="img"
        />
      </div>
    </component>
  </div>
</template>

<script setup lang="ts">
import { inject, computed, onMounted, onUnmounted, ref, nextTick, watch } from "vue";
import { withBase } from "vitepress";
import "photoswipe/style.css";
import VerticalContainer from "./containers/VerticalContainer.vue";
import ImageCardSquare from "./cards/ImageCardSquare.vue";
import TitleCard from "./cards/HeaderCard.vue";

interface GalleryImage {
  path: string;
  folder: string;
  filename: string;
}

interface ImageGalleryProps {
  title: string;
  titleLines?: number;
  date: string;
  link?: string;
  folders?: string[];
  images?: string[];
  excludeExtensions?: string[];
  includeExtensions?: string[];
  format?: "debug";
  galleryDataKey?: string;
  forceSort?: string[];
}

const galleryId = ref(`gallery-${Math.random().toString(36).substr(2, 9)}`);
let lightbox: any = null;

const initPhotoSwipe = async () => {
  if (lightbox) {
    lightbox.destroy();
    lightbox = null;
  }

  await nextTick();

  const { default: PhotoSwipeLightbox } = await import("photoswipe/lightbox");

  setTimeout(() => {
    try {
      lightbox = new PhotoSwipeLightbox({
        gallery: `#${galleryId.value}`,
        children: "a",
        pswpModule: () => import("photoswipe"),
      });
      lightbox.init();
    } catch (error) {
      console.error("PhotoSwipe initialization error:", error);
    }
  }, 100);
};

const props = withDefaults(defineProps<ImageGalleryProps>(), {
  titleLines: 2,
});

const galleryData = inject<GalleryImage[]>(props.galleryDataKey || "galleryData", []);

onMounted(() => {
  initPhotoSwipe();
});

onUnmounted(() => {
  if (lightbox) {
    lightbox.destroy();
    lightbox = null;
  }
});

const formattedDate = computed(() => {
  const options: Intl.DateTimeFormatOptions = {
    year: "numeric",
    month: "long",
    day: "numeric",
  };
  const dateObj = new Date(props.date);
  return isNaN(dateObj.getTime())
    ? props.date
    : dateObj.toLocaleDateString(undefined, options);
});

const containerComponent = computed(() => VerticalContainer);

const processedFolders = computed(() => {
  return props.folders?.map((folder) => withBase(folder)) || [];
});

const filteredImages = computed(() => {
  if (!processedFolders.value.length && !props.images?.length) {
    return galleryData.filter((image) => {
      const ext = image.filename.split(".").pop()?.toLowerCase();
      if (props.excludeExtensions?.length && ext && props.excludeExtensions.includes(ext))
        return false;
      if (
        props.includeExtensions?.length &&
        ext &&
        !props.includeExtensions.includes(ext)
      ) {
        return false;
      }
      return true;
    });
  }

  return galleryData.filter((image) => {
    const ext = image.filename.split(".").pop()?.toLowerCase();
    if (props.excludeExtensions?.length && ext && props.excludeExtensions.includes(ext))
      return false;
    if (props.includeExtensions?.length && ext && !props.includeExtensions.includes(ext))
      return false;

    const inFolder = processedFolders.value.includes(image.folder);
    const inImages = props.images?.includes(image.path) || false;

    return inFolder || inImages;
  });
});

const sortedImageUrls = computed(() => {
  if (props.forceSort && props.forceSort.length > 0) {
    const sorted = [...props.forceSort];
    filteredImages.value.forEach((img) => {
      if (!sorted.includes(img.path)) {
        sorted.push(img.path);
      }
    });
    return sorted;
  }

  return filteredImages.value.map((img) => img.path).sort();
});

watch(sortedImageUrls, () => {
  initPhotoSwipe();
});
</script>

<style scoped>
.image-gallery-container {
  margin: 1rem auto;
  display: flex;
  flex-direction: column;
}

.gallery-container {
  display: flex;
  flex-direction: column;
  width: 100%;
}

.image-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 1rem;
}

.no-images {
  text-align: center;
  color: var(--vp-c-text-3);
  padding: 2rem;
  font-size: 1.2rem;
}
</style>
