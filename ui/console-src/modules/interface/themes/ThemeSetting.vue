<script lang="ts" setup>
import StickyBlock from "@/components/sticky-block/StickyBlock.vue";
import type { FormKitSchemaCondition, FormKitSchemaNode } from "@formkit/core";
import type { Setting, Theme } from "@halo-dev/api-client";
import { consoleApiClient } from "@halo-dev/api-client";
import { Toast, VButton } from "@halo-dev/components";
import { useQuery, useQueryClient } from "@tanstack/vue-query";
import { cloneDeep, set } from "lodash-es";
import type { Ref } from "vue";
import { computed, inject, ref, toRaw } from "vue";
import { useI18n } from "vue-i18n";

const { t } = useI18n();
const queryClient = useQueryClient();

const group = inject<Ref<string>>("activeTab", ref(""));

const selectedTheme = inject<Ref<Theme | undefined>>("selectedTheme");
const setting = inject<Ref<Setting | undefined>>("setting", ref());

const saving = ref(false);

const { data: configMapData, suspense } = useQuery({
  queryKey: ["core:theme:configMap:data", selectedTheme],
  queryFn: async () => {
    const { data } = await consoleApiClient.theme.theme.fetchThemeJsonConfig({
      name: selectedTheme?.value?.metadata.name as string,
    });
    return data;
  },
  enabled: computed(() => {
    return !!setting.value && !!selectedTheme?.value;
  }),
});

const currentConfigMapGroupData = computed(() => {
  return configMapData.value?.[group.value];
});

const formSchema = computed(() => {
  if (!setting.value) {
    return;
  }
  const { forms } = setting.value.spec;
  return forms.find((item) => item.group === group?.value)?.formSchema as (
    | FormKitSchemaCondition
    | FormKitSchemaNode
  )[];
});

const handleSaveConfigMap = async (data: object) => {
  saving.value = true;

  if (!selectedTheme?.value) {
    saving.value = false;
    return;
  }

  await consoleApiClient.theme.theme.updateThemeJsonConfig({
    name: selectedTheme?.value?.metadata.name,
    body: set(cloneDeep(configMapData.value) || {}, group.value, data),
  });

  Toast.success(t("core.common.toast.save_success"));

  queryClient.invalidateQueries({ queryKey: ["core:theme:configMap:data"] });

  saving.value = false;
};

await suspense();
</script>
<template>
  <div class="p-4">
    <FormKit
      v-if="group && formSchema && currentConfigMapGroupData"
      :id="group"
      :value="currentConfigMapGroupData || {}"
      :name="group"
      :preserve="true"
      type="form"
      @submit="handleSaveConfigMap"
    >
      <FormKitSchema
        :schema="toRaw(formSchema)"
        :data="toRaw(currentConfigMapGroupData)"
      />
    </FormKit>
    <StickyBlock
      v-permission="['system:themes:manage']"
      class="-mx-4 -mb-4 rounded-b-base rounded-t-lg bg-white p-4 pt-5"
      position="bottom"
    >
      <VButton
        :loading="saving"
        type="secondary"
        @click="$formkit.submit(group || '')"
      >
        {{ $t("core.common.buttons.save") }}
      </VButton>
    </StickyBlock>
  </div>
</template>
