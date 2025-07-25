<script lang="ts" setup>
import { rbacAnnotations } from "@/constants/annotations";
import { pluginLabels, roleLabels } from "@/constants/labels";
import { formatDatetime } from "@/utils/date";
import { usePermission } from "@/utils/permission";
import {
  PluginStatusPhaseEnum,
  coreApiClient,
  type Plugin,
  type Role,
} from "@halo-dev/api-client";
import {
  VAlert,
  VButton,
  VDescription,
  VDescriptionItem,
  VSwitch,
} from "@halo-dev/components";
import { useQuery } from "@tanstack/vue-query";
import type { Ref } from "vue";
import { computed, inject, ref } from "vue";
import { usePluginLifeCycle } from "../../composables/use-plugin";
import PluginConditionsModal from "../PluginConditionsModal.vue";

const { currentUserHasPermission } = usePermission();

const plugin = inject<Ref<Plugin | undefined>>("plugin");
const { changeStatus, changingStatus } = usePluginLifeCycle(plugin);

interface RoleTemplateGroup {
  module: string | null | undefined;
  roles: Role[];
}

const { data: pluginRoleTemplates } = useQuery({
  queryKey: ["plugin-roles", plugin?.value?.metadata.name],
  queryFn: async () => {
    const { data } = await coreApiClient.role.listRole({
      page: 0,
      size: 0,
      labelSelector: [
        `${pluginLabels.NAME}=${plugin?.value?.metadata.name}`,
        `${roleLabels.TEMPLATE}=true`,
        "!halo.run/hidden",
      ],
    });

    return data.items;
  },
  cacheTime: 0,
  enabled: computed(
    () =>
      !!plugin?.value?.metadata.name &&
      currentUserHasPermission(["system:roles:view"])
  ),
});

const pluginRoleTemplateGroups = computed<RoleTemplateGroup[]>(() => {
  const groups: RoleTemplateGroup[] = [];
  pluginRoleTemplates.value?.forEach((role) => {
    const group = groups.find(
      (group) =>
        group.module === role.metadata.annotations?.[rbacAnnotations.MODULE]
    );
    if (group) {
      group.roles.push(role);
    } else {
      groups.push({
        module: role.metadata.annotations?.[rbacAnnotations.MODULE],
        roles: [role],
      });
    }
  });
  return groups;
});

// Error alert
const conditionsModalVisible = ref(false);

const errorAlertVisible = computed(() => {
  const { phase } = plugin?.value?.status || {};

  return (
    phase !== PluginStatusPhaseEnum.Started &&
    phase !== PluginStatusPhaseEnum.Disabled
  );
});

const lastCondition = computed(() => {
  return plugin?.value?.status?.conditions?.[0];
});
</script>

<template>
  <PluginConditionsModal
    v-if="conditionsModalVisible && plugin"
    :plugin="plugin"
    @close="conditionsModalVisible = false"
  />
  <div class="overflow-hidden rounded-b-base">
    <div class="flex items-center justify-between bg-white px-4 py-4 sm:px-6">
      <div>
        <h3 class="text-lg font-medium leading-6 text-gray-900">
          {{ $t("core.plugin.detail.header.title") }}
        </h3>
      </div>
      <div v-permission="['system:plugins:manage']">
        <VSwitch
          :loading="changingStatus"
          :model-value="plugin?.spec.enabled"
          @change="changeStatus"
        />
      </div>
    </div>
    <div
      v-if="errorAlertVisible && lastCondition"
      class="w-full px-4 pb-2 sm:px-6"
    >
      <VAlert type="error" :title="lastCondition.reason" :closable="false">
        <template #description>
          <div class="overflow-x-auto">
            <pre>{{ lastCondition.message }}</pre>
          </div>
        </template>
        <template #actions>
          <VButton size="sm" @click="conditionsModalVisible = true">
            {{ $t("core.plugin.detail.operations.view_conditions.button") }}
          </VButton>
        </template>
      </VAlert>
    </div>
    <div class="border-t border-gray-200">
      <VDescription>
        <VDescriptionItem label="ID" :content="plugin?.metadata.name" />
        <VDescriptionItem
          :label="$t('core.plugin.detail.fields.description')"
          :content="plugin?.spec.description || $t('core.common.text.none')"
        />
        <VDescriptionItem :label="$t('core.plugin.detail.fields.author')">
          <a
            v-if="plugin?.spec.author"
            :href="plugin?.spec.author.website"
            class="hover:text-gray-600"
            target="_blank"
          >
            {{ plugin?.spec.author.name }}
          </a>
          <span v-else>
            {{ $t("core.common.text.none") }}
          </span>
        </VDescriptionItem>
        <VDescriptionItem
          :label="$t('core.plugin.detail.fields.version')"
          :content="plugin?.spec.version"
        />
        <VDescriptionItem
          :label="$t('core.plugin.detail.fields.requires')"
          :content="plugin?.spec.requires"
        />
        <VDescriptionItem :label="$t('core.plugin.detail.fields.homepage')">
          <a
            v-if="plugin?.spec.homepage"
            :href="plugin?.spec.homepage"
            class="hover:text-gray-600"
            target="_blank"
          >
            {{ plugin?.spec.homepage }}
          </a>
          <span v-else>
            {{ $t("core.common.text.none") }}
          </span>
        </VDescriptionItem>
        <VDescriptionItem :label="$t('core.plugin.detail.fields.repo')">
          <a
            v-if="plugin?.spec.repo"
            :href="plugin.spec.repo"
            class="hover:text-gray-600"
            target="_blank"
          >
            {{ plugin.spec.repo }}
          </a>
          <span v-else>
            {{ $t("core.common.text.none") }}
          </span>
        </VDescriptionItem>
        <VDescriptionItem :label="$t('core.plugin.detail.fields.issues')">
          <a
            v-if="plugin?.spec.issues"
            :href="plugin.spec.issues"
            class="hover:text-gray-600"
            target="_blank"
          >
            {{ plugin.spec.issues }}
          </a>
          <span v-else>
            {{ $t("core.common.text.none") }}
          </span>
        </VDescriptionItem>
        <VDescriptionItem :label="$t('core.plugin.detail.fields.license')">
          <ul
            v-if="plugin?.spec.license && plugin?.spec.license.length"
            class="list-inside"
            :class="{ 'list-disc': plugin?.spec.license.length > 1 }"
          >
            <li v-for="(license, index) in plugin.spec.license" :key="index">
              <a v-if="license.url" :href="license.url" target="_blank">
                {{ license.name }}
              </a>
              <span v-else>
                {{ license.name }}
              </span>
            </li>
          </ul>
          <span v-else>
            {{ $t("core.common.text.none") }}
          </span>
        </VDescriptionItem>
        <VDescriptionItem
          :label="$t('core.plugin.detail.fields.role_templates')"
        >
          <dl
            v-if="pluginRoleTemplateGroups.length"
            class="divide-y divide-gray-100"
          >
            <div
              v-for="(group, groupIndex) in pluginRoleTemplateGroups"
              :key="groupIndex"
              class="rounded bg-gray-50 px-4 py-5 hover:bg-gray-100 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6"
            >
              <dt class="text-sm font-medium text-gray-900">
                {{ group.module }}
              </dt>
              <dd class="mt-1 text-sm text-gray-900 sm:col-span-2 sm:mt-0">
                <ul class="space-y-2">
                  <li v-for="(role, index) in group.roles" :key="index">
                    <div
                      class="inline-flex w-72 cursor-pointer flex-row items-center gap-4 rounded border p-5 hover:border-primary"
                    >
                      <div class="inline-flex flex-col gap-y-3">
                        <span class="font-medium text-gray-900">
                          {{
                            role.metadata.annotations?.[
                              rbacAnnotations.DISPLAY_NAME
                            ]
                          }}
                        </span>
                        <span
                          v-if="
                            role.metadata.annotations?.[
                              rbacAnnotations.DEPENDENCIES
                            ]
                          "
                          class="text-xs text-gray-400"
                        >
                          {{
                            $t("core.role.common.text.dependent_on", {
                              roles: JSON.parse(
                                role.metadata.annotations?.[
                                  rbacAnnotations.DEPENDENCIES
                                ]
                              ).join(", "),
                            })
                          }}
                        </span>
                      </div>
                    </div>
                  </li>
                </ul>
              </dd>
            </div>
          </dl>
          <span v-else>
            {{ $t("core.common.text.none") }}
          </span>
        </VDescriptionItem>
        <VDescriptionItem
          :label="$t('core.plugin.detail.fields.creation_time')"
          :content="formatDatetime(plugin?.metadata.creationTimestamp)"
        />
        <VDescriptionItem
          :label="$t('core.plugin.detail.fields.last_starttime')"
          :content="formatDatetime(plugin?.status?.lastStartTime)"
        />
        <VDescriptionItem
          :label="$t('core.plugin.detail.fields.load_location')"
          :content="plugin?.status?.loadLocation"
        ></VDescriptionItem>
      </VDescription>
    </div>
  </div>
</template>
