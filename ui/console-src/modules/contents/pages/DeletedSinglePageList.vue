<script lang="ts" setup>
import PostContributorList from "@/components/user/PostContributorList.vue";
import { formatDatetime, relativeTimeTo } from "@/utils/date";
import { usePermission } from "@/utils/permission";
import { generateThumbnailUrl } from "@/utils/thumbnail";
import type { ListedSinglePage, SinglePage } from "@halo-dev/api-client";
import { consoleApiClient, coreApiClient } from "@halo-dev/api-client";
import {
  Dialog,
  IconAddCircle,
  IconDeleteBin,
  IconRefreshLine,
  Toast,
  VButton,
  VCard,
  VDropdownItem,
  VEmpty,
  VEntity,
  VEntityContainer,
  VEntityField,
  VLoading,
  VPageHeader,
  VPagination,
  VSpace,
  VStatusDot,
} from "@halo-dev/components";
import { useQuery } from "@tanstack/vue-query";
import { ref, watch } from "vue";
import { useI18n } from "vue-i18n";

const { currentUserHasPermission } = usePermission();
const { t } = useI18n();

const selectedPageNames = ref<string[]>([]);
const checkedAll = ref(false);
const keyword = ref("");

const page = ref(1);
const size = ref(20);
const total = ref(0);

const {
  data: singlePages,
  isLoading,
  isFetching,
  refetch,
} = useQuery<ListedSinglePage[]>({
  queryKey: ["deleted-singlePages", page, size, keyword],
  queryFn: async () => {
    const { data } = await consoleApiClient.content.singlePage.listSinglePages({
      labelSelector: [`content.halo.run/deleted=true`],
      page: page.value,
      size: size.value,
      keyword: keyword.value,
    });

    total.value = data.total;

    return data.items;
  },
  refetchInterval(data) {
    const deletedSinglePages = data?.filter(
      (singlePage) =>
        !!singlePage.page.metadata.deletionTimestamp ||
        !singlePage.page.spec.deleted
    );
    return deletedSinglePages?.length ? 1000 : false;
  },
});

const checkSelection = (singlePage: SinglePage) => {
  return selectedPageNames.value.includes(singlePage.metadata.name);
};

const handleCheckAllChange = (e: Event) => {
  const { checked } = e.target as HTMLInputElement;

  if (checked) {
    selectedPageNames.value =
      singlePages.value?.map((singlePage) => {
        return singlePage.page.metadata.name;
      }) || [];
  } else {
    selectedPageNames.value = [];
  }
};

const handleDeletePermanently = async (singlePage: SinglePage) => {
  Dialog.warning({
    title: t("core.deleted_page.operations.delete.title"),
    description: t("core.deleted_page.operations.delete.description"),
    confirmType: "danger",
    confirmText: t("core.common.buttons.confirm"),
    cancelText: t("core.common.buttons.cancel"),
    onConfirm: async () => {
      await coreApiClient.content.singlePage.deleteSinglePage({
        name: singlePage.metadata.name,
      });
      await refetch();

      Toast.success(t("core.common.toast.delete_success"));
    },
  });
};

const handleDeletePermanentlyInBatch = async () => {
  Dialog.warning({
    title: t("core.deleted_page.operations.delete_in_batch.title"),
    description: t("core.deleted_page.operations.delete_in_batch.description"),
    confirmType: "danger",
    confirmText: t("core.common.buttons.confirm"),
    cancelText: t("core.common.buttons.cancel"),
    onConfirm: async () => {
      await Promise.all(
        selectedPageNames.value.map((name) => {
          return coreApiClient.content.singlePage.deleteSinglePage({
            name,
          });
        })
      );
      await refetch();
      selectedPageNames.value = [];

      Toast.success(t("core.common.toast.delete_success"));
    },
  });
};

const handleRecovery = async (singlePage: SinglePage) => {
  Dialog.warning({
    title: t("core.deleted_page.operations.recovery.title"),
    description: t("core.deleted_page.operations.recovery.description"),
    confirmText: t("core.common.buttons.confirm"),
    cancelText: t("core.common.buttons.cancel"),
    onConfirm: async () => {
      await coreApiClient.content.singlePage.patchSinglePage({
        name: singlePage.metadata.name,
        jsonPatchInner: [
          {
            op: "add",
            path: "/spec/deleted",
            value: false,
          },
        ],
      });

      await refetch();

      Toast.success(t("core.common.toast.recovery_success"));
    },
  });
};

const handleRecoveryInBatch = async () => {
  Dialog.warning({
    title: t("core.deleted_page.operations.recovery_in_batch.title"),
    description: t(
      "core.deleted_page.operations.recovery_in_batch.description"
    ),
    confirmText: t("core.common.buttons.confirm"),
    cancelText: t("core.common.buttons.cancel"),
    onConfirm: async () => {
      await Promise.all(
        selectedPageNames.value.map((name) => {
          const singlePage = singlePages.value?.find(
            (item) => item.page.metadata.name === name
          )?.page;

          if (!singlePage) {
            return Promise.resolve();
          }

          return coreApiClient.content.singlePage.patchSinglePage({
            name: singlePage.metadata.name,
            jsonPatchInner: [
              {
                op: "add",
                path: "/spec/deleted",
                value: false,
              },
            ],
          });
        })
      );
      await refetch();
      selectedPageNames.value = [];

      Toast.success(t("core.common.toast.recovery_success"));
    },
  });
};

watch(selectedPageNames, (newValue) => {
  checkedAll.value = newValue.length === singlePages.value?.length;
});

watch(
  () => keyword.value,
  () => {
    page.value = 1;
  }
);
</script>

<template>
  <VPageHeader :title="$t('core.deleted_page.title')">
    <template #icon>
      <IconDeleteBin class="text-green-600" />
    </template>
    <template #actions>
      <VButton :route="{ name: 'SinglePages' }" size="sm">
        {{ $t("core.common.buttons.back") }}
      </VButton>
      <VButton
        v-permission="['system:singlepages:manage']"
        :route="{ name: 'SinglePageEditor' }"
        type="secondary"
      >
        <template #icon>
          <IconAddCircle />
        </template>
        {{ $t("core.common.buttons.new") }}
      </VButton>
    </template>
  </VPageHeader>
  <div class="m-0 md:m-4">
    <VCard :body-class="['!p-0']">
      <template #header>
        <div class="block w-full bg-gray-50 px-4 py-3">
          <div
            class="relative flex flex-col flex-wrap items-start gap-4 sm:flex-row sm:items-center"
          >
            <div
              v-permission="['system:singlepages:manage']"
              class="hidden items-center sm:flex"
            >
              <input
                v-model="checkedAll"
                type="checkbox"
                @change="handleCheckAllChange"
              />
            </div>
            <div class="flex w-full flex-1 items-center sm:w-auto">
              <SearchInput v-if="!selectedPageNames.length" v-model="keyword" />
              <VSpace v-else>
                <VButton type="danger" @click="handleDeletePermanentlyInBatch">
                  {{ $t("core.common.buttons.delete_permanently") }}
                </VButton>
                <VButton type="default" @click="handleRecoveryInBatch">
                  {{ $t("core.common.buttons.recovery") }}
                </VButton>
              </VSpace>
            </div>
            <VSpace spacing="lg" class="flex-wrap">
              <div class="flex flex-row gap-2">
                <div
                  class="group cursor-pointer rounded p-1 hover:bg-gray-200"
                  @click="refetch()"
                >
                  <IconRefreshLine
                    :class="{ 'animate-spin text-gray-900': isFetching }"
                    class="h-4 w-4 text-gray-600 group-hover:text-gray-900"
                  />
                </div>
              </div>
            </VSpace>
          </div>
        </div>
      </template>
      <VLoading v-if="isLoading" />
      <Transition v-else-if="!singlePages?.length" appear name="fade">
        <VEmpty
          :message="$t('core.deleted_page.empty.message')"
          :title="$t('core.deleted_page.empty.title')"
        >
          <template #actions>
            <VSpace>
              <VButton @click="refetch">
                {{ $t("core.common.buttons.refresh") }}
              </VButton>
              <VButton
                v-permission="['system:singlepages:view']"
                :route="{ name: 'SinglePages' }"
                type="secondary"
              >
                {{ $t("core.common.buttons.back") }}
              </VButton>
            </VSpace>
          </template>
        </VEmpty>
      </Transition>
      <Transition v-else appear name="fade">
        <VEntityContainer>
          <VEntity
            v-for="singlePage in singlePages"
            :key="singlePage.page.metadata.name"
            :is-selected="checkSelection(singlePage.page)"
          >
            <template
              v-if="currentUserHasPermission(['system:singlepages:manage'])"
              #checkbox
            >
              <input
                v-model="selectedPageNames"
                :value="singlePage.page.metadata.name"
                type="checkbox"
              />
            </template>
            <template #start>
              <VEntityField v-if="singlePage.page.spec.cover">
                <template #description>
                  <div
                    class="aspect-h-2 aspect-w-3 w-20 overflow-hidden rounded-md"
                  >
                    <img
                      class="h-full w-full object-cover"
                      :src="
                        generateThumbnailUrl(singlePage.page.spec.cover, 's')
                      "
                    />
                  </div>
                </template>
              </VEntityField>
              <VEntityField
                :title="singlePage.page.spec.title"
                max-width="30rem"
              >
                <template #description>
                  <VSpace>
                    <span class="text-xs text-gray-500">
                      {{
                        $t("core.page.list.fields.visits", {
                          visits: singlePage.stats.visit || 0,
                        })
                      }}
                    </span>
                    <span class="text-xs text-gray-500">
                      {{
                        $t("core.page.list.fields.comments", {
                          comments: singlePage.stats.totalComment || 0,
                        })
                      }}
                    </span>
                  </VSpace>
                </template>
              </VEntityField>
            </template>
            <template #end>
              <VEntityField>
                <template #description>
                  <PostContributorList
                    :owner="singlePage.owner"
                    :contributors="singlePage.contributors"
                  />
                </template>
              </VEntityField>
              <VEntityField v-if="!singlePage?.page?.spec.deleted">
                <template #description>
                  <VStatusDot
                    v-tooltip="$t('core.common.tooltips.recovering')"
                    state="success"
                    animate
                  />
                </template>
              </VEntityField>
              <VEntityField v-if="singlePage?.page?.metadata.deletionTimestamp">
                <template #description>
                  <VStatusDot
                    v-tooltip="$t('core.common.status.deleting')"
                    state="warning"
                    animate
                  />
                </template>
              </VEntityField>
              <VEntityField
                v-if="singlePage.page.spec.publishTime"
                v-tooltip="formatDatetime(singlePage.page.spec.publishTime)"
                :description="relativeTimeTo(singlePage.page.spec.publishTime)"
              >
              </VEntityField>
            </template>
            <template
              v-if="currentUserHasPermission(['system:singlepages:manage'])"
              #dropdownItems
            >
              <VDropdownItem
                type="danger"
                @click="handleDeletePermanently(singlePage.page)"
              >
                {{ $t("core.common.buttons.delete_permanently") }}
              </VDropdownItem>
              <VDropdownItem @click="handleRecovery(singlePage.page)">
                {{ $t("core.common.buttons.recovery") }}
              </VDropdownItem>
            </template>
          </VEntity>
        </VEntityContainer>
      </Transition>

      <template #footer>
        <VPagination
          v-model:page="page"
          v-model:size="size"
          :page-label="$t('core.components.pagination.page_label')"
          :size-label="$t('core.components.pagination.size_label')"
          :total-label="
            $t('core.components.pagination.total_label', { total: total })
          "
          :total="total"
          :size-options="[20, 30, 50, 100]"
        />
      </template>
    </VCard>
  </div>
</template>
