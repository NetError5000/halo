<script lang="ts" setup>
import { computed, inject, onMounted, ref, watch } from "vue";
import { IconErrorWarning } from "../../icons/icons";
import { AvatarGroupContextInjectionKey, type AvatarProps } from "./interface";

const props = withDefaults(defineProps<AvatarProps>(), {
  size: "md",
  circle: false,
});

const groupProps = inject(AvatarGroupContextInjectionKey, undefined);

const size = computed(() => groupProps?.size || props.size);
const circle = computed(() => groupProps?.circle || props.circle);
const width = computed(() => groupProps?.width || props.width);
const height = computed(() => groupProps?.height || props.height);

const isLoading = ref(false);
const error = ref(false);
let init = true;

const loadImage = async (isInit: boolean) => {
  if (!props.src) {
    error.value = true;
    return;
  }

  isLoading.value = true;
  try {
    if (!props.src) {
      error.value = true;
      return Promise.reject();
    }
    if (!isInit) {
      error.value = false;
    }
    const image = new Image();
    image.src = props.src;
    return new Promise((resolve, reject) => {
      image.onload = () => resolve(image);
      image.onerror = (err) => {
        error.value = true;
        reject(err);
      };
    });
  } catch (_) {
    error.value = true;
  } finally {
    isLoading.value = false;
  }
};

watch([() => props.alt, () => props.src], async () => loadImage(init));

onMounted(async () => {
  loadImage(init);
  init = false;
});

const classes = computed(() => {
  const result = [`avatar-${circle.value ? "circle" : "square"}`];
  if (size.value) {
    result.push(`avatar-${size.value}`);
  }
  return result;
});

const styles = computed(() => {
  const result: Record<string, string> = {};
  if (width.value) {
    result.width = width.value;
  }
  if (height.value) {
    result.height = height.value;
  }
  return result;
});

const placeholderText = computed(() => {
  if (!props.alt) {
    return undefined;
  }
  const words = props.alt.split(" ");
  if (words.length === 1) {
    return words[0].charAt(0).toUpperCase();
  }
  if (words.length > 1) {
    return words[0].charAt(0).toUpperCase() + words[1].charAt(0).toUpperCase();
  }
  return undefined;
});
</script>

<template>
  <div class="avatar-wrapper" :class="classes" :style="styles">
    <div v-if="isLoading || error" class="avatar-fallback">
      <svg
        v-if="isLoading"
        class="avatar-loading"
        fill="none"
        viewBox="0 0 24 24"
        xmlns="http://www.w3.org/2000/svg"
      >
        <circle
          class="opacity-25"
          cx="12"
          cy="12"
          r="10"
          stroke="currentColor"
          stroke-width="4"
        ></circle>
        <path
          class="opacity-75"
          d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
          fill="currentColor"
        ></path>
      </svg>
      <span v-else-if="placeholderText" class="avatar-placeholder">
        {{ placeholderText }}
      </span>
      <IconErrorWarning v-else class="avatar-error" />
    </div>
    <img v-else :src="src" :alt="alt" />
  </div>
</template>

<style lang="scss">
.avatar-wrapper {
  @apply inline-flex items-center justify-center overflow-hidden bg-gray-100;

  img {
    @apply h-full w-full object-cover;
  }

  .avatar-fallback {
    @apply flex h-full w-full items-center justify-center;
  }

  .avatar-loading {
    @apply h-5 w-5 animate-spin;
  }

  .avatar-placeholder {
    @apply text-sm font-medium text-gray-800;
  }

  .avatar-error {
    @apply h-5 w-5 text-red-500;
  }

  &.avatar-circle {
    @apply rounded-full;
  }

  &.avatar-square {
    @apply rounded-base;
  }

  &.avatar-xs {
    @apply h-6 w-6;

    .avatar-placeholder {
      @apply text-xs;
    }
  }

  &.avatar-sm {
    @apply h-8 w-8;

    .avatar-placeholder {
      @apply text-xs;
    }
  }

  &.avatar-md {
    @apply h-10 w-10;
  }

  &.avatar-lg {
    @apply h-12 w-12;
  }
}
</style>
