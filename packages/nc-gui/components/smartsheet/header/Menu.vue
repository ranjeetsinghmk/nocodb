<script lang="ts" setup>
import type { ColumnReqType } from 'nocodb-sdk'
import { PlanLimitTypes, RelationTypes, UITypes, isLinksOrLTAR } from 'nocodb-sdk'
import { SmartsheetStoreEvents } from '#imports'

const props = defineProps<{ virtual?: boolean; isOpen: boolean; isHiddenCol?: boolean }>()

const emit = defineEmits(['edit', 'addColumn', 'update:isOpen'])

const virtual = toRef(props, 'virtual')

const isOpen = useVModel(props, 'isOpen', emit)

const { eventBus, allFilters } = useSmartsheetStoreOrThrow()

const column = inject(ColumnInj)

const reloadDataHook = inject(ReloadViewDataHookInj)

const meta = inject(MetaInj, ref())

const view = inject(ActiveViewInj, ref())

const isLocked = inject(IsLockedInj)

const isPublic = inject(IsPublicInj, ref(false))

const isExpandedForm = inject(IsExpandedFormOpenInj, ref(false))

const { insertSort } = useViewSorts(view, () => reloadDataHook?.trigger())

const { $api, $e } = useNuxtApp()

const { t } = useI18n()

const { getMeta } = useMetas()

const { addUndo, defineModelScope, defineViewScope } = useUndoRedo()

const showDeleteColumnModal = ref(false)

const { gridViewCols } = useViewColumnsOrThrow()

const { fieldsToGroupBy, groupByLimit } = useViewGroupByOrThrow(view)

const setAsDisplayValue = async () => {
  try {
    const currentDisplayValue = meta?.value?.columns?.find((f) => f.pv)

    isOpen.value = false

    await $api.dbTableColumn.primaryColumnSet(column?.value?.id as string)

    await getMeta(meta?.value?.id as string, true)

    eventBus.emit(SmartsheetStoreEvents.FIELD_RELOAD)

    // Successfully updated as primary column
    // message.success(t('msg.success.primaryColumnUpdated'))

    $e('a:column:set-primary')

    addUndo({
      redo: {
        fn: async (id: string) => {
          await $api.dbTableColumn.primaryColumnSet(id)

          await getMeta(meta?.value?.id as string, true)

          eventBus.emit(SmartsheetStoreEvents.FIELD_RELOAD)

          // Successfully updated as primary column
          // message.success(t('msg.success.primaryColumnUpdated'))
        },
        args: [column?.value?.id as string],
      },
      undo: {
        fn: async (id: string) => {
          await $api.dbTableColumn.primaryColumnSet(id)

          await getMeta(meta?.value?.id as string, true)

          eventBus.emit(SmartsheetStoreEvents.FIELD_RELOAD)

          // Successfully updated as primary column
          // message.success(t('msg.success.primaryColumnUpdated'))
        },
        args: [currentDisplayValue?.id],
      },
      scope: defineModelScope({ model: meta.value }),
    })
  } catch (e) {
    message.error(t('msg.error.primaryColumnUpdateFailed'))
  }
}

const sortByColumn = async (direction: 'asc' | 'desc') => {
  await insertSort({
    column: column!.value,
    direction,
    reloadDataHook,
  })
}

const isDuplicateDlgOpen = ref(false)
const selectedColumnExtra = ref<any>()
const duplicateDialogRef = ref<any>()

const duplicateVirtualColumn = async () => {
  let columnCreatePayload = {}

  // generate duplicate column title
  const duplicateColumnTitle = getUniqueColumnName(`${column!.value.title} copy`, meta!.value!.columns!)

  columnCreatePayload = {
    ...column!.value!,
    ...(column!.value.colOptions ?? {}),
    title: duplicateColumnTitle,
    column_name: duplicateColumnTitle.replace(/\s/g, '_'),
    id: undefined,
    colOptions: undefined,
    order: undefined,
    system: false,
  }

  try {
    const gridViewColumnList = (await $api.dbViewColumn.list(view.value?.id as string)).list

    const currentColumnIndex = gridViewColumnList.findIndex((f) => f.fk_column_id === column!.value.id)
    let newColumnOrder
    if (currentColumnIndex === gridViewColumnList.length - 1) {
      newColumnOrder = gridViewColumnList[currentColumnIndex].order! + 1
    } else {
      newColumnOrder = (gridViewColumnList[currentColumnIndex].order! + gridViewColumnList[currentColumnIndex + 1].order!) / 2
    }

    await $api.dbTableColumn.create(meta!.value!.id!, {
      ...columnCreatePayload,
      pv: false,
      view_id: view.value!.id as string,
      column_order: {
        order: newColumnOrder,
        view_id: view.value!.id as string,
      },
    } as ColumnReqType)
    await getMeta(meta!.value!.id!, true)

    eventBus.emit(SmartsheetStoreEvents.FIELD_RELOAD)
    reloadDataHook?.trigger()

    // message.success(t('msg.success.columnDuplicated'))
  } catch (e) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
  // closing dropdown
  isOpen.value = false
}

const openDuplicateDlg = async () => {
  if (!column?.value) return
  if (
    column.value.uidt &&
    [
      UITypes.Lookup,
      UITypes.Rollup,
      UITypes.CreatedTime,
      UITypes.LastModifiedTime,
      UITypes.CreatedBy,
      UITypes.LastModifiedBy,
    ].includes(column.value.uidt as UITypes)
  ) {
    duplicateVirtualColumn()
  } else {
    const gridViewColumnList = (await $api.dbViewColumn.list(view.value?.id as string)).list

    const currentColumnIndex = gridViewColumnList.findIndex((f) => f.fk_column_id === column!.value.id)
    let newColumnOrder
    if (currentColumnIndex === gridViewColumnList.length - 1) {
      newColumnOrder = gridViewColumnList[currentColumnIndex].order! + 1
    } else {
      newColumnOrder = (gridViewColumnList[currentColumnIndex].order! + gridViewColumnList[currentColumnIndex + 1].order!) / 2
    }

    selectedColumnExtra.value = {
      pv: false,
      view_id: view.value!.id as string,
      column_order: {
        order: newColumnOrder,
        view_id: view.value!.id as string,
      },
    }

    if (column.value.uidt === UITypes.Formula) {
      nextTick(() => {
        duplicateDialogRef?.value?.duplicate()
      })
    } else {
      isDuplicateDlgOpen.value = true
    }

    isOpen.value = false
  }
}

// add column before or after current column
const addColumn = async (before = false) => {
  const gridViewColumnList = (await $api.dbViewColumn.list(view.value?.id as string)).list

  const currentColumnIndex = gridViewColumnList.findIndex((f) => f.fk_column_id === column!.value.id)

  let newColumnOrder
  if (before) {
    if (currentColumnIndex === 0) {
      newColumnOrder = gridViewColumnList[currentColumnIndex].order / 2
    } else {
      newColumnOrder = (gridViewColumnList[currentColumnIndex].order! + gridViewColumnList[currentColumnIndex - 1]?.order) / 2
    }
  } else {
    if (currentColumnIndex === gridViewColumnList.length - 1) {
      newColumnOrder = gridViewColumnList[currentColumnIndex].order + 1
    } else {
      newColumnOrder = (gridViewColumnList[currentColumnIndex].order! + gridViewColumnList[currentColumnIndex + 1]?.order) / 2
    }
  }

  emit('addColumn', {
    column_order: {
      order: newColumnOrder,
      view_id: view.value?.id as string,
    },
  })
}

// hide the field in view
const hideOrShowField = async () => {
  const gridViewColumnList = (await $api.dbViewColumn.list(view.value?.id as string)).list

  const currentColumn = gridViewColumnList.find((f) => f.fk_column_id === column!.value.id)

  const promises = [$api.dbViewColumn.update(view.value!.id!, currentColumn!.id!, { show: !currentColumn.show })]

  if (isExpandedForm.value) {
    promises.push(getMeta(meta?.value?.id as string, true))
  }

  await Promise.all(promises)

  eventBus.emit(SmartsheetStoreEvents.FIELD_RELOAD)
  if (!currentColumn.show) {
    reloadDataHook?.trigger()
  }

  addUndo({
    redo: {
      fn: async function redo(id: string, show: boolean) {
        const promises = [$api.dbViewColumn.update(view.value!.id!, id, { show: !show })]

        if (isExpandedForm.value) {
          promises.push(getMeta(meta?.value?.id as string, true))
        }

        await Promise.all(promises)

        eventBus.emit(SmartsheetStoreEvents.FIELD_RELOAD)
        if (!show) {
          reloadDataHook?.trigger()
        }
      },
      args: [currentColumn!.id, currentColumn.show],
    },
    undo: {
      fn: async function undo(id: string, show: boolean) {
        const promises = [$api.dbViewColumn.update(view.value!.id!, id, { show })]

        if (isExpandedForm.value) {
          promises.push(getMeta(meta?.value?.id as string, true))
        }

        await Promise.all(promises)

        eventBus.emit(SmartsheetStoreEvents.FIELD_RELOAD)
        reloadDataHook?.trigger()
        if (show) {
          reloadDataHook?.trigger()
        }
      },
      args: [currentColumn!.id, currentColumn.show],
    },
    scope: defineViewScope({ view: view.value }),
  })
}

const handleDelete = () => {
  // closing the dropdown
  // when modal opens
  isOpen.value = false
  showDeleteColumnModal.value = true
}

const onEditPress = () => {
  isOpen.value = false
  emit('edit')
}

const onInsertBefore = () => {
  isOpen.value = false
  addColumn(true)
}
const onInsertAfter = () => {
  isOpen.value = false
  addColumn()
}

const isDeleteAllowed = computed(() => {
  return column?.value && !column.value.system
})
const isDuplicateAllowed = computed(() => {
  return column?.value && !column.value.system
})
const isFilterSupported = computed(
  () =>
    !!(meta.value?.columns || []).find((f) => f.id === column?.value?.id && ![UITypes.QrCode, UITypes.Barcode].includes(f.uidt)),
)

const { getPlanLimit } = useWorkspace()

const isFilterLimitExceeded = computed(
  () =>
    allFilters.value.filter((f) => !(f.is_group || f.status === 'delete')).length >= getPlanLimit(PlanLimitTypes.FILTER_LIMIT),
)

const isGroupedByThisField = computed(() => !!gridViewCols.value[column?.value?.id]?.group_by)

const isGroupBySupported = computed(() => !!(fieldsToGroupBy.value || []).find((f) => f.id === column?.value?.id))

const isGroupByLimitExceeded = computed(() => {
  const groupBy = Object.values(gridViewCols.value).filter((c) => c.group_by)
  return !(fieldsToGroupBy.value.length && fieldsToGroupBy.value.length > groupBy.length && groupBy.length < groupByLimit)
})

const filterOrGroupByThisField = (event: SmartsheetStoreEvents) => {
  if (column?.value) {
    eventBus.emit(event, column.value)
  }
  isOpen.value = false
}
</script>

<template>
  <a-dropdown
    v-if="!isLocked"
    v-model:visible="isOpen"
    :trigger="['click']"
    :placement="isExpandedForm ? 'bottomLeft' : 'bottomRight'"
    overlay-class-name="nc-dropdown-column-operations !border-1 rounded-lg !shadow-xl "
    @click.stop="isOpen = !isOpen"
  >
    <div @dblclick.stop>
      <div v-if="isExpandedForm" class="h-[1px]">&nbsp;</div>
      <GeneralIcon v-else icon="arrowDown" class="text-grey h-full text-grey nc-ui-dt-dropdown cursor-pointer outline-0 mr-2" />
    </div>
    <template #overlay>
      <NcMenu
        class="flex flex-col gap-1 border-gray-200 nc-column-options"
        :class="{
          'min-w-[256px]': isExpandedForm,
        }"
      >
        <NcMenuItem @click="onEditPress">
          <div class="nc-column-edit nc-header-menu-item">
            <component :is="iconMap.ncEdit" class="text-gray-700" />
            <!-- Edit -->
            {{ $t('general.edit') }}
          </div>
        </NcMenuItem>
        <NcMenuItem v-if="isExpandedForm && !column?.pk" :disabled="!isDuplicateAllowed" @click="openDuplicateDlg">
          <div v-e="['a:field:duplicate']" class="nc-column-duplicate nc-header-menu-item">
            <component :is="iconMap.duplicate" class="text-gray-700" />
            <!-- Duplicate -->
            {{ t('general.duplicate') }}
          </div>
        </NcMenuItem>
        <a-divider v-if="!column?.pv" class="!my-0" />
        <NcMenuItem v-if="!column?.pv" @click="hideOrShowField">
          <div v-e="['a:field:hide']" class="nc-column-insert-before nc-header-menu-item">
            <component :is="isHiddenCol ? iconMap.eye : iconMap.eyeSlash" class="text-gray-700 !w-3.75 !h-3.75" />
            <!-- Hide Field -->
            {{ isHiddenCol ? $t('general.showField') : $t('general.hideField') }}
          </div>
        </NcMenuItem>
        <NcMenuItem
          v-if="(!virtual || column?.uidt === UITypes.Formula) && !column?.pv && !isHiddenCol"
          @click="setAsDisplayValue"
        >
          <div class="nc-column-set-primary nc-header-menu-item item">
            <GeneralIcon icon="star" class="text-gray-700 !w-4.25 !h-4.25" />

            <!--       todo : tooltip -->
            <!-- Set as Display value -->
            {{ $t('activity.setDisplay') }}
          </div>
        </NcMenuItem>

        <template v-if="!isExpandedForm">
          <a-divider v-if="!isLinksOrLTAR(column) || column.colOptions.type !== RelationTypes.BELONGS_TO" class="!my-0" />

          <template v-if="!isLinksOrLTAR(column) || column.colOptions.type !== RelationTypes.BELONGS_TO">
            <NcMenuItem @click="sortByColumn('asc')">
              <div v-e="['a:field:sort', { dir: 'asc' }]" class="nc-column-insert-after nc-header-menu-item">
                <component
                  :is="iconMap.sortDesc"
                  class="text-gray-700 !rotate-180 !w-4.25 !h-4.25"
                  :style="{
                    transform: 'rotate(180deg)',
                  }"
                />

                <!-- Sort Ascending -->
                {{ $t('general.sortAsc') }}
              </div>
            </NcMenuItem>
            <NcMenuItem @click="sortByColumn('desc')">
              <div v-e="['a:field:sort', { dir: 'desc' }]" class="nc-column-insert-before nc-header-menu-item">
                <!-- Sort Descending -->
                <component :is="iconMap.sortDesc" class="text-gray-700 !w-4.25 !h-4.25" />
                {{ $t('general.sortDesc').trim() }}
              </div>
            </NcMenuItem>
          </template>

          <a-divider class="!my-0" />

          <NcTooltip :disabled="isFilterSupported && !isFilterLimitExceeded">
            <template #title>
              {{
                !isFilterSupported
                  ? "This field type doesn't support filtering"
                  : isFilterLimitExceeded
                  ? 'Filter by limit exceeded'
                  : ''
              }}
            </template>
            <NcMenuItem
              :disabled="!isFilterSupported || isFilterLimitExceeded"
              @click="filterOrGroupByThisField(SmartsheetStoreEvents.FILTER_ADD)"
            >
              <div v-e="['a:field:add:filter']" class="nc-column-filter nc-header-menu-item">
                <component :is="iconMap.filter" class="text-gray-700" />
                <!-- Filter by this field -->
                Filter by this field
              </div>
            </NcMenuItem>
          </NcTooltip>

          <NcTooltip
            :disabled="(isGroupBySupported && !isGroupByLimitExceeded) || isGroupedByThisField || !(isEeUI && !isPublic)"
          >
            <template #title>{{
              !isGroupBySupported
                ? "This field type doesn't support grouping"
                : isGroupByLimitExceeded
                ? 'Group by limit exceeded'
                : ''
            }}</template>
            <NcMenuItem
              :disabled="isEeUI && !isPublic && (!isGroupBySupported || isGroupByLimitExceeded) && !isGroupedByThisField"
              @click="
                filterOrGroupByThisField(
                  isGroupedByThisField ? SmartsheetStoreEvents.GROUP_BY_REMOVE : SmartsheetStoreEvents.GROUP_BY_ADD,
                )
              "
            >
              <div v-e="['a:field:add:groupby']" class="nc-column-groupby nc-header-menu-item">
                <component :is="iconMap.group" class="text-gray-700" />
                <!-- Group by this field -->
                {{ isGroupedByThisField ? "Don't group by this field" : 'Group by this field' }}
              </div>
            </NcMenuItem>
          </NcTooltip>

          <a-divider class="!my-0" />

          <NcMenuItem v-if="!column?.pk" :disabled="!isDuplicateAllowed" @click="openDuplicateDlg">
            <div v-e="['a:field:duplicate']" class="nc-column-duplicate nc-header-menu-item">
              <component :is="iconMap.duplicate" class="text-gray-700" />
              <!-- Duplicate -->
              {{ t('general.duplicate') }}
            </div>
          </NcMenuItem>
          <NcMenuItem @click="onInsertAfter">
            <div v-e="['a:field:insert:after']" class="nc-column-insert-after nc-header-menu-item">
              <component :is="iconMap.colInsertAfter" class="text-gray-700 !w-4.5 !h-4.5" />
              <!-- Insert After -->
              {{ t('general.insertAfter') }}
            </div>
          </NcMenuItem>
          <NcMenuItem v-if="!column?.pv" @click="onInsertBefore">
            <div v-e="['a:field:insert:before']" class="nc-column-insert-before nc-header-menu-item">
              <component :is="iconMap.colInsertBefore" class="text-gray-600 !w-4.5 !h-4.5" />
              <!-- Insert Before -->
              {{ t('general.insertBefore') }}
            </div>
          </NcMenuItem>
        </template>
        <a-divider v-if="!column?.pv" class="!my-0" />

        <NcMenuItem v-if="!column?.pv" :disabled="!isDeleteAllowed" class="!hover:bg-red-50" @click="handleDelete">
          <div class="nc-column-delete nc-header-menu-item text-red-600">
            <component :is="iconMap.delete" />
            <!-- Delete -->
            {{ $t('general.delete') }}
          </div>
        </NcMenuItem>
      </NcMenu>
    </template>
  </a-dropdown>
  <SmartsheetHeaderDeleteColumnModal v-model:visible="showDeleteColumnModal" />
  <DlgColumnDuplicate
    v-if="column"
    ref="duplicateDialogRef"
    v-model="isDuplicateDlgOpen"
    :column="column"
    :extra="selectedColumnExtra"
  />
</template>

<style scoped>
.nc-header-menu-item {
  @apply text-dropdown flex items-center gap-2;
}

.nc-column-options {
  .nc-icons {
    @apply !w-5 !h-5;
  }
}

:deep(.ant-dropdown-menu-item:not(.ant-dropdown-menu-item-disabled)) {
  @apply !hover:text-black text-gray-700;
}
:deep(.ant-dropdown-menu-item.ant-dropdown-menu-item-disabled .nc-icon) {
  @apply text-current;
}
</style>
