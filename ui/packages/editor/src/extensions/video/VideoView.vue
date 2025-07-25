<script lang="ts" setup>
import { i18n } from "@/locales";
import type { NodeViewProps } from "@/tiptap/vue-3";
import { NodeViewWrapper } from "@/tiptap/vue-3";
import { computed, onMounted, ref } from "vue";

const props = defineProps<NodeViewProps>();

const src = computed({
  get: () => {
    return props.node?.attrs.src;
  },
  set: (src: string) => {
    props.updateAttributes({ src: src });
  },
});

const controls = computed(() => {
  return props.node.attrs.controls;
});

const autoplay = computed(() => {
  return props.node.attrs.autoplay;
});

const loop = computed(() => {
  return props.node.attrs.loop;
});

function handleSetFocus() {
  props.editor.commands.setNodeSelection(props.getPos());
}

const inputRef = ref();

onMounted(() => {
  if (!src.value) {
    inputRef.value.focus();
  }
});
</script>

<template>
  <node-view-wrapper as="div" class="inline-block w-full">
    <div
      class="relative inline-block h-full max-w-full overflow-hidden text-center transition-all"
      :style="{
        width: node.attrs.width,
      }"
    >
      <div v-if="!src" class="p-1.5">
        <input
          ref="inputRef"
          v-model.lazy="src"
          class="block w-full rounded-md border border-gray-300 bg-gray-50 px-2 py-1.5 text-sm text-gray-900 focus:border-blue-500 focus:ring-blue-500"
          :placeholder="i18n.global.t('editor.common.placeholder.link_input')"
          tabindex="-1"
          @focus="handleSetFocus"
        />
      </div>
      <video
        v-else
        :controls="controls"
        :autoplay="autoplay"
        :loop="loop"
        class="m-0 rounded-md"
        :src="node!.attrs.src"
        :style="{
          width: node.attrs.width,
          height: node.attrs.height,
        }"
        @mouseenter="handleSetFocus"
      ></video>
    </div>
  </node-view-wrapper>
</template>
