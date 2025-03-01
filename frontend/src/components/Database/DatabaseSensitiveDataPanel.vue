<template>
  <div class="w-full space-y-4">
    <FeatureAttentionForInstanceLicense
      v-if="hasSensitiveDataFeature && isMissingLicenseForInstance"
      feature="bb.feature.sensitive-data"
    />
    <div class="textinfolabel">
      {{ $t("settings.sensitive-data.description") }}
      <a
        href="https://www.bytebase.com/docs/security/mask-data?source=console"
        class="normal-link inline-flex flex-row items-center"
        target="_blank"
      >
        {{ $t("common.learn-more") }}
        <heroicons-outline:external-link class="w-4 h-4" />
      </a>
    </div>
    <div
      class="flex flex-col space-x-2 lg:flex-row gap-y-4 justify-between items-end lg:items-center"
    >
      <SearchBox v-model:value="state.searchText" style="max-width: 100%" />
      <div class="flex items-center space-x-2">
        <MaskingLevelDropdown
          v-model:level="state.selectedMaskLevel"
          style="width: 12rem"
          :clearable="true"
          :level-list="[
            MaskingLevel.FULL,
            MaskingLevel.PARTIAL,
            MaskingLevel.NONE,
          ]"
        />
        <NButton
          type="primary"
          :disabled="
            state.pendingGrantAccessColumn.length === 0 || !hasPermission
          "
          @click="onGrantAccessButtonClick"
        >
          <template #icon>
            <ShieldCheckIcon v-if="hasSensitiveDataFeature" class="w-4" />
            <FeatureBadge
              v-else
              feature="bb.feature.sensitive-data"
              custom-class="text-white"
            />
          </template>
          {{ $t("settings.sensitive-data.grant-access") }}
        </NButton>
      </div>
    </div>

    <SensitiveColumnTable
      v-if="hasSensitiveDataFeature"
      :row-clickable="true"
      :row-selectable="true"
      :show-operation="hasPermission && hasSensitiveDataFeature"
      :column-list="filteredColumnList"
      :checked-column-index-list="checkedColumnIndexList"
      @click="onRowClick"
      @checked:update="updateCheckedColumnList($event)"
    />

    <NoDataPlaceholder v-else />
  </div>

  <FeatureModal
    feature="bb.feature.sensitive-data"
    :open="state.showFeatureModal"
    :instance="database.instanceResource"
    @cancel="state.showFeatureModal = false"
  />

  <GrantAccessDrawer
    v-if="
      state.showGrantAccessDrawer && state.pendingGrantAccessColumn.length > 0
    "
    :column-list="
      state.pendingGrantAccessColumn.map((maskData) => ({
        database,
        maskData,
      }))
    "
    :project-name="database.project"
    @dismiss="
      () => {
        state.showGrantAccessDrawer = false;
        state.pendingGrantAccessColumn = [];
      }
    "
  />

  <SensitiveColumnDrawer
    v-if="
      filteredColumnList.length > 0 &&
      state.showSensitiveColumnDrawer &&
      state.pendingGrantAccessColumn.length === 1
    "
    :database="database"
    :mask="
      state.pendingGrantAccessColumn.length === 1
        ? state.pendingGrantAccessColumn[0]
        : filteredColumnList[0]
    "
    @dismiss="
      () => {
        state.showSensitiveColumnDrawer = false;
        state.pendingGrantAccessColumn = [];
      }
    "
  />
</template>

<script lang="tsx" setup>
import { ShieldCheckIcon } from "lucide-vue-next";
import { NButton } from "naive-ui";
import { computed, reactive, watch } from "vue";
import { useI18n } from "vue-i18n";
import { useRouter } from "vue-router";
import { updateColumnConfig } from "@/components/ColumnDataTable/utils";
import {
  FeatureModal,
  FeatureBadge,
  FeatureAttentionForInstanceLicense,
} from "@/components/FeatureGuard";
import GrantAccessDrawer from "@/components/SensitiveData/GrantAccessDrawer.vue";
import SensitiveColumnDrawer from "@/components/SensitiveData/SensitiveColumnDrawer.vue";
import MaskingLevelDropdown from "@/components/SensitiveData/components/MaskingLevelDropdown.vue";
import SensitiveColumnTable from "@/components/SensitiveData/components/SensitiveColumnTable.vue";
import type { MaskData } from "@/components/SensitiveData/types";
import { isCurrentColumnException } from "@/components/SensitiveData/utils";
import { SearchBox } from "@/components/v2";
import {
  featureToRef,
  pushNotification,
  usePolicyV1Store,
  useSubscriptionV1Store,
  useDatabaseCatalog,
} from "@/store";
import { type ComposedDatabase } from "@/types";
import { MaskingLevel } from "@/types/proto/v1/common";
import { PolicyType } from "@/types/proto/v1/org_policy_service";
import { autoDatabaseRoute, hasProjectPermissionV2 } from "@/utils";
import NoDataPlaceholder from "../misc/NoDataPlaceholder.vue";

const props = defineProps<{
  database: ComposedDatabase;
}>();

interface LocalState {
  searchText: string;
  showFeatureModal: boolean;
  isLoading: boolean;
  sensitiveColumnList: MaskData[];
  pendingGrantAccessColumn: MaskData[];
  showGrantAccessDrawer: boolean;
  showSensitiveColumnDrawer: boolean;
  selectedMaskLevel?: MaskingLevel;
}

const state = reactive<LocalState>({
  searchText: "",
  showFeatureModal: false,
  isLoading: false,
  sensitiveColumnList: [],
  pendingGrantAccessColumn: [],
  showGrantAccessDrawer: false,
  showSensitiveColumnDrawer: false,
});

const hasPermission = computed(() => {
  // TODO(ed): the permission and subscription check for db config update
  return hasProjectPermissionV2(
    props.database.projectEntity,
    "bb.databases.update"
  );
});

const { t } = useI18n();
const router = useRouter();
const policyStore = usePolicyV1Store();
const subscriptionStore = useSubscriptionV1Store();

const hasSensitiveDataFeature = featureToRef("bb.feature.sensitive-data");

const isMissingLicenseForInstance = computed(() =>
  subscriptionStore.instanceMissingLicense(
    "bb.feature.sensitive-data",
    props.database.instanceResource
  )
);

const databaseCatalog = useDatabaseCatalog(props.database.name, false);

const updateList = async () => {
  state.isLoading = true;
  const sensitiveColumnList: MaskData[] = [];

  for (const schema of databaseCatalog.value.schemas) {
    for (const table of schema.tables) {
      for (const column of table.columns?.columns ?? []) {
        if (
          column.maskingLevel === MaskingLevel.MASKING_LEVEL_UNSPECIFIED
        ) {
          continue;
        }
        sensitiveColumnList.push({
          schema: schema.name,
          table: table.name,
          column: column.name,
          maskingLevel: column.maskingLevel,
          fullMaskingAlgorithmId: column.fullMaskingAlgorithmId,
          partialMaskingAlgorithmId: column.partialMaskingAlgorithmId,
        });
      }
    }
  }

  state.sensitiveColumnList = sensitiveColumnList;
  state.isLoading = false;
};

watch(databaseCatalog, updateList, { immediate: true, deep: true });

const filteredColumnList = computed(() => {
  let list = state.sensitiveColumnList;
  if (state.selectedMaskLevel) {
    list = list.filter((item) => item.maskingLevel === state.selectedMaskLevel);
  }
  const searchText = state.searchText.trim().toLowerCase();
  if (searchText) {
    list = list.filter(
      (item) =>
        item.column.includes(searchText) ||
        item.table.includes(searchText) ||
        item.schema.includes(searchText)
    );
  }
  return list;
});

const removeSensitiveColumn = async (sensitiveColumn: MaskData) => {
  await updateColumnConfig({
    database: props.database.name,
    schema: sensitiveColumn.schema,
    table: sensitiveColumn.table,
    column: sensitiveColumn.column,
    columnCatalog: {
      maskingLevel: MaskingLevel.MASKING_LEVEL_UNSPECIFIED,
    },
  });
  await removeMaskingExceptions(sensitiveColumn);
};

const removeMaskingExceptions = async (sensitiveColumn: MaskData) => {
  const policy = await policyStore.getOrFetchPolicyByParentAndType({
    parentPath: props.database.project,
    policyType: PolicyType.MASKING_EXCEPTION,
  });
  if (!policy) {
    return;
  }

  const exceptions = (
    policy.maskingExceptionPolicy?.maskingExceptions ?? []
  ).filter(
    (exception) =>
      !isCurrentColumnException(exception, {
        database: props.database,
        maskData: sensitiveColumn,
      })
  );

  policy.maskingExceptionPolicy = {
    ...(policy.maskingExceptionPolicy ?? {}),
    maskingExceptions: exceptions,
  };
  await policyStore.upsertPolicy({
    parentPath: props.database.project,
    policy,
  });
};

const onColumnRemove = async (column: MaskData) => {
  await removeSensitiveColumn(column);
  pushNotification({
    module: "bytebase",
    style: "SUCCESS",
    title: t("common.updated"),
  });
};

const onRowClick = async (
  item: MaskData,
  row: number,
  action: "VIEW" | "DELETE" | "EDIT"
) => {
  switch (action) {
    case "VIEW": {
      const query: Record<string, string> = {
        table: item.table,
      };
      if (item.schema != "") {
        query.schema = item.schema;
      }
      router.push({
        ...autoDatabaseRoute(router, props.database),
        query,
      });
      break;
    }
    case "DELETE":
      await onColumnRemove(item);
      break;
    case "EDIT":
      state.pendingGrantAccessColumn = [item];
      if (isMissingLicenseForInstance.value) {
        state.showFeatureModal = true;
        return;
      }
      state.showSensitiveColumnDrawer = true;
      break;
  }
};

const onGrantAccessButtonClick = () => {
  if (!hasSensitiveDataFeature.value) {
    state.showFeatureModal = true;
    return;
  }
  state.showGrantAccessDrawer = true;
};

const checkedColumnIndexList = computed(() => {
  const resp = [];
  for (const column of state.pendingGrantAccessColumn) {
    const index = filteredColumnList.value.findIndex((col) => {
      return (
        col.table === column.table &&
        col.schema === column.schema &&
        col.column === column.column
      );
    });
    if (index >= 0) {
      resp.push(index);
    }
  }
  return resp;
});

const updateCheckedColumnList = (indexes: number[]) => {
  state.pendingGrantAccessColumn = [];
  for (const index of indexes) {
    const col = filteredColumnList.value[index];
    if (col) {
      state.pendingGrantAccessColumn.push(col);
    }
  }
};
</script>
