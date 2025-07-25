<script lang="ts" setup>
import { useRoleForm, useRoleTemplateSelection } from "@/composables/use-role";
import { rbacAnnotations } from "@/constants/annotations";
import { SUPER_ROLE_NAME } from "@/constants/constants";
import { pluginLabels, roleLabels } from "@/constants/labels";
import { formatDatetime } from "@/utils/date";
import { resolveDeepDependencies } from "@/utils/role";
import type { Role } from "@halo-dev/api-client";
import { coreApiClient } from "@halo-dev/api-client";
import {
  IconShieldUser,
  VAlert,
  VButton,
  VCard,
  VDescription,
  VDescriptionItem,
  VPageHeader,
  VTabbar,
  VTag,
} from "@halo-dev/components";
import { useQuery } from "@tanstack/vue-query";
import { computed, ref, watch } from "vue";
import { useI18n } from "vue-i18n";
import { useRoute } from "vue-router";

const route = useRoute();
const { t } = useI18n();

const tabActiveId = ref("detail");

const { data: roleTemplates } = useQuery({
  queryKey: ["role-templates"],
  queryFn: async () => {
    const { data } = await coreApiClient.role.listRole({
      page: 0,
      size: 0,
      labelSelector: [`${roleLabels.TEMPLATE}=true`, "!halo.run/hidden"],
    });
    return data.items;
  },
});

const { roleTemplateGroups, handleRoleTemplateSelect, selectedRoleTemplates } =
  useRoleTemplateSelection(roleTemplates);

const { formState, isSubmitting, handleCreateOrUpdate } = useRoleForm();

const isSystemReserved = computed(() => {
  return (
    formState.value.metadata.labels?.[roleLabels.SYSTEM_RESERVED] === "true"
  );
});

const isSuperRole = computed(() => {
  return formState.value.metadata.name === SUPER_ROLE_NAME;
});

const getRoleCountText = computed(() => {
  if (formState.value.metadata.name === SUPER_ROLE_NAME) {
    return t("core.role.common.text.contains_all_permissions");
  }

  const dependencies = new Set<string>(
    resolveDeepDependencies(formState.value, roleTemplates.value || [])
  );

  return t("core.role.common.text.contains_n_permissions", {
    count: dependencies.size || 0,
  });
});

watch(
  () => selectedRoleTemplates.value,
  (newValue) => {
    if (formState.value.metadata.annotations) {
      formState.value.metadata.annotations[rbacAnnotations.DEPENDENCIES] =
        JSON.stringify(Array.from(newValue));
    }
  }
);

const { refetch } = useQuery<Role>({
  queryKey: ["role", route.params.name],
  queryFn: async () => {
    const { data } = await coreApiClient.role.getRole({
      name: route.params.name as string,
    });
    return data;
  },
  onSuccess(data) {
    formState.value = data;
    selectedRoleTemplates.value = new Set<string>(
      resolveDeepDependencies(data, roleTemplates.value || [])
    );
  },
  enabled: computed(() => !!roleTemplates.value),
});

const handleUpdateRole = async () => {
  await handleCreateOrUpdate();
  await refetch();
};
</script>
<template>
  <VPageHeader :title="$t('core.role.detail.title')">
    <template #icon>
      <IconShieldUser />
    </template>
  </VPageHeader>
  <div class="m-0 md:m-4">
    <VCard :body-class="['!p-0']">
      <template #header>
        <VTabbar
          v-model:active-id="tabActiveId"
          :items="[
            { id: 'detail', label: $t('core.role.detail.tabs.detail') },
            {
              id: 'permissions',
              label: $t('core.role.detail.tabs.permissions'),
            },
          ]"
          class="w-full !rounded-none"
          type="outline"
        ></VTabbar>
      </template>
      <div v-if="tabActiveId === 'detail'">
        <div class="px-4 py-4 sm:px-6">
          <h3 class="text-lg font-medium leading-6 text-gray-900">
            {{ $t("core.role.detail.header.title") }}
          </h3>
          <p
            class="mt-1 flex max-w-2xl items-center gap-2 text-sm text-gray-500"
          >
            <span>{{ getRoleCountText }}</span>
          </p>
        </div>
        <div class="border-t border-gray-200">
          <VDescription>
            <VDescriptionItem
              :label="$t('core.role.detail.fields.display_name')"
            >
              {{
                formState.metadata?.annotations?.[
                  rbacAnnotations.DISPLAY_NAME
                ] || formState.metadata?.name
              }}
            </VDescriptionItem>
            <VDescriptionItem
              :label="$t('core.role.detail.fields.name')"
              :content="formState.metadata?.name"
            />
            <VDescriptionItem :label="$t('core.role.detail.fields.type')">
              <VTag>
                {{
                  isSystemReserved
                    ? t("core.role.common.text.system_reserved")
                    : t("core.role.common.text.custom")
                }}
              </VTag>
            </VDescriptionItem>
            <VDescriptionItem
              :label="$t('core.role.detail.fields.creation_time')"
              :content="formatDatetime(formState.metadata.creationTimestamp)"
            />
          </VDescription>
        </div>
      </div>

      <div v-if="tabActiveId === 'permissions'">
        <div v-if="isSystemReserved" class="px-4 py-5">
          <VAlert
            :title="$t('core.common.text.tip')"
            :description="
              $t(
                'core.role.permissions_detail.system_reserved_alert.description'
              )
            "
            class="w-full sm:w-1/4"
            :closable="false"
          />
        </div>

        <div>
          <dl class="divide-y divide-gray-100">
            <div
              v-for="(group, index) in roleTemplateGroups"
              :key="index"
              class="bg-white px-4 py-5 hover:bg-gray-50 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6"
            >
              <dt class="text-sm font-medium text-gray-900">
                <div>
                  {{ $t(`core.rbac.${group.module}`, group.module as string) }}
                </div>
                <div
                  v-if="
                    group.roles.length &&
                    group.roles[0].metadata.labels?.[pluginLabels.NAME]
                  "
                  class="mt-3 text-xs text-gray-500"
                >
                  <i18n-t
                    keypath="core.role.common.text.provided_by_plugin"
                    tag="div"
                  >
                    <template #plugin>
                      <RouterLink
                        :to="{
                          name: 'PluginDetail',
                          params: {
                            name: group.roles[0].metadata.labels?.[
                              pluginLabels.NAME
                            ],
                          },
                        }"
                        class="hover:text-blue-600"
                      >
                        {{
                          group.roles[0].metadata.labels?.[pluginLabels.NAME]
                        }}
                      </RouterLink>
                    </template>
                  </i18n-t>
                </div>
              </dt>
              <dd class="mt-1 text-sm text-gray-900 sm:col-span-2 sm:mt-0">
                <ul class="space-y-2">
                  <li v-for="role in group.roles" :key="role.metadata.name">
                    <label
                      class="inline-flex w-72 cursor-pointer flex-row items-center gap-4 rounded-base border p-5 hover:border-primary"
                    >
                      <input
                        v-if="!isSuperRole"
                        v-model="selectedRoleTemplates"
                        :value="role.metadata.name"
                        type="checkbox"
                        :disabled="isSystemReserved"
                        @change="handleRoleTemplateSelect"
                      />
                      <input v-else type="checkbox" checked disabled />
                      <div class="flex flex-1 flex-col gap-y-3">
                        <span class="font-medium text-gray-900">
                          {{
                            $t(
                              `core.rbac.${
                                role.metadata.annotations?.[
                                  rbacAnnotations.DISPLAY_NAME
                                ]
                              }`,
                              role.metadata.annotations?.[
                                rbacAnnotations.DISPLAY_NAME
                              ] as string
                            )
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
                              )
                                .map((item: string) =>
                                  $t(`core.rbac.${item}`, item as string)
                                )
                                .join("，"),
                            })
                          }}
                        </span>
                      </div>
                    </label>
                  </li>
                </ul>
              </dd>
            </div>
          </dl>
          <div v-permission="['system:roles:manage']" class="p-4">
            <VButton
              :loading="isSubmitting"
              type="secondary"
              :disabled="isSystemReserved"
              @click="handleUpdateRole"
            >
              {{ $t("core.common.buttons.save") }}
            </VButton>
          </div>
        </div>
      </div>
    </VCard>
  </div>
</template>
