<script lang="ts" setup>
import { useAppTitle } from "@/composables/use-title";
import { i18n } from "@/locales";
import { useGlobalInfoStore } from "@/stores/global-info";
import type { FormKitConfig } from "@formkit/core";
import { useFavicon } from "@vueuse/core";
import type { OverlayScrollbars } from "overlayscrollbars";
import {
  useOverlayScrollbars,
  type UseOverlayScrollbarsParams,
} from "overlayscrollbars-vue";
import { storeToRefs } from "pinia";
import { computed, inject, onMounted, provide, reactive } from "vue";
import { RouterView } from "vue-router";

useAppTitle();

const { globalInfo } = storeToRefs(useGlobalInfoStore());

// Favicon
const defaultFavicon = "/console/favicon.ico";
const favicon = computed(() => {
  return globalInfo.value?.favicon || defaultFavicon;
});

useFavicon(favicon);

// body scroll
const body = document.querySelector("body");
const reactiveParams = reactive<UseOverlayScrollbarsParams>({
  options: {
    scrollbars: {
      autoHide: "scroll",
      autoHideDelay: 600,
    },
  },
  defer: true,
});
const [initialize, instance] = useOverlayScrollbars(reactiveParams);
onMounted(() => {
  if (body) initialize({ target: body });
});

provide<() => OverlayScrollbars | null>("bodyScrollInstance", instance);

// setup formkit locale
// see https://formkit.com/essentials/internationalization
const formkitLocales = {
  en: "en",
  zh: "zh",
  "en-US": "en",
  "zh-CN": "zh",
};
const formkitConfig = inject(Symbol.for("FormKitConfig")) as FormKitConfig;
formkitConfig.locale = formkitLocales[i18n.global.locale.value] || "zh";

// Fix 100vh issue in ios devices
function setViewportProperty(doc: HTMLElement) {
  let prevClientHeight: number;
  const customVar = "--vh";

  function handleResize() {
    const clientHeight = doc.clientHeight;
    if (clientHeight === prevClientHeight) return;
    requestAnimationFrame(function updateViewportHeight() {
      doc.style.setProperty(customVar, clientHeight * 0.01 + "px");
      prevClientHeight = clientHeight;
    });
  }

  handleResize();
  return handleResize;
}

window.addEventListener(
  "resize",
  setViewportProperty(document.documentElement)
);
</script>

<template>
  <RouterView />
</template>

<style lang="scss">
body {
  background: #eff4f9;
}

html,
body,
#app {
  min-height: 100vh;
}

*,
*::before,
*::after {
  box-sizing: border-box;
}

.v-popper__popper {
  outline: none;
}

.v-popper--theme-tooltip {
  pointer-events: none;
}
</style>
