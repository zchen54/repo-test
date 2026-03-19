<template>
  <div ref="TaskListRef" class="relative">
    <!-- <b style="position: absolute; left: 10px; top: 2px">{{ dynamicKey }}</b> -->
    <!-- 拖动选中的矩形框 -->
    <div v-if="canSetThColor && isSelecting" class="selection-box" :style="selectionBoxStyle"></div>
    <!-- 框住选中的表头 -->
    <div v-if="canSetThColor && selectedHeaders.length" class="selected-headers-box" :style="selectedHeadersBoxStyle">
      <!-- 左右拖动框选表头的范围 和拖动列宽度有点冲突 而且选择表头后马上弹出设置背景色 这个功能就不必要了 暂不支持 -->
      <!-- <div class="resize-handle" @mousedown="handleResizeHandleMouseDown"></div> -->

      <el-color-picker
        ref="ThBgColorPicker"
        v-model="thBgColorValue"
        size="mini"
        :predefine="__ThBgColorEnum"
        class="th-bg-color-picker"
        @change="handleConfirmBgColor"
        @active-change="handleThBgActiveChange"
        @keyup.esc.native="handleCancelBgColor"
      ></el-color-picker>
    </div>

    <!-- <div
      class="dev-panel"
      style="position: fixed; right: 0; bottom: 0; z-index: 10000; padding: 10px; background-color: rgba(255, 255, 255, 0.7); text-align: right; pointer-events: none"
    >
      <div class="mt-8">
        {{ dynamicKey }}
        <el-button class="ml-10" style="pointer-events: auto" @click="logTableStore">Log</el-button>
      </div>
      <div v-if="Array.isArray(tableExpandKeys)" class="mt-8">tableExpandKeys: {{ tableExpandKeys.join(', ') }}</div>
      <div v-if="Array.isArray(planExpandKeys)" class="mt-8">planExpandKeys: {{ planExpandKeys.join(', ') }}</div>
    </div> -->

    <div class="task-list">
      <el-skeleton animated :loading="skeletonLoading" :rows="10">
        <div>
          <template v-if="canViewWorkflowItem() || isFilterSelector">
            <div v-if="tableData.length || isFilterSelector" ref="TaskListWrapRef" class="relative">
              <div
                v-if="tableLoading || batchUpdateLoading"
                v-loading="true"
                :element-loading-spinner="batchUpdateLoading ? 'el-icon-loading' : undefined"
                :element-loading-text="batchUpdateLoading ? translateTitle('common.batchUpdatingText') : undefined"
                class="list-loading-mask"
              ></div>
              <!-- <b style="position: absolute; top: -10px; left: -10px; z-index: 1000">{{ dynamicKey }}</b> -->
              <el-table
                ref="taskListTable"
                :key="dynamicKey"
                :data="tableData"
                :height="isReviewBugs ? undefined : tableHeight"
                :max-height="isReviewBugs ? tableHeight : undefined"
                :row-class-name="getRowClassName"
                highlight-current-row
                class="task-list-table"
                :class="{
                  'allow-drag-header': allowDragHeader,
                  'hidden-border': !canSetThColor && !hasGroupBy,
                  'allow-header-color': canSetThColor || hasGroupBy,
                  'group-by-table': hasGroupBy,
                  'config-version-table': isConfigWorkflow,
                  'tree-list-table': useLazyTreeTable,
                }"
                border
                :stripe="hasGroupBy ? undefined : true"
                row-key="rowKey"
                :default-expand-all="hasGroupBy"
                :lazy="useLazyTreeTable"
                :load="useLazyTreeTable ? loadChildren : undefined"
                :tree-props="hasGroupBy || useLazyTreeTable ? { children: 'children', hasChildren: 'hasChildren' } : undefined"
                :span-method="hasGroupBy ? groupSpanMethod : undefined"
                :select-on-indeterminate="hasGroupBy ? false : true"
                @header-dragend="onHeaderDragend"
                @current-change="handleCurrentChange"
                @row-click="handleTableRowClick"
                @row-contextmenu="rightClickRow"
                @select="handleTableSelect"
                @select-all="handleTableSelectAll"
                @selection-change="handleSelectionChange"
                @expand-change="onExpandChange"
              >
                <el-table-column v-if="showTableSelection" type="selection" width="32" :selectable="rowSelectable" reserve-selection></el-table-column>
                <el-table-column
                  v-for="(col, colIndex) in tbDynamicColumns"
                  :key="col.key || `${col.id}_${col.wkfId || 0}_${colIndex}`"
                  :column-key="`${col.id}_${col.wkfId || 0}`"
                  :label-class-name="`${labelClassNamePrefix}${col.wkfId || ''}_${col.id}`"
                  :label="col.title || col.label"
                  :width="col.width"
                  :align="col.align"
                  :class-name="col.className"
                >
                  <template slot="header">
                    <div
                      class="flex task-list-header"
                      :class="{ 'flex-center': col.align === 'center', 'groups-table-header': hasGroupBy }"
                      :style="`background-color: ${col.backgroundColor}; color: ${col.color}`"
                      @dblclick.stop="handleAutoAdjustWidth(col)"
                    >
                      <template v-if="!col.ignoreAllEvents">
                        <span class="text-overflow col-title" :class="{ 'has-wkfId': col.wkfId > 0 }" :title="col.title || col.label">{{ col.title || col.label }}</span>
                      </template>
                      <template v-if="(showHeaderMenus || showOrderBy) && col.fieldId !== FieldNameId && !col.ignoreAllEvents">
                        <el-popover
                          v-if="showHeaderMenus"
                          :ref="`AddColumnsPop_${col.id}_${col.wkfId || 0}_${colIndex}`"
                          placement="bottom"
                          width="160"
                          trigger="click"
                          popper-class="add-columns-popover"
                        >
                          <span slot="reference"></span>
                          <div class="columns-select">
                            <template v-if="addColumnsOptions.length">
                              <div v-for="group in addColumnsOptions" :key="group.label" class="columns-group">
                                <div v-if="addColumnsOptions.length > 1" class="group-label">{{ group.label }}</div>
                                <div class="group-options">
                                  <div
                                    v-for="opt in group.options"
                                    :id="`Option_${opt.id}`"
                                    :key="opt.id"
                                    class="option-item flex flex-between align-center text-overflow"
                                    :title="opt.name"
                                    @click="onColumnsSelect(opt, col, colIndex)"
                                  >
                                    <span>{{ opt.name }}</span>
                                  </div>
                                </div>
                              </div>
                            </template>
                            <div v-else class="no-columns">{{ translateTitle('task.TaskList.meiYouKeTianJia') }}</div>
                          </div>
                        </el-popover>
                        <DropdownMenu :menus="thMenus" :data="{ ...col, colIndex }" class="ml-4 col-controller">
                          <XSvgIcon icon="arrow-down-s-line" style="color: #999; font-size: 12px; vertical-align: middle; margin-top: -2px" />
                        </DropdownMenu>
                      </template>
                    </div>
                  </template>
                  <template slot-scope="{ row, $index }">
                    <template v-if="row.isGroup">
                      <template v-if="row.value === 'unset'">
                        <b class="group-label">{{ row.key }}:</b>
                        <span class="group-value ml-8 mr-10">{{ translateTitle('common.groupUnsetLabel') }}</span>
                        <template v-if="row._showCount && row._showCount > row._count">
                          <span class="group-count ml-10">{{ translateTitle('common.groupCountAndTotal', { num: row._count, total: row._showCount }) }}</span>
                        </template>
                        <span v-else-if="row._showCount" class="group-count ml-10">{{ translateTitle('common.groupCount', { num: row._showCount }) }}</span>
                      </template>
                      <template v-else>
                        <b class="group-label">{{ row.key }}:</b>
                        <span class="group-value ml-8 mr-10">{{ row.value }}</span>
                        <template v-if="row._showCount && row._showCount > row._count">
                          <span class="group-count ml-10">{{ translateTitle('common.groupCountAndTotal', { num: row._count, total: row._showCount }) }}</span>
                        </template>
                        <span v-else-if="row._showCount" class="group-count ml-10">{{ translateTitle('common.groupCount', { num: row._showCount }) }}</span>
                      </template>
                      <span class="group-count ml-10">{{ genGroupAggregateInfo(row) }}</span>
                    </template>
                    <template v-else>
                      <div
                        class="col-field-wrap flex-1 min-w-0"
                        :class="{ 'is-leaf': useLazyTreeTable && col.id === FieldNameId && !row.hasChildren && _get(row, [FieldNameId, 'level']) === 1 }"
                      >
                        <component
                          :is="getComponent(col, row)"
                          v-loading="_get(row, [col.fieldId, 'loading'])"
                          element-loading-spinner="el-icon-loading"
                          v-bind="{ ...getBindData(col, row, $index) }"
                          class="x-field-editor"
                          :class="{ 'is-required': getBindData(col, row, $index).isRequired }"
                          v-on="{ ...getEvent(col, row) }"
                        />
                      </div>
                    </template>
                  </template>
                </el-table-column>
                <el-table-column min-width="1" label-class-name="cell-for-scrollbar"></el-table-column>
              </el-table>
              <div class="flex flex-center align-center mt-8">
                <el-pagination
                  :current-page.sync="formData.pageNo"
                  :layout="pagerLayout"
                  :page-size="formData.pageSize"
                  :total="total"
                  :background="paginationBackground"
                  class="m-0"
                  @current-change="handlePageChange"
                  @size-change="handleSizeChange"
                />
                <el-button v-if="!isDashboardWidget" icon="el-icon-refresh" class="btn-refresh ml-20" plain @click="handleRefreshCurrentPage"></el-button>
              </div>
            </div>
            <DrEmptyPage v-else style="height: 80vh">
              <img src="../../../../assets/empty_images/noData.png" alt="" />
              <div class="mt-16 mb-20">
                {{ translateTitle('task.TaskList.ninHaiMeiYouGong') }}
                <span v-if="hasWorkflowPermission('canAddItem') && workflowInfo.issueTypeId !== WORKFLOW_TEST_RUN && !pageReadOnly">
                  {{ translateTitle('task.TaskList.quChuangJianYiGe') }}
                </span>
              </div>
              <div>
                <el-button
                  v-if="hasWorkflowPermission('canAddItem') && workflowInfo.issueTypeId !== WORKFLOW_TEST_RUN && !pageReadOnly"
                  type="primary"
                  icon="el-icon-plus"
                  @click="$emit('create-task')"
                >
                  {{ translateTitle('task.TaskList.chuangJianGongZuoXiang') }}
                </el-button>
              </div>
            </DrEmptyPage>
          </template>
          <NoAccess v-else type="canViewItem" style="height: 80vh" />
        </div>
      </el-skeleton>
    </div>

    <!-- 右键菜单：开始 -->
    <Vab-contextmenu v-if="showContextmenu" ref="contextmenu">
      <div v-for="(group, gIndex) in fileMenus" :key="gIndex">
        <div v-for="(item, index) in group" :key="index">
          <Vab-contextmenu-submenu v-if="item.children && item.children.length">
            <template slot="title">
              <x-svg-icon :icon="item.icon" :style="`color: ${item.color}`" />
              <span class="ml-8">{{ item.text }}</span>
            </template>
            <div v-for="(childGroup, cGIndex) in item.children" :key="cGIndex">
              <Vab-contextmenu-item
                v-for="(child, cIndex) in childGroup"
                :key="cIndex"
                :disabled="child.disabled"
                class="flex flex-between align-center"
                @click="child.onClick(currentRow, child)"
              >
                <span>
                  <x-svg-icon :icon="child.icon" :style="`color: ${child.color}`" />
                  <span class="ml-8">{{ child.text }}</span>
                </span>
                <span v-if="child.hotKey" class="contextmenu-item-hotkey ml-30">{{ child.hotKey }}</span>
              </Vab-contextmenu-item>
              <Vab-contextmenu-item v-if="cGIndex < item.children.length - 1" divider></Vab-contextmenu-item>
            </div>
          </Vab-contextmenu-submenu>
          <Vab-contextmenu-item v-else :disabled="item.disabled" class="flex flex-between align-center" @click="item.onClick(currentRow, item)">
            <span>
              <x-svg-icon :icon="item.icon" :style="`color: ${item.color}`" />
              <span class="ml-8">{{ item.text }}</span>
            </span>
            <span v-if="item.hotKey" class="contextmenu-item-hotkey ml-30">{{ item.hotKey }}</span>
          </Vab-contextmenu-item>
        </div>
        <Vab-contextmenu-item v-if="gIndex < fileMenus.length - 1" divider></Vab-contextmenu-item>
      </div>
    </Vab-contextmenu>
    <!-- 右键菜单：结束 -->

    <UpdateTaskDialog ref="UpdateTaskDialog" @item-updated="onItemUpdated" />

    <UpdateRequiredFieldsDialog ref="UpdateRequiredFieldsDialog" />

    <!-- 可追溯性 -->
    <TraceabilityDialog ref="TraceabilityDialog" />

    <DrImagePreview ref="DrImagePreview" />

    <ConfigItemDialog v-if="isConfigWorkflow" ref="ConfigItemDialog" @close="handleConfigItemDialogClose" />

    <!-- 配置升级 -->
    <CreateConfigVersionDialog
      v-if="isConfigWorkflow"
      ref="CreateConfigVersionDialog"
      :project-id="_get(workflowInfo, 'project.id')"
      :workflow-id="_get(workflowInfo, 'id')"
      @on-success="onConfigUpgradeCreated"
    />

    <FTADrawerDialog v-if="isFTAWorkflow" ref="FTADrawerDialog" />
  </div>
</template>
<script>
  import {
    isItemLocked,
    lockItem,
    unlockItem,
    getWorkflowItemsForListMode,
    getWorkflowListItemChildren,
    updateWorkflowItemField,
    getWorkflowItemForListMode,
    deleteWorkflowItem,
    getWorkflowItemInfo,
    getTestRunVersion,
  } from '@/api/project/task.js'
  import { mapGetters } from 'vuex'
  import { WORKFLOW_CHECKLIST, WORKFLOW_CONFIG_ITEM, WORKFLOW_DOCUMENT, WORKFLOW_FTA, WORKFLOW_TEST_RUN, WORKFLOW_WORK_LOG } from '@/store/workflowType'
  import { fieldDefaultConfig, isPlainTextField } from '@/utils/enum'
  import {
    FieldConfigItemId,
    FieldDeliverableId,
    FieldProgressId,
    FieldSpentEstimatedHourId,
    FieldSubjectId,
    propertyTypes,
    FieldDocumentHierarchyId,
    FieldClassDuration,
  } from '@/store/fieldType.js'
  import { translateTitle } from '@/utils/i18n'
  import {
    FieldClassChoice,
    FieldClassDate,
    FieldClassInt,
    FieldDescriptionId,
    FieldFlagsId,
    FieldParentId,
    FieldIdId,
    FieldSN,
    FieldNameId,
    FieldStatusId,
    formatFieldValue,
    FieldCreatedById,
    FieldSpentHourId,
    aggregate_func_count,
    aggregate_func_sum,
    aggregate_func_average,
    aggregate_func_min,
    aggregate_func_max,
  } from '@/store/fieldType'
  import propertyItems from '@/views/project/task/components/propertyItems'
  import TdName from '@/views/project/task/components/TdName.vue'
  import TdItemId from '@/views/project/task/components/TdItemId.vue'
  import TdItemSN from '@/views/project/task/components/TdItemSN'
  import { canViewWorkflowItem, hasWorkflowPermission, copyToClip } from '@/utils/tools'
  import workflowPermissionMixins from '@/views/project/task/workflowPermissionMixins'
  import _ from 'lodash'
  import $ from 'jquery'
  import XSvgIcon from '@/vab/components/XSvgIcon'
  import { getFilterFields } from '@/api/project/filter'
  import { getListPage, setListPage } from '@/utils/storage'
  import { checkBit } from '@/utils/enum'
  import { TASK_FLAGS } from '@/store/task'
  import { drFormatDuration, isHTMLElement } from '@/utils/common'
  import UpdateRequiredFieldsDialog from '@/views/project/task/components/UpdateRequiredFieldsDialog'
  import { Code_Required_Fields_Empty } from '@/utils/request'
  import { genCancelToken } from '@/utils/request.js'
  import { getFilterItems } from '@/api/project/filter'
  import SpentHourField from '@/views/project/task/components/SpentHourField.vue'
  // 代码块高亮
  import prismjs from '@public/static/tinymce/assets/prism.js'
  import '@public/static/tinymce/assets/prism.css'
  import DrImagePreview from '@/components/DrImagePreview'
  import pickImagesMixins from '@/components/DrImagePreview/pickImagesMixins'
  import ItemDescriptionMixins from '@/views/project/task/components/ItemDescriptionMixins'
  import { getReviewBugs } from '@/api/review/index'
  import { addRequestCancelFunc } from '@/utils/tools.js'
  import { getContrastTextColor } from '@/utils/tools'
  import ConfigItemDialog from '@/views/project/task/components/taskDetail/configItem/ConfigItemDialog.vue'
  import CreateConfigVersionDialog from '@/views/project/task/components/taskDetail/configItem/CreateConfigVersionDialog.vue'
  import { getConfigVersionConfigItems } from '@/api/project/artifact'
  import { getDashboardCacheById, setDashboardCache, dashboard_cache_TTL } from '@/utils/storage'
  import FTADrawerDialog from '@/views/FTA/FTADrawerDialog.vue'
  import { sortObjectByKey } from '@/utils/charts'
  import { directive, VabContextmenu, VabContextmenuItem, VabContextmenuSubmenu } from '@/extra/vabContextmenu'
  import { hasLicense, TRACE_LICENSE, hasPermission } from '@/store/permissions/permission'
  import UpdateTaskDialog from '@/views/project/task/components/UpdateTaskDialog'
  import TraceabilityDialog from '@/views/project/task/components/TraceabilityDialog'
  import { ItemRoutePath } from '@/store/cachedType'
  import { removeCacheAfterDeleteItem } from '@/utils/storage'
  import { isItemClosed, isItemResolved } from '@/store/task'

  // group by map 会把不同流程中相同值的 items 合并到一组里（例如优先级为高的故事和任务）
  export const map_key_prefix = 'map_key_'
  export const genItemsGroupByMap = (res, aggregates, sort = false) => {
    // console.log('aggregates', aggregates)
    let _map = {}
    if (res && Array.isArray(res.groups) && Array.isArray(res.list)) {
      res.list.forEach((groupItem) => {
        const item = _.cloneDeep(groupItem)
        let _key = res.groups.map((groupKey) => map_key_prefix + (_.get(item.groups, groupKey) || 'unset'))
        let _exist = _.get(_map, _key)
        let _count = _.get(item, 'count.value', 0)
        let _newCount = _count // 前端用来统计每组的数量（处理列表全选时需要根据数量判断是否全选）
        let _showCount = _.get(item, 'count.value') // 用来显示在 group title 上的数量
        if (_count && !isNaN(_count)) _count = Number(_count)
        // console.log('_exist', _exist, _key)
        if (_exist) {
          let _existCount = _.get(_exist, '_count', 0)
          if (_existCount && !isNaN(_existCount)) _existCount = Number(_existCount)
          _newCount = Number(_existCount || 0) + Number(_count || 0)

          if (_exist._showCount && !isNaN(_exist._showCount)) {
            _showCount = Number(_exist._showCount) + Number(_.get(item, 'count.value') || 0)
            item.count.value = Number(_exist._showCount) + Number(_.get(item, 'count.value') || 0)
          }

          if (aggregates && Array.isArray(aggregates) && aggregates.length) {
            // console.log(_.cloneDeep(aggregates), _.cloneDeep(_exist), _.cloneDeep(item))
            aggregates.forEach((aggregate) => {
              const _alias = typeof aggregate.alias === 'string' ? aggregate.alias.toLowerCase() : null
              const _existValue = _.get(_exist, `${_alias}.value`)
              const _curValue = _.get(item, `${_alias}.value`)
              if (_alias && _existValue && !isNaN(_existValue) && _curValue && !isNaN(_curValue)) {
                if (aggregate.function === aggregate_func_sum) {
                  item[_alias].value = Number(_existValue) + Number(_curValue)
                } else if (aggregate.function === aggregate_func_average) {
                  const _arr = res.list.filter((it) => res.groups.every((groupKey) => _.get(it.groups, groupKey) === _.get(item.groups, groupKey)))
                  const _valueArr = _arr.map((it) =>
                    _.get(it, `${_alias}.value`) && _.get(it, `count.value`) ? Number(_.get(it, `${_alias}.value`)) * Number(_.get(it, `count.value`)) : 0
                  )
                  const _groupCount = _arr.reduce((sum, it) => sum + (_.get(it, `count.value`) ? Number(_.get(it, `count.value`)) : 0), 0)
                  const _groupTotal = _valueArr.reduce((sum, val) => sum + val, 0)
                  item[_alias].value = _groupTotal / _groupCount
                  // console.log('_arr', _.cloneDeep(item.groups), _arr, _valueArr, _groupCount, _groupTotal)
                } else if (aggregate.function === aggregate_func_max) {
                  item[_alias].value = Number(_curValue) > Number(_existValue) ? Number(_curValue) : Number(_existValue)
                } else if (aggregate.function === aggregate_func_min) {
                  item[_alias].value = Number(_curValue) < Number(_existValue) ? Number(_curValue) : Number(_existValue)
                }
              }
            })
          }

          _.set(_map, _key, { ...item, items: [..._exist.items, ...item.items], _count: _newCount, _showCount })
        } else {
          _.set(_map, _key, { ...item, items: item.items, _count: _newCount, _showCount })
        }
      })
    }
    // console.log('_map', _.cloneDeep(_map))
    if (sort) {
      return sortObjectByKey(_map)
    } else {
      return _map
    }
  }

  export default {
    name: 'TaskList',
    directives: { contextmenu: directive },
    components: {
      XSvgIcon,
      VabContextmenu,
      VabContextmenuItem,
      VabContextmenuSubmenu,
      ...propertyItems,
      SpentHourField,
      TdName,
      TdItemId,
      TdItemSN,
      UpdateRequiredFieldsDialog,
      DrImagePreview,
      ConfigItemDialog,
      CreateConfigVersionDialog,
      FTADrawerDialog,
      UpdateTaskDialog,
      TraceabilityDialog,
    },
    mixins: [workflowPermissionMixins, pickImagesMixins, ItemDescriptionMixins],
    props: {
      multipleCheck: { type: Boolean, default: false },
      isRunCases: { type: Boolean, default: false }, // 执行用例列表
      recursively: { type: Boolean, default: false }, // 是否显示递归子用例
      listParentId: { type: Number, default: undefined },
      allowBatch: { type: Boolean, default: false },
      filter: { type: String, default: undefined },
      wholeWord: { type: Boolean, default: false },
      filterId: { type: Number, default: undefined },
      conditions: { type: [Array, Object], default: () => ({}) },
      orderBy: { type: Object, default: () => ({}) },
      groupBy: { type: Array, default: () => [] },
      aggregates: { type: Array, default: () => [] },
      where: { type: String, default: undefined },
      filterFields: { type: Array, default: () => [] },
      isFilterSelector: { type: Boolean, default: false }, // 过滤器
      isReviewBugs: { type: Boolean, default: false }, // 评审的缺陷
      columnsMap: { type: Object, default: () => ({}) }, // 过滤器 显示的列
      predefined: { type: Object, default: undefined }, // 未分配 未排期
      showHeaderMenus: { type: Boolean, default: true },
      showOrderBy: { type: Boolean, default: true },
      allowDragHeader: { type: Boolean, default: true },
      defaultTableHeight: { type: Number, default: undefined }, // 默认列表高度
      defaultTableWidth: { type: Number, default: undefined }, // 默认列表宽度
      defaultPageSize: { type: Number, default: undefined },
      paginationBackground: { type: Boolean, default: true },
      isDashboardWidget: { type: Boolean, default: undefined },
      dashboardWidgetKey: { type: String, default: undefined },
      useWidgetCache: { type: Boolean, default: undefined }, // 是否使用组件缓存（我当前的工作项组件 只缓存第一个 tab ）
      refreshItemByWS: { type: Boolean, default: true }, // 是否通过 websocket 刷新当前行
      maxHeightByParent: { type: Boolean, default: undefined }, // 根据父容器计算表格最大高度
      commentTargetItemId: { type: Number, default: undefined }, // 高亮相关的评论
      isMeetingMode: { type: Boolean, default: false },
      canSetThColor: { type: Boolean, default: false }, // 表头可设置颜色
      mode: { type: String, default: undefined }, // 模式
      isTreeListMode: { type: Boolean, default: false }, // 树状列表
      useQuickFilter: { type: Boolean, default: false }, // 快捷过滤
      quickFilterCondition: { type: Object, default: undefined }, // 快捷过滤条件
      quickFilterFieldIds: { type: Array, default: () => [] }, // 支持快捷过滤的字段
      showContextmenu: { type: Boolean, default: false }, // 显示右键菜单（暂时只在列表模式显示。筛选器不方便判断权限暂不显示）
      listReadonly: { type: Boolean, default: false },
      useSelection: { type: Boolean, default: false }, // 允许选择，目前在筛选条件中使用
      selectedIds: { type: Array, default: () => [] }, // 允许选择时，翻页后检查选中项
      testRunEditable: { type: Boolean, default: false }, // 测试执行能否编辑（限制能否点击更新用例）
    },
    data() {
      const group_name_column = { id: -1, label: '', width: 22, className: 'table-column--group-title', key: 'group_title', ignoreAllEvents: true }
      return {
        WORKFLOW_TEST_RUN,
        group_name_column,

        skeletonLoading: true,
        tableLoading: false,
        batchUpdateLoading: false,
        dynamicKey: 0,
        tableData: [],
        tbDynamicColumns: [],
        currentRow: null,
        multipleSelection: [],
        formData: { pageNo: 1, pageSize: 20 },
        layout: 'total, sizes, prev, pager, next, jumper',
        total: 0,
        configMap: {},
        throttle: false, // 列表刷新节流（应该用防抖，但发版前先不改，有风险，保持现状）
        debounceTimer: null, // 测试执行列表防抖
        FieldNameId,
        columnsConfig: {},
        addColumnsFields: [],
        tableHeight: undefined,
        filterFieldsMap: {},
        isDestroy: false,

        cancelReqMap: {},

        focusItem: null,
        sortable: null,
        scrollbarWidth: 17, // 浏览器滚动条宽度

        // ctrl 拖选表头设置颜色
        labelClassNamePrefix: 'dr_th_', // 表头 class
        isSelecting: false, // 正在拖动选择
        selectionStart: { x: 0, y: 0 },
        selectionEnd: { x: 0, y: 0 },
        selectedHeaders: [], // 已选中的表头
        modifierKeyPressed: false, // 按住了 ctrl
        isResizing: false, // 正在拖动调整列宽
        resizeStartX: 0,
        resizeStartWidth: 0,
        thBgColorValue: undefined, // 设置表头颜色
        reloadTable: false, // 切换视图后需要刷新表格
        useLazyTreeTable: false, // 懒加载树状表格
        showParagraph: false, // 显示文档序号
        tableExpandKeys: [], // 展开的行
        planExpandKeys: [], // 待展开的行
        lazyTreeNodeMap: new Map(), // 存储已加载节点
        skipSelectAllFunc: false,
        tableScrollTop: null, // 表格滚动距离

        loadingPromises: null,
        refreshQueue: [],
        isRefreshing: false,

        testRunVersionData: null,

        selectionChangeTriggered: false, // 树状列表全选时偶尔没触发 selection change
      }
    },
    computed: {
      ...mapGetters({
        collapse: 'settings/collapse', // 折叠左侧菜单
        workflowInfo: 'task/workflowInfo',
        orderByFields: 'task/orderByFields',
        userInfo: 'user/userInfo',
        User_Def_WorkflowListDetailMode: 'user/User_Def_WorkflowListDetailMode',
        IsBaselineView: 'project/IsBaselineView',
        workflowPermission: 'permission/userWorkflowPermission',
        projectReadonly: 'task/projectReadonly',
      }),

      // 拖动框选区域
      selectionBoxStyle() {
        const left = Math.min(this.selectionStart.x, this.selectionEnd.x)
        const top = Math.min(this.selectionStart.y, this.selectionEnd.y)
        const width = Math.abs(this.selectionEnd.x - this.selectionStart.x)
        const height = Math.abs(this.selectionEnd.y - this.selectionStart.y)

        return {
          left: `${left}px`,
          top: `${top}px`,
          width: `${width}px`,
          height: `${height}px`,
        }
      },

      // 框住已选中的表头
      selectedHeadersBoxStyle() {
        if (!this.selectedHeaders.length) return {}

        // Calculate the bounding box for all selected headers
        let minLeft = Infinity
        let minTop = Infinity
        let maxRight = 0
        let maxBottom = 0

        const wrapRect = this.$refs.TaskListRef.getBoundingClientRect()

        this.selectedHeaders.forEach((header) => {
          const rect = header.element.getBoundingClientRect()
          const relativeRect = {
            left: rect.left - wrapRect.left,
            top: rect.top - wrapRect.top,
            right: rect.right - wrapRect.left,
            bottom: rect.bottom - wrapRect.top,
          }

          minLeft = Math.min(minLeft, relativeRect.left)
          minTop = Math.min(minTop, relativeRect.top)
          maxRight = Math.max(maxRight, relativeRect.right)
          maxBottom = Math.max(maxBottom, relativeRect.bottom)
        })

        return {
          left: `${minLeft}px`,
          top: `${minTop}px`,
          width: `${maxRight - minLeft}px`,
          height: `${maxBottom - minTop}px`,
        }
      },

      pagerLayout() {
        if (this.isDashboardWidget) {
          if (this.defaultTableWidth < 500) {
            return 'total, prev, next'
          } else {
            return 'total, prev, pager, next'
          }
        } else {
          return this.layout
        }
      },

      isProjectReadonly() {
        // 不是筛选器（可能包含多项目）&& （列表模式/树状列表或子执行列表）&& 当前项目只读
        return !this.isFilterSelector && (!!this.mode || this.isRunCases) && this.projectReadonly
      },

      pageReadOnly() {
        return this.isLockedByAnother || this.IsBaselineView || this.isProjectReadonly || this.listReadonly
      },

      isConfigWorkflow() {
        return this.workflowInfo.issueTypeId === WORKFLOW_CONFIG_ITEM
      },

      isFTAWorkflow() {
        return this.workflowInfo.issueTypeId === WORKFLOW_FTA
      },

      isTestRunWorkflow() {
        return this.workflowInfo.issueTypeId === WORKFLOW_TEST_RUN
      },

      showTableSelection() {
        return (this.multipleCheck || ((this.isRunCases || this.isFilterSelector) && this.allowBatch)) && (!this.pageReadOnly || this.IsBaselineView || this.useSelection)
      },

      thMenus() {
        let menus = []
        if (this.useQuickFilter) {
          menus.push([
            {
              text: this.translateTitle('task.TaskList.quickFilter'),
              icon: 'filter-3-line',
              show: (data) => data.fieldType === FieldClassChoice && !data.referenceType && this.quickFilterFieldIds.includes(data.id),
              onClick: (data) => this.handleQuickFilter(data),
            },
          ])
        }
        if (this.showOrderBy) {
          menus.push([
            {
              text: this.translateTitle('task.TaskList.anCiLieShengXu'),
              icon: 'sort-asc',
              show: (data) => {
                if (this.isFilterSelector) {
                  return this.canOrderFields.some((item) => item.id === data.id) && _.get(this.orderBy, 'field') === data.id && _.get(this.orderBy, 'direction') === 'DESC'
                } else {
                  return this.orderByFields.some((item) => item.field === data.id) && _.get(this.orderBy, 'field') === data.id && _.get(this.orderBy, 'direction') === 'DESC'
                }
              },
              onClick: (data) => this.handleSortColumnASC(data),
            },
            {
              text: this.translateTitle('task.TaskList.anCiLieJiangXu'),
              icon: 'sort-desc',
              show: (data) => {
                if (this.isFilterSelector) {
                  return (this.canOrderFields.some((item) => item.id === data.id) && _.get(this.orderBy, 'field') !== data.id) || _.get(this.orderBy, 'direction') === 'ASC'
                } else {
                  return (this.orderByFields.some((item) => item.field === data.id) && _.get(this.orderBy, 'field') !== data.id) || _.get(this.orderBy, 'direction') === 'ASC'
                }
              },
              onClick: (data) => this.handleSortColumnDESC(data),
            },
          ])
        }
        if (this.showHeaderMenus) {
          menus.push([
            { text: this.translateTitle('task.TaskList.tianJiaYiLie'), icon: 'insert-column-right', divided: true, onClick: (data) => this.handleAddColumn(data) },
            { text: this.translateTitle('task.TaskList.yiChuCiLie'), icon: 'delete-column', onClick: (data) => this.handleRemoveColumn(data) },
            {
              text: this.translateTitle('task.TaskList.xiangZuoYiDong'),
              icon: 'arrow-left-line',
              show: (data) => (this.hasGroupBy ? data.colIndex > 1 : data.colIndex > 0),
              onClick: (data) => this.handleMoveColumnToLeft(data),
            },
            {
              text: this.translateTitle('task.TaskList.xiangYouYiDong'),
              icon: 'arrow-right-line',
              show: (data) => data.colIndex < this.tbDynamicColumns.length - 1,
              onClick: (data) => this.handleMoveColumnToRight(data),
            },
          ])
        }

        if (this.showHeaderMenus && this.isFilterSelector) {
          menus.push([
            {
              text: this.translateTitle('task.TaskList.setAsCommonColField'),
              icon: 'magic-line',
              show: (data) => data.wkfId > 0,
              onClick: (data) => this.handleSetAsCommonField(data),
            },
          ])
        }
        return menus
      },

      addColumnsOptions() {
        let groups = []
        if (this.isFilterSelector) {
          const _ids = this.tbDynamicColumns.map((col) => `${col.id}_${col.wkfId || 0}`)
          const _multi_workflows = Object.keys(this.columnsMap).length > 1

          Object.keys(this.columnsMap).forEach((key) => {
            if (Array.isArray(this.columnsMap[key].configs)) {
              const options = this.columnsMap[key].configs
                .filter((config) => config.isListable)
                .map((config) => ({ ...config, pid: this.columnsMap[key].projectId, wkfId: _multi_workflows ? this.columnsMap[key].workflowId : 0 }))
                .filter((config) => !_ids.includes(`${config.id}_${config.wkfId || 0}`))
              if (options.length) {
                if (key === 'default') {
                  groups.unshift({ label: this.translateTitle('DrFilter.DrFilterConfig.defaultFields'), options })
                } else {
                  groups.push({ label: key, options })
                }
              }
            }
          })
        } else {
          const _ids = this.tbDynamicColumns.map((col) => col.id)
          const options = this.addColumnsFields.filter((col) => !_ids.includes(col.id))
          if (options.length) {
            groups.push({ label: 'Default', options })
          }
        }
        return groups
      },

      canOrderFields() {
        const fields = []
        if (this.isFilterSelector) {
          Object.keys(this.columnsMap).forEach((key) => {
            if (Array.isArray(this.columnsMap[key].configs)) {
              const options = this.columnsMap[key].configs
                .filter((config) => config.orderBy)
                .map((config) => ({ ...config, projectId: this.columnsMap[key].projectId, workflowId: this.columnsMap[key].workflowId }))

              fields.push(...options)
            }
          })
        }
        return fields
      },

      hasGroupBy() {
        return !this.isTreeListMode && Array.isArray(this.groupBy) && this.groupBy.length > 0
      },

      canInsertChild() {
        const ignoreTypes = [WORKFLOW_TEST_RUN, WORKFLOW_DOCUMENT, WORKFLOW_CHECKLIST]
        const issueTypeId = _.get(this.workflowInfo, 'issueTypeId')
        return !ignoreTypes.includes(issueTypeId)
      },

      // 右键菜单
      fileMenus() {
        const menus = [
          [
            { text: this.translateTitle('common.detail'), icon: 'file-list-line', onClick: this.gotoProperty },
            {
              text: this.translateTitle('common.modify'),
              icon: 'edit-line',
              show: () => !this.pageReadOnly && this.currentRow && _.get(this.currentRow, 'editable'),
              onClick: () => this.editTask(),
            },
            { text: this.translateTitle('common.traceItem'), icon: 'flag-line', show: () => hasLicense(TRACE_LICENSE), onClick: this.openTraceability },
          ],
          [
            {
              text: this.translateTitle('common.oneClickExpandAll'),
              icon: 'node-tree',
              show: () => {
                const hasChildren =
                  this.useLazyTreeTable && this.currentRow && (this.currentRow.hasChildren || (Array.isArray(this.currentRow.children) && this.currentRow.children.length))
                if (!hasChildren) return false
                const expanded = _.get(this.$refs.taskListTable, ['store', 'states', 'treeData', this.currentRow.rowKey, 'expanded'], false)
                return !expanded
              },
              onClick: () => this.handleExpandAllChildren(this.currentRow),
            },
            {
              text: this.translateTitle('common.oneClickCollapseAll'),
              icon: 'collapse-icon',
              show: () => {
                const hasChildren =
                  this.useLazyTreeTable && this.currentRow && (this.currentRow.hasChildren || (Array.isArray(this.currentRow.children) && this.currentRow.children.length))
                if (!hasChildren) return false
                const expanded = _.get(this.$refs.taskListTable, ['store', 'states', 'treeData', this.currentRow.rowKey, 'expanded'], false)
                return expanded
              },
              onClick: () => this.handleCollapseAllChildren(this.currentRow),
            },
          ],
          [
            {
              text: this.translateTitle('common.insertChild'),
              icon: 'add-line',
              show: () => this.canInsertChild,
              onClick: this.onMenuInsertChild,
            },
          ],
          [
            {
              text: this.translateTitle('baseline.index.addConfigChild'),
              icon: 'add-line',
              show: () => !this.pageReadOnly && this.currentRow && this.isConfigWorkflow && checkBit(_.get(this.currentRow, `${FieldFlagsId}.value`, 0), TASK_FLAGS.FOLDER),
              onClick: this.handleAddChild,
            },
            {
              text: this.translateTitle('baseline.index.bsVersionUpgrade'),
              icon: 'file-copy-line',
              show: () => !this.pageReadOnly && this.currentRow && this.isConfigWorkflow && !checkBit(_.get(this.currentRow, `${FieldFlagsId}.value`, 0), TASK_FLAGS.FOLDER),
              onClick: this.handleConfigUpgrade,
            },
          ],
          [
            {
              text: this.translateTitle('common.moveToTrash'),
              icon: 'delete-bin-line',
              show: () => this.canDelete && !this.pageReadOnly,
              onClick: this.onMenuDelete,
            },
          ],
          [
            {
              text: this.translateTitle('common.shareMenu'),
              icon: 'share-line',
              children: [
                [
                  { text: this.translateTitle('common.copyItemId'), icon: 'icon-copy-id', onClick: this.shareItemId },
                  { text: this.translateTitle('common.copyItemLink'), icon: 'share-line', onClick: this.shareItemLink },
                ],
              ],
            },
          ],
        ]

        let showMenus = []

        const filterMenus = (list) => {
          let _arr = []
          list.forEach((group) => {
            let _group = []
            group.forEach((item) => {
              if (this.currentRow) {
                if (Array.isArray(item.children)) {
                  let _children = filterMenus(item.children)
                  if (_children.length) {
                    if (item.show ? item.show(this.currentRow) : true) {
                      _group.push({ ...item, children: _children })
                    }
                  }
                } else if (item.show ? item.show(this.currentRow) : true) {
                  _group.push(item)
                }
              }
            })
            if (_group.length) {
              _arr.push(_group)
            }
          })
          return _arr
        }

        showMenus = filterMenus(menus)

        return showMenus
      },
    },
    watch: {
      collapse() {
        // 折叠菜单时重新计算列宽
        setTimeout(() => {
          this.adjustColumnWidths()
        }, 100)
      },
      hasGroupBy() {
        this.skeletonLoading = true
      },
      useLazyTreeTable() {
        // 在普通列表切换为树状列表时，需要重新渲染 el-table
        this.dynamicKey++
      },
      showTableSelection(val) {
        if (!val) {
          this.clearMultipleSelection()
        }
      },
      // dynamicKey() {
      //   console.log('dynamicKey', this.dynamicKey, this.useLazyTreeTable, this.tableExpandKeys.length > 0)
      //   if (this.useLazyTreeTable && this.tableExpandKeys.length > 0) {
      //     this.planExpandKeys = _.cloneDeep(this.tableExpandKeys)
      //     this.tableLoading = true
      //     this.resetLazyTreeTableStates()
      //     setTimeout(() => {
      //       this.autoExpandAfterReload()
      //     }, 100)
      //   }
      // },
    },
    created() {
      // setTimeout(() => {
      //   document.getElementById('Drome-ALM-Pro').addEventListener('click', this.blurAll)
      // }, 50)
      this.getAddColumnsOptions()

      this.initPageCache()
    },
    mounted() {
      this.$baseEventBus.$on('on-workflow-item-delete', this.onWorkflowItemDelete)
      this.$baseEventBus.$on('on-workflow-item-change', this.onWorkflowItemChange)
      this.$baseEventBus.$on('on-test-run-finish', this.onTestRunFinish)
      this.$baseEventBus.$on('after-merge-suspected', this.afterMergeSuspected)
      this.$baseEventBus.$on('after-clear-suspected', this.afterClearSuspected)
      this.$baseEventBus.$on('on-association-suspected-merge', this.onAssociationSuspectedMerge)
      this.$baseEventBus.$on('ws-update-list-fields', this.onWSUpdateListFields)
      this.$baseEventBus.$on('on-detail-dialog-no-permission', this.setListItemNoPermission)
      this.$baseEventBus.$on('refresh-task-list', this.onRefreshTaskList)
      this.$baseEventBus.$on('ws-on-batch-update-list-items', this.onWSBatchUpdateListFields)
      this.$baseEventBus.$on('open-config-item-dialog', this.handleOpenConfigItemDialog)
      this.$baseEventBus.$on('open-config-upgrade-dialog', this.handleOpenConfigUpgradeDialog)
      window.addEventListener('resize', this.calcTableHeight)
      document.addEventListener('contextmenu', this.handleGlobalContextmenu)

      // 选中表头 Start
      if (this.canSetThColor) {
        window.addEventListener('keydown', this.handleKeyDown)
        window.addEventListener('keyup', this.handleKeyUp)
        document.addEventListener('mousemove', this.handleResizeMouseMove)
        document.addEventListener('mouseup', this.handleResizeMouseUp)
        document.querySelector('#Drome-ALM').addEventListener('click', this.handleAppClick)
      }
      // 选中表头 End

      // 获取滚动条宽度
      this.scrollbarWidth = this.getScrollbarWidth()
      this.calcTableHeight()
    },
    beforeDestroy() {
      if (this.debounceTimer) clearTimeout(this.debounceTimer)
      this.isDestroy = true
      this.$baseEventBus.$off('on-workflow-item-delete', this.onWorkflowItemDelete)
      this.$baseEventBus.$off('on-workflow-item-change', this.onWorkflowItemChange)
      this.$baseEventBus.$off('on-test-run-finish', this.onTestRunFinish)
      this.$baseEventBus.$off('after-merge-suspected', this.afterMergeSuspected)
      this.$baseEventBus.$off('after-clear-suspected', this.afterClearSuspected)
      this.$baseEventBus.$off('on-association-suspected-merge', this.onAssociationSuspectedMerge)
      this.$baseEventBus.$off('ws-update-list-fields', this.onWSUpdateListFields)
      this.$baseEventBus.$off('on-detail-dialog-no-permission', this.setListItemNoPermission)
      this.$baseEventBus.$off('refresh-task-list', this.onRefreshTaskList)
      this.$baseEventBus.$off('ws-on-batch-update-list-items', this.onWSBatchUpdateListFields)
      this.$baseEventBus.$off('open-config-item-dialog', this.handleOpenConfigItemDialog)
      this.$baseEventBus.$off('open-config-upgrade-dialog', this.handleOpenConfigUpgradeDialog)
      window.removeEventListener('resize', this.calcTableHeight)
      document.removeEventListener('contextmenu', this.handleGlobalContextmenu)

      // 选中表头 Start
      if (this.canSetThColor) {
        window.removeEventListener('keydown', this.handleKeyDown)
        window.removeEventListener('keyup', this.handleKeyUp)
        document.removeEventListener('mousemove', this.handleResizeMouseMove)
        document.removeEventListener('mouseup', this.handleResizeMouseUp)
        const DromeAPP = document.querySelector('#Drome-ALM')
        if (DromeAPP) {
          DromeAPP.removeEventListener('click', this.handleAppClick)
        }

        // 移除表格横向滚动监听
        if (this.$refs.taskListTable && this.$refs.taskListTable.$el) {
          // const headerWrapper = this.$refs.taskListTable.$el.querySelector('.el-table__header-wrapper')
          // if (headerWrapper) {
          //   headerWrapper.removeEventListener('scroll', this.handleTableScroll)
          // }
          const bodyWrapper = this.$refs.taskListTable.$el.querySelector('.el-table__body-wrapper')
          if (bodyWrapper) {
            bodyWrapper.removeEventListener('scroll', this.handleTableScroll)
          }
        }
      }
      // 选中表头 End

      this.checkLockItemAndUnlock()

      this.cancelUnfinishedRequest()
      // this.remove()

      if (this.sortable) {
        this.sortable.destroy()
      }
      this.sortable = null
    },
    methods: {
      _get: _.get,
      translateTitle,
      isPlainTextField,
      canViewWorkflowItem,
      hasWorkflowPermission,

      // 获取滚动条宽度
      getScrollbarWidth() {
        const outer = document.createElement('div')
        outer.style.visibility = 'hidden'
        outer.style.overflow = 'scroll'
        document.body.appendChild(outer)

        const inner = document.createElement('div')
        outer.appendChild(inner)

        const width = outer.offsetWidth - inner.offsetWidth
        document.body.removeChild(outer)
        return width
      },

      // // reload table
      // reRenderTableByDynamicKey() {
      //   this.getTableScrollHeight()
      //   this.dynamicKey++
      //   setTimeout(() => {
      //     this.scrollBackAfterReloadTable()
      //   }, 100)
      // },

      // // 获取 taskListTable 滚动的高度
      // getTableScrollHeight() {
      //   const table = this.$refs.taskListTable
      //   if (table && table.bodyWrapper) {
      //     this.tableScrollTop = table.bodyWrapper.scrollTop
      //   }
      // },

      // // 刷新表格后，回到之前滚动的位置
      // scrollBackAfterReloadTable() {
      //   if (this.tableScrollTop !== null) {
      //     this.$nextTick(() => {
      //       const table = this.$refs.taskListTable
      //       if (table && table.bodyWrapper) {
      //         table.bodyWrapper.scrollTop = this.tableScrollTop
      //       }
      //     })
      //   }
      // },

      initPageCache() {
        if (this.isFilterSelector) {
          if (this.defaultPageSize) {
            this.formData.pageSize = this.defaultPageSize
          }
        } else {
          const { workflowId, runItemId } = this.$route.params
          const _key = this.isRunCases ? `${workflowId}_${runItemId}` : workflowId
          const _page = getListPage(_key)
          if (_page) {
            this.formData.pageNo = +_page.pageNo
            this.formData.pageSize = +_page.pageSize
            if (this.formData.pageSize === 0) {
              this.formData.pageSize = 20
            }
          }
        }
      },

      deleteItemInLazyTree(itemId) {
        if (!itemId) return

        // 找到删除的元素及子项
        let deletedIds = [itemId] // includes children
        this.findItem(itemId, (row, index, siblings, parent) => {
          const loopChildren = (children) => {
            if (Array.isArray(children) && children.length) {
              children.forEach((child) => {
                if (child.itemId) {
                  deletedIds.push(child.itemId)
                }
                if (child.children) {
                  loopChildren(child.children)
                }
              })
            }
          }

          if (row.children) {
            loopChildren(row.children)
          }
        })

        if (this.multipleSelection && this.multipleSelection.length) {
          // 清除已选中但被删除的项
          this.multipleSelection = this.multipleSelection.filter((item) => !deletedIds.includes(item.itemId))
          this.$emit('selection-change', this.multipleSelection)

          // 在 table 缓存中，清除已选中但被删除的项
          const table = this.$refs.taskListTable
          let selection = _.get(table, 'store.states.selection')
          if (Array.isArray(selection)) {
            let _newSelection = selection.filter((item) => !deletedIds.includes(item.itemId))
            this.$set(table.store.states, 'selection', _newSelection)
          }
        }

        // 移除 table data 中被删除的项
        this.findItem(itemId, (row, index, siblings, parent) => {
          siblings.splice(index, 1)
        })

        this.$nextTick(() => {
          this.checkTableAllSelected()
        })
      },
      onWorkflowItemDelete(data) {
        this.$emit('on-item-deleted', data)

        if (this.useLazyTreeTable) {
          if (data && data.itemId) {
            this.deleteItemInLazyTree(data.itemId)
          }
        } else {
          if (this.isRunCases) {
            this.$emit('refresh-run-cases-tree', data)
          }

          this.handleGetWorkflowItems(null, false)
        }
      },
      removeItemsAfterMoved(itemIds) {
        if (this.useLazyTreeTable) {
          if (Array.isArray(itemIds)) {
            itemIds.forEach((itemId) => {
              this.deleteItemInLazyTree(itemId)
            })
          }
        } else {
          this.handleGetWorkflowItems(null, false)
        }
      },
      onWorkflowItemChange(data) {
        if (this.IsBaselineView) return

        if (this.$store.getters['task/taskDetailDialogOpened']) {
          this.handleGetWorkflowItems(null, false)
        } else {
          this.refreshItemById(data.id)
        }
      },
      onAssociationSuspectedMerge(data) {
        this.handleGetWorkflowItems(null, false)
      },
      onRefreshTaskList(data, parent) {
        if (data) {
          if (data.itemId) {
            this.refreshItemById(data.itemId)
          }
        } else {
          if (!this.isTreeListMode) {
            this.handleGetWorkflowItems(null, false)
          }
        }

        if (this.isTreeListMode && parent && parent.itemId) {
          this.refreshChildrenById(parent.itemId)
        }
      },
      onTestRunFinish(data) {
        this.refreshItemById(data.itemId)
      },

      refreshChildrenById(rowId) {
        // console.log('refreshChildrenById', rowId, this.lazyTreeNodeMap.has(rowId))
        if (this.loadingPromises && this.loadingPromises[rowId]) {
          delete this.loadingPromises[rowId]
        }

        const { lazyTreeNodeMap } = this.$refs.taskListTable.store.states

        if (this.lazyTreeNodeMap.has(rowId)) {
          // console.log('has loaded')
          const { row: cachedRow, treeNode, resolve } = this.lazyTreeNodeMap.get(rowId)

          const p = this.loadChildren(cachedRow, treeNode, resolve)
          if (p && typeof p.then === 'function') {
            p.then((children) => {
              // console.log('store', this.$refs.taskListTable.store.states)
              // console.log('p.then', children)
              if (children && children.length > 0) {
                this.$refs.taskListTable && this.$refs.taskListTable.toggleRowExpansion(cachedRow, true)
              }

              this.$nextTick(() => {
                this.checkTableAllSelected()
              })
            })
          }
        } else if (_.get(lazyTreeNodeMap, `item_${rowId}`)) {
          // 更新子项
          // console.log('更新子项', rowId)
          this.loadChildrenAndExecute(rowId, null, true)
        } else {
          this.findItem(rowId, (row, index, siblings, parent) => {
            this.$set(row, 'hasChildren', true)
            this.$set(row, 'children', [])
          })

          setTimeout(() => {
            const expandBtn = document.querySelector(`.Row_${rowId} .el-table__expand-icon`)
            // console.log('expandBtn', expandBtn)
            if (expandBtn && typeof expandBtn.click === 'function' && !expandBtn.classList.contains('el-table__expand-icon--expanded')) {
              expandBtn.click()
            }
          }, 100)
        }
      },

      calcTableHeight() {
        if (!this.useSelection) {
          this.clearMultipleSelection()
        }

        if (this.defaultTableHeight) {
          this.tableHeight = this.defaultTableHeight
          return
        }

        const _appHeight = document.getElementById('Drome-ALM-Pro').clientHeight
        if (this.maxHeightByParent) {
          const parent = this.$refs.TaskListRef.parentElement
          const innerHeight = parent.clientHeight // 父容器内部高度
          const computedStyle = window.getComputedStyle(parent)
          const paddingTop = computedStyle.paddingTop
          const paddingBottom = computedStyle.paddingBottom
          const footerHeight = 40 // 表格底部和分页高度
          this.tableHeight = innerHeight - parseFloat(paddingTop) - parseFloat(paddingBottom) - footerHeight
        } else {
          if (this.isRunCases) {
            this.tableHeight = _appHeight - 190
          } else if (this.isReviewBugs) {
            this.tableHeight = _appHeight - 280
          } else if (this.isFilterSelector && this.predefined) {
            this.tableHeight = _appHeight * 0.9 - 240
          } else {
            this.tableHeight = _appHeight - 160
          }
        }

        // 窗口大小改变时重新计算列宽
        this.$nextTick(() => {
          this.adjustColumnWidths()
        })
      },

      onExpandChange(row, expanded) {
        if (!this.useLazyTreeTable) return

        const id = _.get(row, [FieldIdId, 'value'])
        const rowKey = row && row.rowKey ? row.rowKey : `item_${id}`

        if (expanded) {
          if (!this.tableExpandKeys.includes(rowKey)) {
            this.tableExpandKeys.push(rowKey)
          }
        } else {
          let childrenIds = []
          let removeRowKeys = [rowKey]
          const findAllChildren = (data) => {
            if (Array.isArray(data)) {
              data.forEach((item) => {
                childrenIds.push(_.get(item, [FieldIdId, 'value']))
                removeRowKeys.push(item.rowKey)
                if (Array.isArray(item.children) && item.children.length > 0) {
                  findAllChildren(item.children)
                }
              })
            }
          }

          findAllChildren(row.children)
          this.tableExpandKeys = this.tableExpandKeys.filter((key) => !removeRowKeys.includes(key))

          // this.$set(row, 'children', [])
          // if (this.loadingPromises && this.loadingPromises[id]) {
          //   delete this.loadingPromises[id]
          // }
          // if (this.lazyTreeNodeMap && typeof this.lazyTreeNodeMap.delete === 'function') {
          //   this.lazyTreeNodeMap.delete(id)
          // }
          // const treeData = this.$refs.taskListTable.store.states.treeData
          // if (treeData[row.rowKey]) {
          // this.$set(treeData[row.rowKey], 'loaded', false)
          // this.$set(treeData[row.rowKey], 'expanded', false)
          // }
        }
      },

      async getAddColumnsOptions() {
        if (this.isReviewBugs) {
        } else if (this.isFilterSelector) {
          // this.addColumnsFields = this.columnsFieldsMap
        } else {
          const { projectId, workflowId } = this.$route.params
          const { bsid } = this.$route.query
          const res = await getFilterFields({ pid: projectId, wkfId: workflowId, bsid: bsid ? +bsid : undefined })
          let arr = []
          if (Array.isArray(res.default)) {
            arr.push(...res.default.sort((a, b) => a.offset - b.offset))
          }
          if (Array.isArray(res.reference)) {
            arr.push(...res.reference.sort((a, b) => a.offset - b.offset))
          }
          this.addColumnsFields = arr
        }
      },

      getFilterSearchList(params, showSkeletonLoading = true) {
        if (showSkeletonLoading) {
          this.skeletonLoading = true
          this.configMap = {}
          this.tableData = []
        }
        this.skeletonLoading = false
        this.tableLoading = true

        let propertyList = []

        if (Array.isArray(this.filterFields)) {
          let _obj = {}
          this.filterFields.forEach((item) => {
            _obj[`${item.wkfId || 0}_${item.fid}`] = item
          })
          this.filterFieldsMap = _obj
        }

        let columns = []
        this.filterFields.forEach((field) => {
          let hasFound = false
          if (this.columnsMap.default && Array.isArray(this.columnsMap.default.configs) && !field.wkfId) {
            this.columnsMap.default.configs.some((config) => {
              if (config.id === field.fid) {
                columns.push({ ...config, pid: this.columnsMap.default.projectId, wkfId: 0 })
                hasFound = true
              }
            })
          }

          if (!hasFound) {
            Object.keys(this.columnsMap).some((key) => {
              if (Object.keys(this.columnsMap).length === 1 || !field.wkfId || field.wkfId === this.columnsMap[key].workflowId) {
                if (key !== 'default' && Array.isArray(this.columnsMap[key].configs)) {
                  return this.columnsMap[key].configs.some((config) => {
                    if (config.id === field.fid) {
                      columns.push({ ...config, pid: this.columnsMap[key].projectId, wkfId: field.wkfId })
                      return true
                    }
                  })
                }
              }
            })
          }
        })

        // console.log('columns 333', _.cloneDeep(columns), _.cloneDeep(this.filterFields))

        let _tempMap = {}
        columns.forEach((item) => {
          _tempMap[item.id] = item
        })
        this.configMap = _tempMap
        propertyList = columns.filter((config) => config.id === FieldNameId || !config.isHidden).map(this.formatColObj)

        if (!this.useSelection || !this.tbDynamicColumns.length) {
          if (this.hasGroupBy) {
            this.tbDynamicColumns = [this.group_name_column, ...propertyList]
          } else {
            this.tbDynamicColumns = propertyList
          }
        }
        this.$nextTick(() => {
          this.adjustColumnWidths()
        })

        if (!this.useSelection) {
          this.dynamicKey++
        }
        this.doAfterTableRender()

        const data = {
          conditions: this.conditions,
          where: this.where,
          orderBy: this.orderBy,
          groupBy: this.hasGroupBy ? JSON.stringify(this.groupBy) : undefined,
          aggregates: this.hasGroupBy ? JSON.stringify(this.aggregates) : undefined,
          fields: this.filterFields,
          ...this.formData,
        }
        // console.log('search', _.cloneDeep(data))
        data.conditions = JSON.stringify(data.conditions)
        data.orderBy = JSON.stringify(data.orderBy)

        if (this.predefined) {
          data.predefined = this.predefined
        }

        if (this.isDashboardWidget && this.dashboardWidgetKey) {
          if (this.useWidgetCache && this.formData.pageNo === 1) {
            // 我当前的工作项组件 获取第一页缓存的数据
            const _storage = getDashboardCacheById(this.dashboardWidgetKey)
            if (_storage && _storage.data && _storage.updatedAt && Date.now() - _storage.updatedAt < dashboard_cache_TTL) {
              const _data = JSON.parse(_storage.data)
              if (_data) {
                // 十分钟内再次请求时使用缓存
                this.formData.pageNo = _data.pageNo
                this.formData.pageSize = _data.pageSize
                this.total = _data.total
                this.tableData = _data.tableData
                this.skeletonLoading = false
                this.tableLoading = false
                return
              }
            }
          } else {
            setDashboardCache({ i: this.dashboardWidgetKey })
            this.$emit('check-widget-storage')
          }
        }

        if (typeof this.cancelReqMap.cancelLoadFilterItemList === 'function') {
          this.cancelReqMap.cancelLoadFilterItemList()
        }
        const cancelToken = genCancelToken((cancel) => {
          addRequestCancelFunc('cancelLoadFilterList', cancel)
          this.cancelReqMap.cancelLoadFilterItemList = cancel
        })
        getFilterItems(data, cancelToken).then(
          (res) => {
            this.tableLoading = false
            if (res) {
              this.total = res.total || 0
              if (Array.isArray(res.list) && res.list.length) {
                if (this.hasGroupBy) {
                  if (res.groups) {
                    const _map = genItemsGroupByMap(res, this.aggregates)

                    let _groups = []
                    const genGroups = (obj, index = 0, path = []) => {
                      return Object.keys(obj)
                        .map((key) => {
                          let _path = [...path, key]
                          let _item = {
                            ...obj[key],
                            items: undefined,
                            isGroup: true,
                            key: res.groups[index],
                            value: key.replace(map_key_prefix, ''),
                            rowKey: _path.join('_'),
                            level: _path.length,
                            _count: Number(_.get(obj[key], '_count', 0)),
                          }
                          if (Array.isArray(obj[key].items)) {
                            return {
                              ..._item,
                              _count: obj[key]._count,
                              children: obj[key].items.map((item) => {
                                return { ...item, ...item.fields, rowKey: `${_item.rowKey}_item_${item.id}`, level: _path.length + 1 }
                              }),
                            }
                          } else {
                            return { ..._item, _count: 0, children: genGroups(obj[key], index + 1, _path) }
                          }
                        })
                        .sort((a, b) => (a.value === 'unset' ? 1 : b.value === 'unset' ? -1 : 0))
                        .filter((group) => Array.isArray(group.children) && group.children.length)
                    }
                    _groups = genGroups(_map)

                    const sumGroupCounts = (groups) => {
                      if (!Array.isArray(groups)) return 0
                      return groups.reduce((sum, group) => {
                        if (Array.isArray(group.children) && group.children.length > 0) {
                          // 递归对子分组求和
                          group._count = sumGroupCounts(group.children)
                          return sum + group._count
                        } else {
                          // 叶子分组，count 已经是最终值
                          return sum + 1
                        }
                      }, 0)
                    }
                    const sumGroupTotal = (groups) => {
                      if (!Array.isArray(groups)) return 0
                      return groups.reduce((sum, group) => {
                        if (group._count) {
                          return sum + group._count
                        } else if (Array.isArray(group.children) && group.children.length > 0) {
                          group._count = sumGroupTotal(group.children)
                          return sum + group._count
                        } else {
                          return sum
                        }
                      }, 0)
                    }
                    sumGroupCounts(_groups)
                    sumGroupTotal(_groups)

                    this.tableData = _groups

                    // setTimeout(() => {
                    //   console.log('this.tableData', _.cloneDeep(this.tbDynamicColumns), _.cloneDeep(this.tableData))
                    // }, 1000)
                  }
                } else {
                  this.tableData = res.list.map((item) => {
                    return { ...item, ...item.fields, rowKey: `item_${item.id}` }
                  })
                }

                if (this.reloadTable) {
                  this.dynamicKey++
                  this.reloadTable = false
                }

                if (this.isDashboardWidget && this.dashboardWidgetKey) {
                  if (this.useWidgetCache && this.formData.pageNo === 1) {
                    // 我当前的工作项组件 缓存第一页数据
                    let _data = {
                      i: this.dashboardWidgetKey,
                      data: {
                        pageNo: this.formData.pageNo,
                        pageSize: this.formData.pageSize,
                        total: this.total,
                        tableData: this.tableData,
                      },
                    }
                    setDashboardCache(_data)
                    this.$emit('check-widget-storage')
                  } else {
                    setDashboardCache({ i: this.dashboardWidgetKey })
                    this.$emit('check-widget-storage')
                  }
                }
              } else if (this.formData.pageNo > 1) {
                this.formData.pageNo = this.formData.pageNo - 1
                this.getFilterSearchList()
              } else {
                this.tableData = []
              }

              if (this.tableData.length) {
                this.adjustColumnWidths()
                setTimeout(() => {
                  if (this.useSelection) {
                    this.checkTableSelectionByIds()
                  }

                  if (this.sortable) {
                    this.sortable.destroy()
                  }
                  this.columnDrop()

                  // 调用列宽度自适应方法
                }, 100)
              }
            }
          },
          (error) => {
            if (!(error && error.message === 'Request canceled by user')) {
              this.tableLoading = false
            }
          }
        )
      },

      groupSpanMethod({ row, column, rowIndex, columnIndex }) {
        if (row.isGroup) {
          if (column.type === 'selection') {
            return [1, 1]
          } else if (column.className === 'table-column--group-title') {
            return [1, this.tbDynamicColumns.length]
          } else {
            return [0, 0]
          }
        }
      },

      getReviewBugList() {
        this.tableLoading = true
        getReviewBugs({ ...this.predefined, ...this.formData }).then((res) => {
          this.skeletonLoading = false
          this.tableLoading = false
          if (res) {
            this.total = res.total || 0
            if (Array.isArray(res.list) && res.list.length) {
              let _fieldsMap = _.get(res.list, '0.fields')
              let _tempMap = {}
              Object.values(_fieldsMap).forEach((item) => {
                _tempMap[item.config.id] = item.config
              })
              this.configMap = _tempMap

              const propertyList = Object.values(this.configMap).map(this.formatColObj)
              this.tbDynamicColumns = propertyList

              this.tableData = res.list.map((item) => {
                return { ...item, ...item.fields }
              })
              if (this.reloadTable) {
                this.dynamicKey++
                this.reloadTable = false
              }
            } else if (this.formData.pageNo > 1) {
              this.formData.pageNo = this.formData.pageNo - 1
              this.getReviewBugList()
            } else {
              this.tableData = []
            }
          } else {
            this.tableData = []
          }
        })
      },

      // 调整列宽度。优先级：视图配置 > 字段配置 > 调整名称宽度（无任何宽度配置时，将多出来的宽度分给名称列）
      adjustColumnWidths() {
        // 检查是否需要执行自适应宽度调整，如果视图有保存宽度，则不调整
        if (!this.shouldAdjustColumnWidths()) {
          return
        }

        // 获取容器宽度
        if (!this.$refs.TaskListWrapRef) return

        const containerWidth = this.$refs.TaskListWrapRef.clientWidth
        if (!containerWidth) return

        // 计算所有列的总宽度
        let totalColumnsWidth = 0
        let nameColumnIndex = -1

        this.tbDynamicColumns.forEach((col, index) => {
          if (col.fieldId === FieldNameId) {
            nameColumnIndex = index
          } else {
            totalColumnsWidth += parseInt(col.width || 0)
          }
        })

        // 如果有选择列，添加其宽度
        if (this.showTableSelection) {
          totalColumnsWidth += 32 // 选择列宽度
        }

        totalColumnsWidth += this.scrollbarWidth + 1 // 减去纵向滚动条的宽度
        // 如果总宽度小于容器宽度且找到了名称列
        if (totalColumnsWidth < containerWidth && nameColumnIndex !== -1) {
          // 计算多余的宽度
          const extraWidth = containerWidth - totalColumnsWidth

          // 将多余宽度分配给名称列
          const nameColumn = this.tbDynamicColumns[nameColumnIndex]
          let newWidth

          let nameFieldMinWidth = 240 // 默认名称列最小宽度
          if (this.isDashboardWidget) {
            nameFieldMinWidth = 170 // 百科小组件中，名称列最小宽度。兼容以前的配置，尽量优化显示效果
          }

          if (nameColumn.colspan) {
            newWidth = nameColumn.colspan // 字段配置优先
          } else if (extraWidth < nameFieldMinWidth) {
            newWidth = nameFieldMinWidth
          } else {
            newWidth = extraWidth
          }

          // 更新名称列宽度
          this.$set(this.tbDynamicColumns[nameColumnIndex], 'width', newWidth)
          // this.dynamicKey++ // 触发表格重新渲染
          this.doAfterTableRender()
        }
      },

      // 检查是否应该调整列宽度
      shouldAdjustColumnWidths() {
        if (!(Array.isArray(this.filterFields) && this.filterFields.length)) {
          return true
        }

        const hasWidth = this.filterFields.some((field) => {
          return field.width > 0
        })

        return !hasWidth
      },

      handleRefreshCurrentPage() {
        if (this.useLazyTreeTable) {
          this.reloadTable = true
          this.lazyTreeNodeMap = new Map()
          this.loadingPromises = null
        }

        this.handleGetWorkflowItems()
      },

      handleGetWorkflowItems(params, showSkeletonLoading = true) {
        if (this.isRunCases) {
          if (this.debounceTimer) clearTimeout(this.debounceTimer)
          this.debounceTimer = setTimeout(() => {
            this._handleGetWorkflowItems(params, showSkeletonLoading)
          }, 100)
        } else {
          this._handleGetWorkflowItems(params, showSkeletonLoading)
        }
      },

      _handleGetWorkflowItems(params, showSkeletonLoading = true) {
        if (this.isReviewBugs) {
          this.getReviewBugList()
          return
        }
        if (this.isFilterSelector) {
          this.getFilterSearchList(params, showSkeletonLoading)
          return
        }

        if (this.throttle) {
          return
        }
        this.throttle = true
        if (!canViewWorkflowItem()) {
          this.skeletonLoading = false
          return
        }

        const { projectId, runItemId } = this.$route.params
        const { bsid } = this.$route.query
        const wkfId = params && params.workflowId ? params.workflowId : this.$route.params.workflowId
        if (showSkeletonLoading && !this.quickFilterCondition) {
          this.tableLoading = true
        }
        if (this.skeletonLoading && !this.quickFilterCondition) {
          this.tableLoading = false
        }
        let propertyList = []
        const _filterFieldIds = Array.isArray(this.filterFields) ? this.filterFields.map((item) => item.fid || item.fieldId) : []
        if (Array.isArray(this.filterFields)) {
          let _obj = {}
          this.filterFields.forEach((item) => {
            _obj[`${item.wkfId || 0}_${item.fid || item.fieldId}`] = item
          })
          this.filterFieldsMap = _obj
        }
        let orderBy = this.orderBy
        if (this.isTreeListMode && !_.get(orderBy, 'field')) {
          orderBy = { field: FieldDocumentHierarchyId, direction: 'DESC' }
        }

        let conditions = typeof this.conditions === 'object' ? _.cloneDeep(this.conditions) : undefined
        if (conditions && typeof this.quickFilterCondition === 'object') {
          const _keysArr = Object.keys(conditions)
          if (_keysArr.length) {
            const _key = `#DrQL${_keysArr.length + 1}`
            conditions[_key] = _.cloneDeep(this.quickFilterCondition)
          } else {
            conditions['#DrQL1'] = _.cloneDeep(this.quickFilterCondition)
          }
        }

        let queryParams = {
          projectId: projectId,
          workflowId: wkfId,
          filterId: this.filterId,
          filter: this.filter,
          wholeWord: this.wholeWord,
          conditions: conditions,
          orderBy: orderBy,
          groupBy: this.hasGroupBy ? JSON.stringify(this.groupBy) : undefined,
          aggregates: this.hasGroupBy ? JSON.stringify(this.aggregates) : undefined,
          fieldIds: _filterFieldIds,
          mode: this.mode,
          nodeId: _.get(this.focusItem, 'id') || undefined,
          ...this.formData,
          bsid: bsid ? +bsid : undefined,
        }

        if (this.isRunCases) {
          if (this.listParentId) {
            queryParams.parentId = this.listParentId
            queryParams.recursively = this.recursively
          } else {
            queryParams.parentId = runItemId
            queryParams.recursively = true
          }
        }

        const cancelToken = genCancelToken((cancel) => {
          this.cancelReqMap.cancelLoadListMode = cancel
        })
        getWorkflowItemsForListMode(queryParams, cancelToken).then(
          (data) => {
            if (this.isDestroy || this.$route.params.workflowId != wkfId) return

            const doItLater = () => {
              this.$emit('hide-list-loading')
              this.skeletonLoading = false
              this.tableLoading = false
              this.useLazyTreeTable = this.isTreeListMode
              this.showParagraph = _.get(this.orderBy, 'field') === FieldDocumentHierarchyId || this.isTreeListMode
              this.throttle = false
              if (data) {
                let _tempMap = {}
                data.config.forEach((item) => {
                  _tempMap[item.id] = item
                })
                this.configMap = _tempMap
                propertyList = data.config
                  .filter((config) => !(Array.isArray(_filterFieldIds) && _filterFieldIds.length) || _filterFieldIds.includes(config.id))
                  .filter((config) => config.id === FieldNameId || !config.isHidden)
                  .map(this.formatColObj)
                if (this.hasGroupBy) {
                  this.tbDynamicColumns = [this.group_name_column, ...propertyList]
                } else if (this.isTreeListMode) {
                  this.tbDynamicColumns = propertyList.sort((a, b) => (a.id === FieldNameId ? -1 : 0))
                } else {
                  this.tbDynamicColumns = propertyList
                }
                this.doAfterTableRender()
                if (data.values && Array.isArray(data.values.list) && data.values.list.length) {
                  if (this.hasGroupBy) {
                    if (data.values.groups) {
                      const _map = genItemsGroupByMap(data.values, this.aggregates)

                      let _groups = []
                      const genGroups = (obj, index = 0, path = []) => {
                        return Object.keys(obj)
                          .map((key) => {
                            let _path = [...path, key]
                            let _item = {
                              ...obj[key],
                              items: undefined,
                              isGroup: true,
                              key: data.values.groups[index],
                              value: key.replace(map_key_prefix, ''),
                              rowKey: _path.join('_'),
                              level: _path.length,
                              _count: Number(_.get(obj[key], '_count', 0)),
                            }
                            const _permsMap = obj[key].perms || {}
                            if (Array.isArray(obj[key].items)) {
                              return {
                                ..._item,
                                _count: obj[key]._count,
                                children: obj[key].items.map((item) => {
                                  const itemId = _.get(item, [FieldIdId, 'value'])
                                  return {
                                    ...item,
                                    editable: _permsMap[itemId] ? true : false,
                                    itemId: itemId,
                                    rowKey: `${_item.rowKey}_item_${itemId}`,
                                    level: _path.length + 1,
                                  }
                                }),
                              }
                            } else {
                              return { ..._item, _count: 0, children: genGroups(obj[key], index + 1, _path) }
                            }
                          })
                          .sort((a, b) => (a.value === 'unset' ? 1 : b.value === 'unset' ? -1 : 0))
                          .filter((group) => Array.isArray(group.children) && group.children.length)
                      }
                      _groups = genGroups(_map)

                      const sumGroupCounts = (groups) => {
                        if (!Array.isArray(groups)) return 0
                        return groups.reduce((sum, group) => {
                          if (Array.isArray(group.children) && group.children.length > 0) {
                            // 递归对子分组求和
                            group._count = sumGroupCounts(group.children)
                            return sum + group._count
                          } else {
                            // 叶子分组，count 已经是最终值
                            return sum + 1
                          }
                        }, 0)
                      }
                      const sumGroupTotal = (groups) => {
                        if (!Array.isArray(groups)) return 0
                        return groups.reduce((sum, group) => {
                          if (group._count) {
                            return sum + group._count
                          } else if (Array.isArray(group.children) && group.children.length > 0) {
                            group._count = sumGroupTotal(group.children)
                            return sum + group._count
                          } else {
                            return sum
                          }
                        }, 0)
                      }
                      sumGroupCounts(_groups)
                      sumGroupTotal(_groups)

                      this.tableData = _groups
                      this.tableExpandKeys = []

                      // setTimeout(() => {
                      //   console.log('this.tableData', _.cloneDeep(this.tbDynamicColumns), _.cloneDeep(this.tableData))
                      // }, 1000)
                    }
                  } else {
                    const _permsMap = data.perms || {}
                    const newList = data.values.list.map((item) => {
                      if (this.isTreeListMode) {
                        item.children = []
                        item.hasChildren = _.get(item, [FieldNameId, 'childCount']) > 0
                      }
                      const itemId = _.get(item, [FieldIdId, 'value'])
                      return { ...item, editable: _permsMap[itemId] ? true : false, itemId: itemId, rowKey: `item_${itemId}` }
                    })

                    if (this.isTreeListMode && !this.reloadTable) {
                      const lazyTreeNodeMap = _.get(this.$refs.taskListTable, 'store.states.lazyTreeNodeMap')
                      if (lazyTreeNodeMap) {
                        newList.forEach((row) => {
                          const loadedChildren = _.get(lazyTreeNodeMap, row.rowKey) // 已加载的子节点，防止刷新子项时去掉了已加载的孙子节点导致不一致
                          if (Array.isArray(loadedChildren) && loadedChildren.length) {
                            row.children = loadedChildren
                          }
                        })
                      }
                    }

                    this.tableData = newList
                  }
                  this.total = data.values.total
                  if (this.reloadTable) {
                    this.dynamicKey++
                    this.reloadTable = false
                  }
                  if (this.focusItem) {
                    if (data.values.pageNo) {
                      this.$set(this.formData, 'pageNo', data.values.pageNo)
                    }
                    this.focusItem = null
                  }
                } else if (this.formData.pageNo > 1) {
                  this.formData.pageNo = this.formData.pageNo - 1
                  this.handleGetWorkflowItems()
                } else {
                  this.tableData = []
                  this.tableExpandKeys = []
                  this.total = 0
                }
                this.$nextTick(() => {
                  const _key = this.isRunCases ? `${wkfId}_${runItemId}` : wkfId
                  setListPage(_key, this.formData)

                  if (_.get(this.currentRow, [FieldIdId, 'value'])) {
                    this.findItem(_.get(this.currentRow, [FieldIdId, 'value']), (row, index, siblings, parent) => {
                      this.$refs.taskListTable && this.$refs.taskListTable.setCurrentRow(row)
                    })
                  }
                  if (this.tableData.length) {
                    this.adjustColumnWidths()
                    setTimeout(() => {
                      if (this.sortable) {
                        this.sortable.destroy()
                      }
                      this.columnDrop()
                      this.highlightCurrentItem()
                      // 调用列宽度自适应方法

                      if (this.tbDynamicColumns.some((col) => col.id === FieldDescriptionId)) {
                        prismjs.highlightAll()
                        this.findImages('.item-desc-content.dr-richtext')
                        this.$store.dispatch('task/findAllDromeLink', { selector: '.item-desc-content.dr-richtext' })
                        this.loadInnerLink('.item-desc-content.dr-richtext')
                      }
                    }, 100)
                  }

                  if (this.isRunCases) {
                    this.getRunCasesVersion()
                  }
                })
              } else {
                this.tableData = []
                this.tableExpandKeys = []
              }

              this.$nextTick(() => {
                this.keepSelectionAfterRefreshList()
              })
            }
            setTimeout(() => {
              doItLater()
            }, 100)
          },
          () => {
            this.$emit('hide-list-loading')
            this.skeletonLoading = false
            this.tableLoading = false
            this.throttle = false
            this.total = 0
            this.formData.pageNo = 1
            this.tableData = []
            this.tableExpandKeys = []
          }
        )
      },

      // 刷新列表后保留之前勾选的项，如果已经不存在的去掉
      keepSelectionAfterRefreshList() {
        // console.log('keep', this.multipleSelection.length)

        if (this.multipleSelection && this.multipleSelection.length) {
          this.selectionChangeTriggered = false
          let selection = []

          if (this.tableData.length) {
            const _existIds = []
            this.loopItem((row) => {
              if (!_existIds.includes(row.itemId)) {
                _existIds.push(row.itemId)
              }
            })

            // console.log('_existIds', _existIds)

            // 去掉已选中但不存在的
            this.multipleSelection = this.multipleSelection.filter((row) => _existIds.includes(row.itemId))
            // 选中之前选中的
            const _selectedIds = this.multipleSelection.map((row) => row.itemId)
            // console.log('_selectedIds', _selectedIds)
            selection = _.get(this.$refs.taskListTable, 'store.states.selection', [])
            if (Array.isArray(selection)) {
              selection = selection.filter((row) => _selectedIds.includes(row.itemId))
              this.$set(this.$refs.taskListTable.store.states, 'selection', selection)
            }

            if (_selectedIds.length) {
              this.forEachItem((row) => {
                if (_selectedIds.includes(row.itemId)) {
                  this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(row, true)
                }
              })
            }

            this.$nextTick(() => {
              this.checkTableAllSelected()
            })
          } else {
            this.clearMultipleSelection()
            selection = []
          }

          setTimeout(() => {
            if (!this.selectionChangeTriggered && !this.useSelection) {
              if (Array.isArray(selection)) {
                this.handleSelectionChange(selection)
              }
            }
          }, 100)
        }
      },

      async getChildrenData(row) {
        // console.log('getChildrenData called', row.itemId, row)
        const { bsid } = this.$route.query
        const id = _.get(row, [FieldIdId, 'value'])

        const cancelToken = genCancelToken((cancel) => {
          this.cancelReqMap.cancelLoadListMode = cancel
        })

        try {
          const res = await getWorkflowListItemChildren({ id, bsid: bsid ? +bsid : undefined }, cancelToken)
          if (res && Array.isArray(res.values)) {
            const _permsMap = res.perms || {}

            const { lazyTreeNodeMap } = this.$refs.taskListTable.store.states

            const _children = res.values.map((item) => {
              if (this.isTreeListMode) {
                item.children = []
                item.hasChildren = _.get(item, [FieldNameId, 'childCount']) > 0
              }
              const itemId = _.get(item, [FieldIdId, 'value'])
              const rowKey = `item_${itemId}`
              const loadedChildren = _.get(lazyTreeNodeMap, rowKey) // 已加载的子节点，防止刷新子项时去掉了已加载的孙子节点导致不一致

              return {
                ...item,
                editable: _permsMap[itemId] ? true : false,
                itemId: itemId,
                rowKey: rowKey,
                parentId: id,
                children: Array.isArray(loadedChildren) && loadedChildren.length ? loadedChildren : [],
              }
            })
            this.$set(row, 'children', _children)
            this.$set(row, 'hasChildren', _children.length > 0)
            return _children
          } else {
            this.$set(row, 'children', [])
            this.$set(row, 'hasChildren', false)
            return []
          }
        } catch (e) {
          this.$set(row, 'children', [])
          this.$set(row, 'hasChildren', false)
          return []
        }
      },

      fixLazyLoadSelection(row, newChildren) {
        const table = this.$refs.taskListTable
        if (!table || !table.store.states.lazyTreeNodeMap) return

        const oldChildren = table.store.states.lazyTreeNodeMap[row.rowKey]
        if (Array.isArray(oldChildren) && oldChildren.length > 0) {
          const selectedOldChildren = oldChildren.filter((c) => table.selection.includes(c))

          if (selectedOldChildren.length > 0) {
            selectedOldChildren.forEach((oldChild) => {
              table.toggleRowSelection(oldChild, false)

              const newChild = newChildren.find((c) => c.itemId === oldChild.itemId)
              if (newChild) {
                table.toggleRowSelection(newChild, true)
              }
            })
          }
        }
      },

      loadChildren(row, treeNode, resolve) {
        // console.log('loadChildren', row, treeNode)
        const id = _.get(row, [FieldIdId, 'value'])
        this.lazyTreeNodeMap.set(id, { row, treeNode, resolve })
        this.$set(row, 'hasChildren', true)

        // console.log('if 1', this.loadingPromises && this.loadingPromises[id])
        if (this.loadingPromises && this.loadingPromises[id]) {
          this.loadingPromises[id].then((children) => {
            // console.log('load 1', children)
            this.fixLazyLoadSelection(row, children)
            resolve(children)
            if (children.length === 0) {
              this.$set(this.$refs.taskListTable.store.states.lazyTreeNodeMap, row.rowKey, [])
            }

            this.$nextTick(() => {
              this.checkTableAllSelected()
            })
          })
          return this.loadingPromises[id]
        }

        const promise = this.getChildrenData(row).then((children) => {
          // console.log('resolve children', children)
          // console.log('load 2', children)
          this.fixLazyLoadSelection(row, children)
          resolve(children)
          if (children.length === 0) {
            this.$set(this.$refs.taskListTable.store.states.lazyTreeNodeMap, row.rowKey, [])
          }

          this.$nextTick(() => {
            this.checkTableAllSelected()
          })
          return children
        })

        if (!this.loadingPromises) this.loadingPromises = {}
        this.loadingPromises[id] = promise
        return promise
      },

      async loadChildrenAndExecute(idOrRow, callback, force = false) {
        let row = idOrRow
        if (typeof idOrRow !== 'object') {
          this.findItem(idOrRow, (r) => {
            row = r
          })
        }

        if (!row || !row.hasChildren) return []

        let children = row.children
        if (force || !children || children.length === 0) {
          const id = _.get(row, [FieldIdId, 'value'])

          // Wait a tick for toggleRowExpansion to possibly trigger loadChildren
          await this.$nextTick()

          let promise = this.loadingPromises && this.loadingPromises[id]
          if (!promise) {
            // Manually trigger load if not triggered by table
            promise = this.getChildrenData(row).then((data) => {
              // console.log('load 3')
              this.fixLazyLoadSelection(row, data)

              // Manually update lazyTreeNodeMap to let table know data is loaded
              if (data.length > 0) {
                this.$set(this.$refs.taskListTable.store.states.lazyTreeNodeMap, row.rowKey, data)
              } else {
                this.$set(this.$refs.taskListTable.store.states.lazyTreeNodeMap, row.rowKey, [])
              }

              // Fix: Manually update treeData state to ensure correct expand/collapse behavior
              // This is necessary because we bypassed the standard load process
              const treeData = this.$refs.taskListTable.store.states.treeData
              if (treeData[row.rowKey]) {
                this.$set(treeData[row.rowKey], 'loaded', true)
                this.$set(treeData[row.rowKey], 'expanded', true)
              }

              this.$nextTick(() => {
                this.checkTableAllSelected()
              })

              return data
            })

            if (!this.loadingPromises) this.loadingPromises = {}
            this.loadingPromises[id] = promise
          }

          children = await promise
        }

        if (callback && typeof callback === 'function') {
          await callback(children, row)
        }
        return children
      },

      async handleExpandAllChildren(row) {
        if (!row) return
        if (!row.hasChildren) return

        // Expand current row
        this.$refs.taskListTable.toggleRowExpansion(row, true)

        await this.loadChildrenAndExecute(row, async (children) => {
          // Recursively expand children
          if (children && children.length > 0) {
            for (const child of children) {
              await this.handleExpandAllChildren(child)
            }
          }
        })
      },

      logTableStore() {
        console.log('log store', this.$refs.taskListTable.store.states)
        const { selection } = this.$refs.taskListTable.store.states
        if (Array.isArray(selection)) {
          let ids = selection.map((row) => row.itemId)
          console.log('ids', ids.length, [...ids])
          ids.forEach((id, index) => {
            ids.forEach((id2, index2) => {
              if (id === id2 && index !== index2) {
                console.log('id 重复了', id)
              }
            })
          })
        }

        this.checkTableAllSelected()
      },

      async handleCollapseAllChildren(row) {
        if (!row) return
        if (!row.hasChildren) return

        this.$refs.taskListTable.toggleRowExpansion(row, false)

        let children = row.children
        if (children && children.length > 0) {
          for (const child of children) {
            await this.handleCollapseAllChildren(child)
          }
        }
      },

      getRunCasesVersion() {
        if (this.IsBaselineView) {
          this.testRunVersionData = null
          return
        }

        const ids = this.tableData.map((row) => row.itemId)
        if (ids.length) {
          getTestRunVersion({ ids: ids }).then((res) => {
            if (res) {
              this.testRunVersionData = res
              this.$emit('set-test-run-version-data', _.cloneDeep(res))
            } else {
              this.testRunVersionData = null
              this.$emit('set-test-run-version-data', null)
            }
          })
        }
      },

      formatColObj(config) {
        if (!this.isFilterSelector && typeof config.wkfId === 'undefined') {
          const { workflowId } = this.$route.params
          config.wkfId = +workflowId
        }
        const newWidth = typeof config.wkfId === 'number' ? _.get(this.filterFieldsMap, `${config.wkfId || 0}_${config.id}.width`) : undefined // 视图中保存的宽度
        const backgroundColor = typeof config.wkfId === 'number' ? _.get(this.filterFieldsMap, `${config.wkfId || 0}_${config.id}.backgroundColor`) : undefined // 视图中保存的背景色

        let width = config.colspan
        if (newWidth) {
          width = newWidth
        } else {
          const selectWidth = this.getSelectWidth(config)
          width = selectWidth ? selectWidth : width
          if (!width) {
            width = fieldDefaultConfig[config.fieldType].minWidth // 将原来的 minWidth 转为 width
            if (config.id === FieldNameId) {
              width = 300 // 将原来的 minWidth 转为固定的 width
            } else if (config.id === FieldProgressId) {
              width = 120
            } else if (config.id === FieldDescriptionId) {
              width = 300
            } else if (config.id === FieldSpentEstimatedHourId) {
              width = 80
            } else if (config.id === FieldConfigItemId) {
              width = 100
            } else if (config.id === FieldDeliverableId) {
              width = 100
            }
          }
        }

        let label = config.title || config.name
        let align = config.fieldType === FieldClassChoice || config.fieldType === FieldClassInt ? 'center' : ''
        if (config.id === FieldIdId || config.id === FieldSN) {
          align = 'left'
        }

        if (_.get(config, 'options', []).some((item) => !!item.path)) {
          width = 160
          align = 'left'
        }

        return {
          fieldId: config.id,
          label: label,
          width,
          align,
          ...config,
          backgroundColor,
          color: backgroundColor ? getContrastTextColor(backgroundColor) : undefined,
          className: this.isTreeListMode && config.id === FieldNameId ? 'table-column--static tree-node-name' : undefined,
        }
      },

      getClientHeight() {
        let clientHeight = 0
        if (document.body.clientHeight && document.documentElement.clientHeight) {
          clientHeight = document.body.clientHeight < document.documentElement.clientHeight ? document.body.clientHeight : document.documentElement.clientHeight
        } else {
          clientHeight = document.body.clientHeight > document.documentElement.clientHeight ? document.body.clientHeight : document.documentElement.clientHeight
        }
        return clientHeight
      },
      getClientWidth() {
        let clientWidth = 0
        if (document.body.clientWidth && document.documentElement.clientWidth) {
          clientWidth = document.body.clientWidth < document.documentElement.clientWidth ? document.body.clientWidth : document.documentElement.clientWidth
        } else {
          clientWidth = document.body.clientWidth > document.documentElement.clientWidth ? document.body.clientWidth : document.documentElement.clientWidth
        }
        return clientWidth
      },

      handleCurrentChange(current) {
        if (current) {
          this.currentRow = current
        }
      },
      rightClickRow(row, column, event) {
        if (!this.showContextmenu || !this.$refs.contextmenu) return

        if (_.get(row, 'isGroup') || !_.get(row, 'itemId')) {
          this.hideContextMenu()
          return
        }

        const lockedItem = this.$store.getters['task/lockedItem']
        if (lockedItem && lockedItem.id) {
          const _taskListEle = document.querySelector('.task-list')
          if (_taskListEle && _taskListEle.click) {
            _taskListEle.click()
          }
        }

        event.preventDefault()
        event.stopPropagation()

        this.currentRow = row
        this.$refs.taskListTable && this.$refs.taskListTable.setCurrentRow(row)

        setTimeout(() => {
          let menuCount = 0
          this.fileMenus.forEach((group) => {
            Array.isArray(group) &&
              group.forEach((item) => {
                menuCount++
              })
          })
          const menuWidth = 150
          const menuHeight = 35 * menuCount + 20
          const clientHeight = this.getClientHeight()
          const clientWidth = this.getClientWidth()
          const defaultX = event.x + 5
          const defaultY = event.y + 5
          let top = clientHeight - defaultY > menuHeight ? defaultY : clientHeight - menuHeight
          let left = clientWidth - defaultX > menuWidth ? defaultX : defaultX - menuWidth - 10
          if (menuCount) {
            this.$refs.contextmenu.show({ top, left })
          }
        }, 100)
      },
      hideContextMenu() {
        this.$refs.contextmenu && this.$refs.contextmenu.hide()
      },
      handleGlobalContextmenu(event) {
        if (!this.showContextmenu) return
        this.hideContextMenu()
      },

      handlePageChange(val) {
        if (!this.useSelection) {
          this.clearMultipleSelection()
        }
        this.formData.pageNo = val
        this.handleGetWorkflowItems()
      },
      /* 每页显示条数变更 */
      handleSizeChange(val) {
        if (!this.useSelection) {
          this.clearMultipleSelection()
        }
        this.formData.pageSize = val
        this.formData.pageNo = 1
        this.handleGetWorkflowItems()
      },

      resetPageNo() {
        this.formData.pageNo = 1
      },

      rowSelectable(row) {
        // if (this.isConfigWorkflow) {
        //   if (Array.isArray(this.multipleSelection) && this.multipleSelection.length) {
        //     if (this.multipleSelection.length > 1 && !this.multipleSelection.some((item) => _.get(item, [FieldIdId, 'value']) === _.get(row, [FieldIdId, 'value']))) {
        //       return false
        //     }
        //     if (this.multipleSelection.some((item) => _.get(item, [FieldIdId, 'value']) === _.get(row, [FieldIdId, 'value']))) {
        //       return true
        //     }
        //   }
        // }

        return !row._noPermissionOnItem && !row.isGroup
      },

      handleTableRowClick(row) {
        if (this.useSelection) {
          this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(row)
        }
      },

      handleTableSelect() {},
      handleTableSelectAll(selection) {
        // console.log('select all', this.skipSelectAllFunc, selection)
        if (this.skipSelectAllFunc) return

        this.selectionChangeTriggered = false

        if (this.hasGroupBy) {
          const _total = this.tableData.reduce((sum, group) => sum + Number(group._count || 0), 0)

          if (_total === this.multipleSelection.length) {
            this.$refs.taskListTable && this.$refs.taskListTable.clearSelection()
          } else {
            this.forEachItem((row) => {
              if (!row.isGroup) {
                this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(row, true)
              }
            })

            setTimeout(() => {
              this.$set(this.$refs.taskListTable.store.states, 'isAllSelected', true)
            }, 100)
          }
        } else if (this.isTreeListMode) {
          // 递归统计所有行（包括 children）
          function countRows(data) {
            let count = 0
            if (Array.isArray(data)) {
              data.forEach((item) => {
                count += 1
                if (Array.isArray(item.children) && item.children.length > 0) {
                  count += countRows(item.children)
                }
              })
            }
            return count
          }
          const _total = countRows(this.tableData, this.multipleSelection.length)

          if (_total === this.multipleSelection.length) {
            this.clearMultipleSelection()
          } else {
            this.forEachItem((row) => {
              this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(row, true)
            })

            setTimeout(() => {
              this.$set(this.$refs.taskListTable.store.states, 'isAllSelected', true)
            }, 100)
          }
        }

        setTimeout(() => {
          if (!this.selectionChangeTriggered) {
            this.handleSelectionChange(selection)
          }
        }, 100)
      },

      handleSelectionChange: _.debounce(function (selection) {
        // console.log('selection change', selection)
        this.selectionChangeTriggered = true

        if (this.useLazyTreeTable) {
          this.multipleSelection = selection
          this.$emit('selection-change', selection)
          this.skipSelectAllFunc = true
          this.$nextTick(() => {
            this.checkTableAllSelected()
          })
          setTimeout(() => {
            this.skipSelectAllFunc = false
          }, 100)
        } else {
          this.multipleSelection = selection
          this.$emit('selection-change', selection)
        }
      }, 100),

      toggleRowSelectionById(itemId, selected) {
        this.findItem(itemId, (row, index, siblings, parent) => {
          this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(row, selected)
        })
      },

      checkTableSelectionByIds() {
        if (Array.isArray(this.selectedIds) && this.selectedIds.length) {
          this.loopItem((item) => {
            if (this.selectedIds.includes(item.id)) {
              this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(item, true)
            } else {
              this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(item, false)
            }
          })
        }
      },

      checkTableAllSelected() {
        // console.log('check select all')
        setTimeout(() => {
          if (this.$refs.taskListTable) {
            const selection = this.$refs.taskListTable.store.states.selection

            // Manually update isAllSelected
            const countRows = (data) => {
              let count = 0
              if (Array.isArray(data)) {
                data.forEach((item) => {
                  count += 1
                  if (Array.isArray(item.children) && item.children.length > 0) {
                    count += countRows(item.children)
                  }
                })
              }
              return count
            }
            const _total = countRows(this.tableData)

            // console.log(_total, selection.length, _.cloneDeep(this.tableData))
            this.$set(this.$refs.taskListTable.store.states, 'isAllSelected', _total === selection.length)
          }
        }, 100)
      },

      onStatusChange(status) {
        this.handleGetWorkflowItems()
      },
      //* 识别组件
      getComponent(col, row) {
        const config = _.get(row, [col.fieldId, 'config']) ? { ...col, ...row[col.fieldId].config } : col
        const preset = ['Duration', 'Boolean', 'ReferenceData', 'Reference', 'Member', 'Link']
        const _flags = _.get(row, [FieldFlagsId, 'value'], 0)
        // 预设组件
        if (this.isFilterSelector && ((col.wkfId && _.get(row, 'workflow.id') !== col.wkfId) || !row[col.fieldId])) {
          return 'TDEmptyField'
        } else if (col.fieldId === FieldNameId) {
          return 'TdName'
        } else if (col.fieldId === FieldIdId) {
          return 'TdItemId'
        } else if (col.fieldId === FieldSN) {
          return 'TdItemSN'
        } else if (col.fieldId === FieldStatusId) {
          return 'StatusIconDropdown'
        } else if (col.fieldId === FieldConfigItemId) {
          if (!checkBit(_flags, TASK_FLAGS.FOLDER)) {
            return 'ConfigItemField'
          }
        } else if (col.fieldId === FieldDeliverableId) {
          return 'DeliverableField'
        } else if (config.id === FieldProgressId) {
          return 'ProProgress'
        } else if (col.fieldId === FieldSpentHourId && this.workflowInfo.issueTypeId !== WORKFLOW_WORK_LOG && this.workflowInfo.issueTypeId !== WORKFLOW_TEST_RUN) {
          return 'SpentHourField'
        } else if (this.isPlainTextField(config.fieldType)) {
          return 'PlainTextField'
        } else if (preset.includes(propertyTypes[config.fieldType])) {
          return 'Pro' + propertyTypes[config.fieldType]
        } else {
          // 默认 文本组件
          // return 'ProNormal'
        }
      },
      //* 生成组件绑定的数据（为了兼容之前的组件做了些数据格式的处理）
      getBindData(col, row, index) {
        const { bsid } = this.$route.query

        const createdBy = _.get(row, [FieldCreatedById, 'value', 0])
        const options = this.isFilterSelector ? _.get(row, [FieldStatusId, 'config', 'options'], []) : _.get(row, [FieldStatusId, 'config', 'options'], [])
        const status = options.find((item) => item.id === _.get(row, [FieldStatusId, 'value']))

        const parentId = _.get(row, [FieldParentId, 'value', 0, 'item', 'id'])
        const isTestRunChild = this.workflowInfo.issueTypeId === WORKFLOW_TEST_RUN && (parentId || this.$route.name === 'Project_TestRunList')

        const _itemId = this.isFilterSelector ? _.get(row, 'id') : _.get(row, [FieldIdId, 'value'])
        const _flags = this.isFilterSelector ? _.get(row, 'flags', 0) : _.get(row, [FieldFlagsId, 'value'], 0)
        const _workflow = this.isFilterSelector ? _.get(row, 'workflow', {}) : this.workflowInfo

        const _itemResolvedOrClosed = isItemResolved(_flags) || isItemClosed(_flags)

        if (col.fieldId === FieldIdId) {
          return {
            current: {
              id: _itemId,
              name: _.get(row, [FieldNameId, 'value']),
              status,
              flags: _flags,
              parentId,
            },
            workflow: _workflow,
          }
        } else if (col.fieldId === FieldSN) {
          const data = {
            projectId: +this.$route.params.projectId,
            workflowId: +_workflow.id,
            itemId: +_itemId,
            name: _.get(row, [FieldNameId, 'value']),
            workflow: _workflow,
            workflowPermission: _workflow.id == this.$route.params.workflowId ? _.cloneDeep(this.workflowPermission) : undefined,
          }
          return {
            current: {
              sn: _.get(row, [FieldSN, 'value']),
              id: _itemId,
              name: _.get(row, [FieldNameId, 'value']),
              status,
              flags: _flags,
              parentId,
            },
            workflow: _workflow,
            textOverflow: false,
            tag: _.get(row, [FieldNameId, 'tag']),
            linkTo: isTestRunChild
              ? () => {
                  this.$baseEventBus.$emit('show-test-run-dialog', data)
                }
              : this.User_Def_WorkflowListDetailMode === 'jump-to-detail'
              ? undefined
              : () => {
                  this.$baseEventBus.$emit('show-task-detail', data)
                },
            bsid: bsid ? +bsid : undefined,
          }
        } else if (col.fieldId === FieldNameId) {
          const data = {
            projectId: _.get(_workflow, 'project.id') || +this.$route.params.projectId,
            workflowId: +_workflow.id,
            itemId: +_itemId,
            name: _.get(row, [FieldNameId, 'value']),
            workflow: _workflow,
            workflowPermission: _workflow.id == this.$route.params.workflowId ? _.cloneDeep(this.workflowPermission) : undefined,
          }
          let linkTo

          if (this.isFTAWorkflow) {
            linkTo = () => {
              this.$refs.FTADrawerDialog.open(data)
            }
          } else if (isTestRunChild) {
            linkTo = () => {
              this.$baseEventBus.$emit('show-test-run-dialog', data)
            }
          } else if (this.User_Def_WorkflowListDetailMode === 'jump-to-detail') {
          } else {
            linkTo = () => {
              // this.$baseEventBus.$emit('show-task-detail', data)
              this.$baseEventBus.$emit('open-work-item-dialog', data)
            }
          }

          return {
            current: {
              id: _itemId,
              name: _.get(row, [FieldNameId, 'value']),
              status,
              description: _.get(row, [FieldDescriptionId, 'value']),
              flags: _flags,
              parentId,
              newVersion: this.isRunCases ? _.get(this.testRunVersionData, `${_itemId}.newVersion`) : undefined,
              newVersionDisabled: this.isRunCases ? this.pageReadOnly || !this.testRunEditable || _itemResolvedOrClosed : undefined,
            },
            workflow: _workflow,
            paragraph: this.showParagraph ? _.get(row, [FieldNameId, 'paragraph']) : undefined,
            level: this.showParagraph && !this.useLazyTreeTable ? _.get(row, [FieldNameId, 'level'], 0) : undefined,
            showItemIcon: this.showParagraph && !this.isTestRunWorkflow, // 测试执行列表不显示图标
            hideStatus: true,
            showId: false,
            textOverflow: false,
            showFolderOrInfoIcon: true,
            tag: _.get(row, [FieldNameId, 'tag']),
            isTaskList: !this.isFilterSelector,
            linkTarget: this.isFilterSelector ? '_blank' : undefined,
            linkTo,
            bsid: bsid ? +bsid : undefined,
          }
        } else if (col.fieldId === FieldStatusId) {
          return {
            layout: 'label',
            data: {
              id: _itemId,
              name: _.get(row, [FieldNameId, 'value']),
              status,
              flags: _flags,
              createdBy,
              parent: _.get(row, [FieldParentId, 'value', 0, 'item']),
              parentId: _.get(row, 'parentId'),
            },
            workflowInfo: _workflow,
            readonly: this.pageReadOnly,
            showUpdate: this.isRunCases ? false : true,
            showDelete: true,
            showProperty: true,
            showConfigItemMenus: this.isConfigWorkflow,
          }
        } else {
          const config = _.get(row, [col.fieldId, 'config']) ? { ...this.formatColObj(row[col.fieldId].config) } : col
          const ignoreRequired = checkBit(_flags, TASK_FLAGS.FOLDER) || checkBit(_flags, TASK_FLAGS.INFORMATION)
          return {
            ref: `xFieldEditor_${col.id}`,
            ...config,
            value: _.get(row, [col.fieldId, 'value']),
            isRequired: ignoreRequired ? false : config.isRequired,
            editable: _.get(row, [col.fieldId, 'editable']) && !this.pageReadOnly,
            itemId: _itemId,
            index: index,
            // layout: 'mini',
            workflow: _workflow,
            fieldLabel: this.translateTitle(col.label),
            selectType: this.getSelectType(col),
            associationEditable: true,
            statusId: _.get(status, 'id'),
          }
        }
      },

      getSelectType(field) {
        const _width = field.width || field.colspan
        if (field.fieldType === FieldClassChoice && !field.isMultiSelection) {
          if (this.isFilterSelector) {
            // 筛选器列表中不显示选项图标，避免多流程时部分字段有选项图标导致整列的值对不齐
            return 'word'
          }
          if (_width) {
            if (_width <= 40) {
              return 'icon'
            } else if (_width <= 75) {
              return 'word'
            } else {
              // 宽度大于 75px 显示图标 + 文字 刚好显示三个文字
              return 'all'
            }
          } else {
            return 'word'
          }
        }
        return undefined
      },
      getSelectWidth(field) {
        const _keyName = _.get(this.workflowInfo, 'keyName')
        if (field.id === FieldIdId || field.id === FieldSN) {
          if (!field.colspan) {
            if (_keyName) {
              const text = `${_keyName}-999999`
              const canvas = document.createElement('canvas')
              const context = canvas.getContext('2d')
              context.font = '12px -apple-system, BlinkMacSystemFont, Segoe UI, Roboto, Oxygen, Ubuntu, Fira Sans, Droid Sans, Helvetica Neue, sans-serif'
              const textWidth = context.measureText(text).width
              return textWidth + 10
            }
            return 120 // 工作项 ID 默认宽度
          } else {
            return field.colspan
          }
        } else if (field.fieldType === FieldClassChoice && !field.isMultiSelection) {
          if (!field.colspan) {
            // 宽度 64px 刚好显示4个文字
            return this.$store.getters['settings/langIsEnglish'] ? 90 : 76
          } else {
            return field.colspan
          }
        }
        return undefined
      },
      // 获取组件事件
      getEvent(col, row) {
        const createdBy = _.get(row, [FieldCreatedById, 'value', 0])
        const options = this.isFilterSelector ? _.get(row, [FieldStatusId, 'config', 'options'], []) : _.get(row, [FieldStatusId, 'config', 'options'], [])
        const status = options.find((item) => item.id === _.get(row, [FieldStatusId, 'value']))

        const _itemId = this.isFilterSelector ? _.get(row, 'id') : _.get(row, [FieldIdId, 'value'])
        const _flags = this.isFilterSelector ? _.get(row, 'flags', 0) : _.get(row, [FieldFlagsId, 'value'], 0)
        const _workflow = this.isFilterSelector ? _.get(row, 'workflow', {}) : this.workflowInfo

        const currentItem = {
          id: _itemId,
          name: _.get(row, [FieldNameId, 'value']),
          status,
          flags: _flags,
          createdBy,
          parent: _.get(row, [FieldParentId, 'value', 0, 'item']),
          workflow: _workflow,
        }

        if (col.fieldId === FieldNameId) {
          return { 'update-run-case': () => this.$emit('update-run-case', currentItem) }
        } else if (col.fieldId === FieldStatusId) {
          return { 'after-delete': (data) => this.handleAfterDelete(data) }
        } else {
          return { 'on-save': (value, options, errorCB) => this.submitEditItem(value, options, errorCB, currentItem) }
        }
      },

      handleAfterDelete(data) {
        this.$emit('on-item-deleted', data)

        if (this.useLazyTreeTable) {
          if (data && data.id) {
            this.deleteItemInLazyTree(data.id)
          }
        } else {
          if (this.isRunCases) {
            this.$emit('refresh-run-cases-tree', data)
          }

          this.handleGetWorkflowItems()
        }
      },

      // 保存修改
      async submitEditItem(value, { fieldId, itemId, index, fieldType, referenceType }, errorCB, currentItem) {
        const valueToSave = formatFieldValue(value, fieldType, referenceType)

        const successFunc = (res) => {
          this.findItem(itemId, (row, index, siblings, parent) => {
            if (row[fieldId]) {
              this.$set(row[fieldId], 'value', value)
            }
          })
          this.unlockAfterSubmit(itemId)
          if (res) {
            this.refreshItemById(res.id)
            this.$emit('on-item-updated', { itemId, fieldId })
          }
        }
        const resetFunc = () => {
          this.findItem(itemId, (row, index, siblings, parent) => {
            if (row[fieldId]) {
              const _temp = _.cloneDeep(row[fieldId].value)
              this.$set(row[fieldId], 'value', _temp)
            }
          })

          this.unlockAfterSubmit(itemId)
          errorCB && errorCB()
        }
        const setCellLoading = (loading) => {
          this.findItem(itemId, (row, index, siblings, parent) => {
            if (row[fieldId]) {
              this.$set(row[fieldId], 'loading', loading)
            }
          })
        }

        try {
          setCellLoading(true)
          const res = await updateWorkflowItemField({ workflowId: _.get(currentItem, 'workflow.id'), itemId, fieldId, value: valueToSave })
          setCellLoading(false)
          successFunc(res)
        } catch (error) {
          setCellLoading(false)
          if (error && error.code == Code_Required_Fields_Empty && Array.isArray(error.data) && error.data.length) {
            this.$refs.UpdateRequiredFieldsDialog.open(
              currentItem,
              [{ id: fieldId, value: valueToSave }],
              error.data,
              (res) => {
                successFunc(res)
              },
              resetFunc
            )
          } else {
            resetFunc()
          }
        }
      },
      refreshItemById(id) {
        const { bsid } = this.$route.query
        return getWorkflowItemForListMode({ id, bsid: bsid ? +bsid : undefined }).then((res) => {
          if (res && typeof res.value === 'object') {
            if (this.isFilterSelector) {
              this.findItem(id, (row, index, siblings, parent) => {
                this.$set(row, 'editable', res.editable ? true : false)
                Object.keys(res.value).forEach((key) => {
                  this.$set(row, key, res.value[key])
                  if (this.isFilterSelector && key == FieldFlagsId && (_.get(res.value[key], 'value') || _.get(res.value[key], 'value') === 0)) {
                    this.$set(row, 'flags', _.get(res.value[key], 'value'))
                  }
                })
              })
            } else {
              this.findItem(id, (row, index, siblings, parent) => {
                this.$set(row, 'editable', res.editable ? true : false)
                if (typeof res.value === 'object') {
                  Object.keys(res.value).forEach((key) => {
                    this.$set(row, key, res.value[key])
                  })
                }
                const isSelected = this.multipleSelection.some((item) => _.get(item, [FieldIdId, 'value']) === id)
                if (isSelected) {
                  this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(row, true)
                }
              })
            }
            this.$emit('on-list-item-refresh', { id, res })
          } else {
            this.setListItemNoPermission(id)
          }
        })
      },
      // 检查是否锁定并锁定
      // checkIsLockBefore(id, cb = () => {}) {
      //   const lockedItem = this.$store.getters['task/lockedItem']
      //   if (lockedItem && lockedItem.id == id) return
      //   isItemLocked({ id }).then((data) => {
      //     if (data.isLocked) {
      //       this.$message.error(this.translateTitle('task.TaskList.jieDianYiBeiD', { locker: data.lockedBy.name }))
      //     } else {
      //       lockItem({ id }).then((data) => {
      //         if (data.isLocked) {
      //           this.$store.commit('task/setLockedItem', { id })
      //           cb()
      //         } else {
      //           this.$message.error(this.translateTitle('common.lockItemFailed'))
      //         }
      //       })
      //     }
      //   })
      // },
      // 解锁
      unlockAfterSubmit(id, cb = () => {}) {
        unlockItem({ id }).then((data) => {
          if (data.isUnlocked) {
            this.$store.commit('task/clearLockedItem', { id })
            cb()
          } else {
            this.$message.error(this.translateTitle('common.unlockFailed'))
          }
          this.$baseEventBus.$emit('remove-listener-beforeunload')
        })
      },
      clearMultipleSelection() {
        if (this.useLazyTreeTable) {
          this.forEachItem((row) => {
            this.$refs.taskListTable && this.$refs.taskListTable.toggleRowSelection(row, false)
          })

          this.multipleSelection = []
          this.$emit('selection-change', [])
          this.$refs.taskListTable && this.$refs.taskListTable.clearSelection()
        } else {
          this.multipleSelection = []
          this.$emit('selection-change', [])
          this.$refs.taskListTable && this.$refs.taskListTable.clearSelection()
        }
      },
      remove() {
        // document.getElementById('Drome-ALM-Pro').removeEventListener('click', this.blurAll)
      },

      onHeaderDragend(newWidth, oldWidth, column) {
        if (!this.allowDragHeader) return

        const _newWidth = parseInt(newWidth)
        this.tbDynamicColumns.forEach((col) => {
          if (`${col.id}_${col.wkfId || 0}` == column.columnKey) {
            this.$set(col, 'width', _newWidth)
            this.$set(col, 'newWidth', _newWidth)
          }
        })
        this.$nextTick(() => {
          let fields = this.getTableHeaderFields()
          this.$emit('update-filter-fields', fields)
          setTimeout(() => {
            if (this.sortable) {
              this.sortable.destroy()
            }
            this.columnDrop()
          }, 100)
        })
        // if (this.columnsConfig[column.columnKey]) {
        //   this.$set(this.columnsConfig[column.columnKey], 'width', newWidth)
        // } else {
        //   this.$set(this.columnsConfig, column.columnKey, { width: newWidth })
        // }
      },

      handleAutoAdjustWidth(col) {
        if (col.ignoreAllEvents) return

        const _text = col.title || col.label

        // 计算文字宽度
        const canvas = document.createElement('canvas')
        const context = canvas.getContext('2d')
        context.font = '14px -apple-system, BlinkMacSystemFont, Segoe UI, Roboto, Oxygen, Ubuntu, Fira Sans, Droid Sans, Helvetica Neue, sans-serif'
        const textWidth = context.measureText(_text).width

        const columnKey = `${col.id}_${col.wkfId || 0}`
        const newWidth = textWidth + 18 // 18 是 th cell padding + border
        if (newWidth < col.width) return // 如果算出来比当前宽度窄，就不处理

        this.onHeaderDragend(newWidth, col.width, { ...col, columnKey })
      },

      handleAddColumn(col) {
        this.$refs[`AddColumnsPop_${col.id}_${col.wkfId || 0}_${col.colIndex}`][0].doShow()
      },
      onColumnsSelect(selectedColumn, col, colIndex) {
        if (this.tbDynamicColumns[colIndex]) {
          let config
          if (this.isFilterSelector) {
            config = selectedColumn
          } else {
            config = _.get(this.tableData, [0, selectedColumn.id, 'config']) ? { ...selectedColumn, ...this.tableData[0][selectedColumn.id].config } : selectedColumn
          }
          const newConfig = this.formatColObj(_.cloneDeep(config))
          this.tbDynamicColumns.splice(colIndex + 1, 0, newConfig)
        }
        this.emitFilterFields(this.isFilterSelector)
        this.$refs[`AddColumnsPop_${col.id}_${col.wkfId || 0}_${colIndex}`][0].doClose()
      },
      handleRemoveColumn(col) {
        if (this.tbDynamicColumns[col.colIndex]) {
          this.tbDynamicColumns.splice(col.colIndex, 1)
        }
        this.emitFilterFields()
      },
      handleMoveColumnToLeft(col) {
        if (this.tbDynamicColumns[col.colIndex]) {
          const _pick = this.tbDynamicColumns.splice(col.colIndex, 1)[0]
          this.tbDynamicColumns.splice(col.colIndex - 1, 0, _pick)
        }
        this.emitFilterFields()
      },
      handleMoveColumnToRight(col) {
        if (this.tbDynamicColumns[col.colIndex]) {
          const _pick = this.tbDynamicColumns.splice(col.colIndex, 1)[0]
          this.tbDynamicColumns.splice(col.colIndex + 1, 0, _pick)
        }
        this.emitFilterFields()
      },
      handleSortColumnASC(col) {
        if (this.isFilterSelector) {
          this.$emit('order-change', col)
        } else {
          this.$baseEventBus.$emit('sort-by-field', col.id)
        }
      },
      handleSortColumnDESC(col) {
        if (this.isFilterSelector) {
          this.$emit('order-change', col)
        } else {
          this.$baseEventBus.$emit('sort-by-field', col.id)
        }
      },
      handleSetAsCommonField(col) {
        this.$emit('set-as-common-field', col)
      },

      handleQuickFilter(data) {
        this.$emit('on-quick-filter', data)
      },

      // 可疑链接弹窗
      afterMergeSuspected(data) {
        this.refreshItemById(data.currentItemId)
      },
      afterClearSuspected(data) {
        this.refreshItemById(data.currentItemId)
      },

      checkLockItemAndUnlock() {
        const lockedItem = this.$store.getters['task/lockedItem']
        if (lockedItem && lockedItem.id) {
          this.unlockAfterSubmit(lockedItem.id)
        }
      },

      columnDrop() {
        if (!this.allowDragHeader) return

        const wrapperTr = document.querySelector('.task-list-table .el-table__header-wrapper tr')
        if (isHTMLElement(wrapperTr)) {
          import('sortablejs').then(({ default: Sortable }) => {
            this.sortable = Sortable.create(wrapperTr, {
              animation: 180,
              delay: 0,
              filter: '.el-table-column--selection, .table-column--group-title, .table-column--static',
              preventOnFilter: false, // Call `event.preventDefault()` when triggered `filter`
              onMove: (evt) => {
                if (
                  _.get(evt, 'related._prevClass').includes('el-table-column--selection') ||
                  _.get(evt, 'related._prevClass').includes('table-column--group-title') ||
                  _.get(evt, 'related._prevClass').includes('table-column--static')
                ) {
                  return false
                } else {
                  return true
                }
              },
              onEnd: (evt) => {
                const _oldIndex = this.showTableSelection ? evt.oldIndex - 1 : evt.oldIndex
                const _newIndex = this.showTableSelection ? evt.newIndex - 1 : evt.newIndex
                if (_oldIndex >= 0 && _oldIndex < this.tbDynamicColumns.length && _newIndex >= 0 && _newIndex < this.tbDynamicColumns.length && _oldIndex !== _newIndex) {
                  const oldItem = this.tbDynamicColumns[_oldIndex]
                  if (oldItem) {
                    this.tbDynamicColumns.splice(_oldIndex, 1)
                    this.tbDynamicColumns.splice(_newIndex, 0, oldItem)
                    this.emitFilterFields(false, false)
                  }
                }
              },
            })
          })
        }
      },
      emitFilterFields(refreshList, renderTable) {
        if (renderTable) {
          // this.dynamicKey++
        } else {
          // this.$nextTick(() => {
          //   this.$refs.taskListTable && this.$refs.taskListTable.doLayout()
          // })
        }
        this.doAfterTableRender()
        this.$nextTick(() => {
          let fields = this.getTableHeaderFields()
          this.$emit('update-filter-fields', fields, refreshList)
          setTimeout(() => {
            if (this.sortable) {
              this.sortable.destroy()
            }
            this.columnDrop()
          }, 100)
        })
      },
      getTableHeaderFields() {
        if (this.isFilterSelector) {
          return this.tbDynamicColumns
            .filter((item) => item.id > 0)
            .map((item) => ({ fid: item.id, width: item.newWidth || item.width, wkfId: item.wkfId, backgroundColor: item.backgroundColor }))
        } else {
          return this.tbDynamicColumns
            .filter((item) => item.id > 0)
            .map((item) => ({ fieldId: item.id, width: item.newWidth || item.width, wkfId: item.wkfId, backgroundColor: item.backgroundColor }))
        }
      },

      onWSUpdateListFields(data) {
        if (this.IsBaselineView) return

        if (this.isRunCases && !this.refreshItemByWS) return

        const { workflowId } = this.$route.params
        if (data.workflowId == workflowId && data.itemId) {
          this.findItem(data.itemId, (row, index, siblings, parent) => {
            if (this.isRunCases && Array.isArray(data.fieldIds) && data.fieldIds.includes(FieldStatusId)) {
              this.$emit('on-item-status-updated') // 执行用例状态更新后要刷新父执行的统计信息
            }
            // this.refreshItemById(data.itemId)
            if (!this.refreshQueue.includes(data.itemId)) {
              this.refreshQueue.push(data.itemId)
            }
            this.processRefreshQueue()
          })
        } else if (this.isReviewBugs) {
          if (data.itemId) {
            this.findItem(data.itemId, (row, index, siblings, parent) => {
              // this.refreshItemById(data.itemId)
              if (!this.refreshQueue.includes(data.itemId)) {
                this.refreshQueue.push(data.itemId)
              }
              this.processRefreshQueue()
            })
          }
        }
      },

      onWSBatchUpdateListFields(data) {
        if (this.IsBaselineView) return

        const { workflowId } = this.$route.params
        if (data.workflowId == workflowId && Array.isArray(data.itemIds)) {
          this.findItems(data.itemIds, (row, index, siblings, parent) => {
            const rowItemId = _.get(row, [FieldIdId, 'value'])
            if (!this.refreshQueue.includes(rowItemId)) {
              this.refreshQueue.push(rowItemId)
            }
          })
          this.processRefreshQueue()
        }
      },

      async processRefreshQueue() {
        if (this.isRefreshing) return
        this.isRefreshing = true

        while (this.refreshQueue.length > 0) {
          const itemId = this.refreshQueue.shift()
          try {
            await this.refreshItemById(itemId)
            await new Promise((resolve) => setTimeout(resolve, 50))
          } catch (error) {
            console.error(error)
          }
        }

        this.isRefreshing = false
      },

      getRowClassName({ row }) {
        let classArr = []
        if (_.get(row, `${FieldIdId}.value`)) {
          const unitClassName = `Row_${_.get(row, `${FieldIdId}.value`)}`
          classArr.push(unitClassName)
        }
        if (row._noPermissionOnItem) {
          classArr.push('row-no-permission')
        }
        if (this.isMeetingMode && this.commentTargetItemId) {
          const _subjects = _.get(row, `${FieldSubjectId}.value`)
          if (Array.isArray(_subjects) && _subjects.some((item) => _.get(item, 'item.id') === this.commentTargetItemId)) {
            classArr.push('highlight-current')
          }
        }
        if (this.hasGroupBy) {
          if (row.isGroup) {
            classArr.push(`row-group-title lv-${row.level}`)
          } else {
            classArr.push('work-item-row')
          }
        }
        if (this.useSelection) {
          classArr.push('selectable-row-pointer')
        }
        // if (this.isConfigWorkflow) {
        //   const _flags = _.get(row, [FieldFlagsId, 'value'], 0)
        //   if (checkBit(_flags, TASK_FLAGS.FOLDER)) {
        //     classArr.push('config-version-folder')
        //   }
        // }
        return classArr.join(' ')
      },

      loopItem(callback = () => {}) {
        const _loop = (data, parent) => {
          if (Array.isArray(data)) {
            data.forEach((item, index) => {
              callback(item, index, data, parent)
              if (Array.isArray(item.children)) {
                _loop(item.children, item)
              }
            })
          }
        }
        _loop(this.tableData)
      },
      findItem(itemId, callback = () => {}) {
        const _loop = (data, parent) => {
          if (Array.isArray(data)) {
            data.forEach((item, index) => {
              if (_.get(item, [FieldIdId, 'value']) === itemId || _.get(item, 'id') === itemId) {
                callback(item, index, data, parent)
              }
              if (Array.isArray(item.children)) {
                _loop(item.children, item)
              }
            })
          }
        }
        _loop(this.tableData)
      },
      findItems(itemIds, callback = () => {}) {
        const _loop = (data, parent) => {
          if (Array.isArray(data)) {
            data.forEach((item, index) => {
              if ((_.get(item, [FieldIdId, 'value']) && itemIds.includes(_.get(item, [FieldIdId, 'value']))) || (_.get(item, 'id') && itemIds.includes(_.get(item, 'id')))) {
                callback(item, index, data, parent)
              }
              if (Array.isArray(item.children)) {
                _loop(item.children, item)
              }
            })
          }
        }
        _loop(this.tableData)
      },
      forEachItem(callback = () => {}) {
        const _loop = (data, parent) => {
          if (Array.isArray(data)) {
            data.forEach((item, index) => {
              callback(item, index, data, parent)
              if (Array.isArray(item.children)) {
                _loop(item.children, item)
              }
            })
          }
        }
        _loop(this.tableData)
      },

      setListItemNoPermission(itemId) {
        this.findItem(itemId, (row, index, siblings, parent) => {
          this.$set(row, '_noPermissionOnItem', true)
          // this.dynamicKey++
          this.doAfterTableRender()
        })
      },

      highlightCurrentItem() {
        if (!this.isRunCases) {
          const { workflowId, runItemId } = this.$route.params
          const _key = this.isRunCases ? `${workflowId}_${runItemId}` : workflowId
          const _page = getListPage(_key)
          if (_page && _page.nodeId) {
            this.findItem(_page.nodeId, (row, index, siblings, parent) => {
              setListPage(workflowId, { nodeId: undefined })

              this.$refs.taskListTable && this.$refs.taskListTable.setCurrentRow(row)
              const $el = document.querySelector(`.Row_${_page.nodeId}`)
              if ($el) {
                $el.scrollIntoView({ block: 'center' })
              }
            })
          }
        }
      },

      cancelUnfinishedRequest() {
        Object.keys(this.cancelReqMap).forEach((key) => {
          if (typeof this.cancelReqMap[key] === 'function') {
            this.cancelReqMap[key]()
          }
        })
      },

      focusItemInList(item) {
        this.focusItem = item
        const { workflowId } = this.$route.params
        setListPage(workflowId, { nodeId: item.id })
        this.$nextTick(() => {
          this.handleGetWorkflowItems()
        })
      },

      // 设置表头颜色 Start
      handleKeyDown(event) {
        // 同时检测 Ctrl 键（Windows）和 Command 键（macOS）
        if (event.ctrlKey || event.metaKey) {
          this.modifierKeyPressed = true

          // 当按下修饰键时，为表格容器添加鼠标事件监听
          if (this.$refs.TaskListRef) {
            this.$refs.TaskListRef.addEventListener('mousedown', this.handleMouseDown)
          }
        }
      },

      handleKeyUp(event) {
        // 检测是否释放了修饰键
        // 注意：event.ctrlKey 和 event.metaKey 在 keyup 事件中已经是 false
        // 所以我们需要检查 key 值
        if (event.key === 'Control' || event.key === 'Meta') {
          this.modifierKeyPressed = false

          // 当释放修饰键时，移除鼠标事件监听
          if (this.$refs.TaskListRef) {
            this.$refs.TaskListRef.removeEventListener('mousedown', this.handleMouseDown)
          }
        }
      },

      handleMouseDown(event) {
        // 只有在按下修饰键的情况下才启用选择功能
        if (!this.modifierKeyPressed) return

        // 获取表格表头区域
        // const tableHeader = this.$refs.taskListTable.$el.querySelector('.el-table__header-wrapper')
        const listWrap = this.$refs.TaskListRef // 整个页面都可以

        // 检查点击是否在表头区域内
        const headerRect = listWrap.getBoundingClientRect()
        if (event.clientX < headerRect.left || event.clientX > headerRect.right || event.clientY < headerRect.top || event.clientY > headerRect.bottom) {
          return
        }

        // 开始选择
        this.isSelecting = true
        this.selectedHeaders = []

        // 记录选择起始位置（相对于 TaskListRef 的位置）
        const wrapRect = this.$refs.TaskListRef.getBoundingClientRect()
        this.selectionStart = {
          x: event.clientX - wrapRect.left,
          y: event.clientY - wrapRect.top,
        }
        this.selectionEnd = { ...this.selectionStart }

        // 添加鼠标移动和鼠标释放事件监听
        document.addEventListener('mousemove', this.handleMouseMove)
        document.addEventListener('mouseup', this.handleMouseUp)

        // 阻止默认行为和事件冒泡
        event.preventDefault()
        event.stopPropagation()
      },

      handleMouseMove(event) {
        if (!this.isSelecting) return

        // 更新选择结束位置
        const wrapRect = this.$refs.TaskListRef.getBoundingClientRect()
        this.selectionEnd = {
          x: event.clientX - wrapRect.left,
          y: event.clientY - wrapRect.top,
        }

        // 检测选中的表头
        this.detectSelectedHeaders()
      },

      handleMouseUp() {
        if (!this.isSelecting) return

        // 结束选择 延迟 50ms 防止触发点击页面取消选中的事件
        setTimeout(() => {
          this.isSelecting = false
        }, 50)

        // 移除事件监听
        document.removeEventListener('mousemove', this.handleMouseMove)
        document.removeEventListener('mouseup', this.handleMouseUp)

        // 处理选中的表头
        this.processSelectedHeaders()
      },

      detectSelectedHeaders() {
        // 获取所有表头单元格
        const headerCells = this.$refs.taskListTable.$el.querySelectorAll('.el-table__header-wrapper th')

        // 获取选择框的位置和尺寸
        const selectionRect = {
          left: Math.min(this.selectionStart.x, this.selectionEnd.x),
          top: Math.min(this.selectionStart.y, this.selectionEnd.y),
          right: Math.max(this.selectionStart.x, this.selectionEnd.x),
          bottom: Math.max(this.selectionStart.y, this.selectionEnd.y),
        }

        // 检查每个表头单元格是否与选择框相交
        this.selectedHeaders = []
        const wrapRect = this.$refs.TaskListRef.getBoundingClientRect()

        headerCells.forEach((cell, index) => {
          const _className = cell.getAttribute('class') || ''
          const cellRect = cell.getBoundingClientRect()
          const relativeRect = {
            left: cellRect.left - wrapRect.left,
            top: cellRect.top - wrapRect.top,
            right: cellRect.right - wrapRect.left,
            bottom: cellRect.bottom - wrapRect.top,
          }

          // 检查单元格是否与选择框相交
          if (
            relativeRect.left < selectionRect.right &&
            relativeRect.right > selectionRect.left &&
            relativeRect.top < selectionRect.bottom &&
            relativeRect.bottom > selectionRect.top &&
            _className.indexOf(this.labelClassNamePrefix) !== -1
          ) {
            const _arr = _className.split(' ')
            let cellKey = ''
            let cellUniClass = ''
            _arr.some((str) => {
              if (typeof str === 'string' && str.trim().indexOf(this.labelClassNamePrefix) === 0) {
                cellKey = str.trim().replace(this.labelClassNamePrefix, '')
                cellUniClass = str.trim()
              }
            })
            this.selectedHeaders.push({
              index,
              element: cell,
              cellKey,
              cellUniClass,
            })
          }
        })
      },

      processSelectedHeaders() {
        // 这里可以处理选中的表头，例如发出事件通知父组件
        if (this.selectedHeaders.length > 0) {
          // 可以在这里触发事件，将选中的表头信息传递给父组件
          this.$emit('headers-selected', this.selectedHeaders)

          // 或者直接在这里处理选中的表头
          // console.log('选中的表头:', this.selectedHeaders, this.$refs.ThBgColorPicker)

          if (this.$refs.ThBgColorPicker) {
            this.$refs.ThBgColorPicker.showPicker = true
            this.$nextTick(() => {
              if (this.$refs.ThBgColorPicker.popperElm) {
                $($(this.$refs.ThBgColorPicker.popperElm)).prepend(`<div class="el-color-picker-title">${translateTitle('task.TaskList.setThBgColor')}</div>`)
                $($(this.$refs.ThBgColorPicker.popperElm)).find('.el-color-dropdown__btns').css('padding-right', '10px')
                $($(this.$refs.ThBgColorPicker.popperElm)).find('.el-color-dropdown__btns .el-button--default').remove()
              }
              // const popperElm = _.get(this.$refs.ThBgColorPicker, 'popperElm')
              //   if (popperElm) {
              //     const rect = popperElm.getBoundingClientRect()
              //     if (rect.right && window.innerWidth && rect.right > window.innerWidth) {
              //       if (_.get(popperElm, 'style.left')) {
              //         setTimeout(() => {
              //           this.$refs.ThBgColorPicker.popperElm.style.left = parseInt(popperElm.style.left) - (rect.right - window.innerWidth - 50) + 'px'
              //         }, 100)
              //       }
              //     }
              //   }
            })
          }
        }
      },

      // 处理调整大小的鼠标按下事件
      handleResizeHandleMouseDown(event) {
        if (this.selectedHeaders.length === 0) return

        this.isResizing = true
        this.resizeStartX = event.clientX

        // 获取当前选中表头的总宽度
        const style = this.selectedHeadersBoxStyle
        this.resizeStartWidth = parseInt(style.width)

        // 阻止默认行为和事件冒泡
        event.preventDefault()
        event.stopPropagation()
      },

      // 处理调整大小的鼠标移动事件
      handleResizeMouseMove(event) {
        if (!this.isResizing) return

        // 计算宽度变化
        const deltaX = event.clientX - this.resizeStartX
        const newWidth = this.resizeStartWidth + deltaX

        // 调整所有选中表头的宽度
        if (newWidth > 50) {
          // 设置最小宽度限制
          this.resizeSelectedHeaders(newWidth)
        }
      },

      // 处理调整大小的鼠标释放事件
      handleResizeMouseUp() {
        this.isResizing = false
      },

      // 调整选中表头的宽度
      resizeSelectedHeaders(newTotalWidth) {
        if (this.selectedHeaders.length === 0) return

        // 计算每个表头应该分配的宽度
        const widthPerHeader = Math.floor(newTotalWidth / this.selectedHeaders.length)

        // 更新每个选中表头的宽度
        this.selectedHeaders.forEach((header) => {
          const colKey = header.cellKey
          if (colKey) {
            const [wkfId, fieldId] = colKey.split('_')

            // 查找对应的列并更新宽度
            this.tbDynamicColumns.forEach((col) => {
              if (`${col.wkfId || 0}_${col.id}` === colKey) {
                this.$set(col, 'width', widthPerHeader)
                this.$set(col, 'newWidth', widthPerHeader)
              }
            })
          }
        })

        // 触发表格重新渲染
        // this.dynamicKey++
        this.doAfterTableRender()

        // 通知父组件列宽已更新
        this.$nextTick(() => {
          let fields = this.getTableHeaderFields()
          this.$emit('update-filter-fields', fields)
        })
      },

      handleAppClick() {
        if (!this.isSelecting) {
          this.thBgColorValue = undefined
          this.selectedHeaders = []
        }
      },
      handleConfirmBgColor(value) {
        this.selectedHeaders.forEach((item) => {
          if (item.cellUniClass) {
            if (value) {
              $(`.${item.cellUniClass} .task-list-header`).css('background-color', value)
              $(`.${item.cellUniClass} .task-list-header`).css('color', getContrastTextColor(value))
            } else {
              $(`.${item.cellUniClass} .task-list-header`).css('background-color', '')
              $(`.${item.cellUniClass} .task-list-header`).css('color', '')
            }

            this.tbDynamicColumns.some((col) => {
              if (`${col.wkfId || ''}_${col.id}` === item.cellKey) {
                if (value) {
                  this.$set(col, 'backgroundColor', value)
                  this.$set(col, 'color', getContrastTextColor(value))
                } else {
                  this.$set(col, 'backgroundColor', undefined)
                  this.$set(col, 'color', undefined)
                }
                return true
              }
            })
          }
        })
        this.emitFilterFields(false, false)

        this.thBgColorValue = undefined
        this.selectedHeaders = []
      },
      handleThBgActiveChange(value) {
        this.selectedHeaders.forEach((item) => {
          if (item.cellUniClass) {
            $(`.${item.cellUniClass} .task-list-header`).css('background-color', value)
            $(`.${item.cellUniClass} .task-list-header`).css('color', getContrastTextColor(value))

            this.tbDynamicColumns.some((col) => {
              if (`${col.wkfId || ''}_${col.id}` === item.cellKey) {
                if (value) {
                  this.$set(col, 'backgroundColor', value)
                  this.$set(col, 'color', getContrastTextColor(value))
                } else {
                  this.$set(col, 'backgroundColor', undefined)
                  this.$set(col, 'color', undefined)
                }
                return true
              }
            })
          }
        })
        this.emitFilterFields(false, false)
      },
      handleCancelBgColor() {
        this.selectedHeaders.forEach((item) => {
          if (item.cellUniClass) {
            this.tbDynamicColumns.some((col) => {
              if (`${col.wkfId || ''}_${col.id}` === item.cellKey) {
                if (col.backgroundColor) {
                  $(`.${item.cellUniClass} .task-list-header`).css('background-color', col.backgroundColor)
                  $(`.${item.cellUniClass} .task-list-header`).css('color', getContrastTextColor(col.backgroundColor))
                } else {
                  $(`.${item.cellUniClass} .task-list-header`).css('background-color', '')
                  $(`.${item.cellUniClass} .task-list-header`).css('color', '')
                }
              }
            })
          }
        })

        if (!this.isSelecting) {
          this.thBgColorValue = undefined
          this.selectedHeaders = []
        }
      },

      // 表格重新渲染后执行
      doAfterTableRender() {
        if (this.canSetThColor) {
          this.hideThBorderByBgColor()
          this.cancelSettingBgColorWhenTableScrolling()
        }
      },

      // 监听横向滚动
      cancelSettingBgColorWhenTableScrolling() {
        // 监听表格横向滚动
        this.$nextTick(() => {
          if (this.$refs.taskListTable && this.$refs.taskListTable.$el) {
            // const headerWrapper = this.$refs.taskListTable.$el.querySelector('.el-table__header-wrapper')
            // if (headerWrapper) {
            //   headerWrapper.removeEventListener('scroll', this.handleTableScroll)
            //   headerWrapper.addEventListener('scroll', this.handleTableScroll)
            // }
            const bodyWrapper = this.$refs.taskListTable.$el.querySelector('.el-table__body-wrapper')
            if (bodyWrapper) {
              bodyWrapper.removeEventListener('scroll', this.handleTableScroll)
              bodyWrapper.addEventListener('scroll', this.handleTableScroll)
            }
          }
        })
      },

      // 处理表格横向滚动
      handleTableScroll(event) {
        // 如果有选中的表头并且正在设置背景色，则取消设置
        if (this.selectedHeaders.length > 0) {
          this.handleAppClick()
        }
      },

      // 去掉连续相同背景色的表头单元格之间的右边框
      hideThBorderByBgColor() {
        this.$nextTick(() => {
          this.tbDynamicColumns.forEach((col, cIndex) => {
            const nextCol = this.tbDynamicColumns[cIndex + 1]
            if (!nextCol) return

            if (col.backgroundColor && col.backgroundColor === nextCol.backgroundColor) {
              $(`.${this.labelClassNamePrefix}${col.wkfId || ''}_${col.id}`).css('border-right', 'none')
            }
          })
        })
      },
      // 设置表头颜色 End

      // 查看配置列表
      handleOpenConfigItemDialog(data) {
        const { bsid } = this.$route.query
        this.$refs.ConfigItemDialog && this.$refs.ConfigItemDialog.open({ ...data, queryBsid: bsid ? +bsid : undefined })
      },

      handleConfigItemDialogClose(data) {
        if (data && data.hasUpdated) {
          this.refreshItemById(data.itemId)
        }
      },

      // 配置升级
      handleOpenConfigUpgradeDialog(data) {
        if (data && data.id && this.$refs.CreateConfigVersionDialog) {
          const _data = _.cloneDeep(data)
          _data._upgradeBsVersion = true
          getConfigVersionConfigItems({ id: _data.id }).then((res) => {
            _data.configs = res && Array.isArray(res.items) ? res.items.filter((it) => !!it.target) : []
            this.$refs.CreateConfigVersionDialog.open(_data)
          })
        }
      },
      onConfigUpgradeCreated(data) {
        if (this.isTreeListMode) {
          if (data) {
            if (data.parentId) {
              this.refreshChildrenById(data.parentId)
            } else {
              this.handleGetWorkflowItems(null, false)
            }
          }
        } else {
          this.handleGetWorkflowItems(null, false)
        }
      },

      genGroupAggregateInfo(row) {
        // console.log(row.key, row.value, _.cloneDeep(row))
        let strArr = []
        if (Array.isArray(this.aggregates)) {
          this.aggregates.forEach((item) => {
            const _alias = item.alias ? item.alias.toLowerCase() : ''
            if (_alias && typeof row[_alias] === 'object') {
              let _value = row[_alias].value
              if (_value && !isNaN(_value)) {
                if (row[_alias].fieldType === FieldClassDuration) {
                  _value = drFormatDuration(Number(_value))
                } else {
                  _value = Math.round(Number(_value) * 100) / 100

                  if (row[_alias].fieldId === FieldProgressId || row[_alias].fieldId === FieldSpentEstimatedHourId) {
                    _value = _value + '%'
                  }
                }
              }

              if (row[_alias].field) {
                strArr.push(`${row[_alias].function}(${row[_alias].field}): ${_value}`)
              } else {
                strArr.push(`${_alias}: ${_value}`)
              }
            }
          })
        }
        // console.log('strArr', strArr)
        return strArr.join(', ')
      },

      // 右键菜单 Start
      // 详情
      gotoProperty() {
        const { bsid } = this.$route.query
        const itemId = _.get(this.currentRow, 'itemId')
        if (itemId) {
          this.$router.push({ path: ItemRoutePath + itemId, query: { bsid: bsid ? bsid : undefined } })
        }
      },

      // 修改
      async editTask() {
        const workflowId = _.get(this.workflowInfo, 'id')
        const itemId = _.get(this.currentRow, 'itemId')
        const workItem = await getWorkflowItemInfo({ workflowId, itemId })
        if (workItem) {
          this.$refs.UpdateTaskDialog.open(workItem)
        }
      },
      onItemUpdated(data) {
        this.refreshItemById(data.id)
      },

      // 追溯此项
      openTraceability() {
        const itemId = _.get(this.currentRow, 'itemId')
        this.$refs.TraceabilityDialog.open({ itemId: itemId })
      },

      async onMenuInsertChild() {
        const workflowId = _.get(this.workflowInfo, 'id')
        const itemId = _.get(this.currentRow, 'itemId')
        const workItem = await getWorkflowItemInfo({ workflowId, itemId })
        if (workItem) {
          const targetWorkflows = [this.workflowInfo]
          const params = { [FieldParentId]: [{ item: { ...workItem, workflow: this.workflowInfo }, association: {} }] }

          this.$baseEventBus.$emit('create-task', { targetWorkflows, params })
        }
      },

      // 添加配置
      async handleAddChild() {
        const workflowId = _.get(this.workflowInfo, 'id')
        const itemId = _.get(this.currentRow, 'itemId')
        const workItem = await getWorkflowItemInfo({ workflowId, itemId })
        if (workItem) {
          const targetWorkflows = [this.workflowInfo]
          const params = { [FieldParentId]: [{ item: { ...workItem, workflow: this.workflowInfo }, association: {} }] }

          this.$baseEventBus.$emit('create-task', { targetWorkflows, params })
        }
      },
      // 配置升级
      async handleConfigUpgrade() {
        const workflowId = _.get(this.workflowInfo, 'id')
        const itemId = _.get(this.currentRow, 'itemId')
        const workItem = await getWorkflowItemInfo({ workflowId, itemId })
        if (workItem) {
          const data = { ...workItem, workflow: this.workflowInfo }
          this.$baseEventBus.$emit('open-config-upgrade-dialog', data)
        }
      },

      // 删除工作项
      onMenuDelete() {
        const itemName = _.get(this.currentRow, `${FieldNameId}.value`)
        const itemId = _.get(this.currentRow, 'itemId')
        let message = itemName
          ? this.translateTitle('task.detail.deleteWorkItemConfirm', { itemName: itemName })
          : this.translateTitle('task.detail.deleteWorkItemIdConfirm', { itemId: itemId })
        this.$baseConfirm(message, this.translateTitle('common.moveToTrash'), () => {
          const workflowId = _.get(this.workflowInfo, 'id')
          deleteWorkflowItem({ wkfId: Number(workflowId), itemId: itemId })
            .then((data) => {
              this.onWorkflowItemDelete({ itemId: itemId })
              removeCacheAfterDeleteItem({ wkfId: Number(workflowId), itemId: itemId })
              this.$message.success(this.translateTitle('common.operationSuccess'))
              if (this.currentRow && itemId === this.currentRow.itemId) {
                this.currentRow = null
              }
            })
            .catch(() => {})
        })
      },

      // 分享
      shareItemId() {
        copyToClip(`#${this.currentRow.itemId}`, () => {
          this.$baseMessage(this.translateTitle('common.copySuccess'), 'success')
        })
      },
      shareItemLink() {
        const { href } = this.$router.resolve({ path: ItemRoutePath + this.currentRow.itemId })
        copyToClip(window.location.origin + href, () => {
          this.$baseMessage(this.translateTitle('common.copyLinkSuccess'), 'success')
        })
      },
      // 右键菜单 End
    },
  }
</script>
<style lang="scss" scoped>
  .task-list {
    height: calc(100vh - 90px);
    padding: 20px 20px 0;
    overflow: auto;
  }

  .selection-box {
    position: absolute;
    border: 1px dashed #409eff;
    background-color: rgba(64, 158, 255, 0.1);
    pointer-events: none;
    z-index: 2001;
  }

  .selected-headers-box {
    position: absolute;
    border: 2px solid #409eff;
    background-color: rgba(64, 158, 255, 0.05);
    pointer-events: none;
    z-index: 2000;

    .resize-handle {
      position: absolute;
      right: -5px;
      top: 0;
      width: 10px;
      height: 100%;
      cursor: col-resize;
      pointer-events: auto;

      &:hover {
        background-color: rgba(64, 158, 255, 0.3);
      }
    }

    ::v-deep {
      .th-bg-color-picker {
        position: absolute;
        top: 0;
        right: 140px;
        opacity: 0;
      }
    }
  }

  ::v-deep {
    // .el-form-item {
    //   margin-bottom: 0;
    // }
    .el-input__inner {
      padding: 0 8px;
    }
    // .el-form-item--mini .el-form-item__label {
    //   overflow: hidden;
    //   font-size: 12px;
    //   font-weight: bold;
    //   line-height: 28px;
    //   color: #333;
    //   text-overflow: ellipsis;
    //   white-space: nowrap;
    // }
    .field-value {
      display: flex;
      align-items: center;
      width: 100%;
      padding: 0 8px;
      overflow: hidden;
      font-size: 12px;
      line-height: 28px;
      color: #666;
      background-color: transparent;
      transition: background-color 0.2s ease-in-out 0s, border-color 0.2s ease-in-out 0s;
    }

    .el-table__header {
      .el-table__cell {
        .cell {
          padding: 0;
          .task-list-header {
            display: flex;
            align-items: center;
            justify-content: space-between;
            height: 30px;
            padding: 0 4px 0 8px;
            color: #333;

            .col-controller {
              display: none;
            }

            .col-title {
              font-weight: 500;
              // &.has-wkfId {
              //   color: #44546f;
              // }
            }
          }
          &:hover {
            .task-list-header {
              .col-controller {
                display: inline-block;
              }
            }
          }
        }
        &.el-table-column--selection {
          .cell {
            background-color: transparent !important;
          }
        }
      }
    }
  }

  .task-list-table {
    &.allow-drag-header {
      ::v-deep {
        .el-table__header {
          tr:hover {
            .el-table__cell {
              border-right: 1px solid #ebeef5 !important;
            }
          }
          .el-table__cell {
            .cell {
              .task-list-header {
                &:hover {
                  opacity: 0.7;
                }
              }
            }
          }
        }
      }
    }

    &.allow-header-color {
      .task-list-header {
        background-color: #f5f7fa;
      }
    }

    .task-list-header {
      &.groups-table-header {
        background-color: #e3e8f2;
      }
    }

    .group-label {
      color: #333;
      font-size: 13px;
    }
    .group-value {
      color: #333;
      font-size: 13px;
    }
    .group-count {
      color: #606266;
      font-size: 12px;
    }

    ::v-deep {
      .el-table__body {
        tr.highlight-current {
          td.el-table__cell {
            background-color: rgba($color: #fef6d5, $alpha: 0.3);
          }

          &:hover {
            td.el-table__cell {
              background-color: #f5f7fa;
            }
          }

          &.current-row {
            td.el-table__cell {
              background-color: #ecf5ff;
            }
          }
        }
      }

      .el-loading-mask {
        background-color: rgba($color: #fff, $alpha: 0.5);
        .el-loading-spinner {
          right: 0;
          margin-top: 0;
          transform: translateY(-50%);
        }
      }
    }

    &.tree-list-table {
      ::v-deep {
        .el-table__body {
          .el-table__cell {
            .cell {
              display: flex;
              align-items: center;
            }
          }

          .tree-node-name {
            .el-table__placeholder {
              width: 23px;
            }
            .col-field-wrap {
              &.is-leaf {
                margin-left: 23px;
              }
              .workflow-name {
                padding-left: 0 !important;
              }
            }
          }
        }
      }
    }
  }

  .config-version-table {
    ::v-deep {
      // .el-table__header {
      //   .el-table-column--selection {
      //     .el-checkbox {
      //       display: none;
      //     }
      //   }
      // }

      // .config-version-folder {
      //   .el-table-column--selection {
      //     .el-checkbox {
      //       display: none;
      //     }
      //   }
      // }
    }
  }

  .x-field-editor {
    &.status-icon-dropdown {
      display: inline-block;
      width: 100%;
      padding: 0 8px;
    }

    &.is-required {
      ::v-deep {
        .el-input__inner,
        .el-textarea__inner,
        .el-color-picker__trigger {
          border-color: #f5222d;
        }
        .el-select .el-input.is-focus .el-input__inner {
          border-color: #f5222d;
        }
        .pro-reference.focus,
        .pro-member.focus {
          border-color: #f5222d;
        }
      }
    }

    ::v-deep {
      .el-dropdown {
        width: 100%;
      }
      .status-dropdown-reference {
        display: inline-flex;
        width: 100%;
      }
      .status-disabled {
        width: 100%;
        .status-label {
          width: 100%;
        }
      }
    }
  }

  .columns-select {
    padding: 6px 0;
    max-height: 300px;
    overflow-y: auto;
    .group-label {
      padding: 0 10px;
      line-height: 24px;
      color: #999;
      font-size: 12px;
    }
    .option-item {
      padding: 4px 16px;
      line-height: 24px;
      color: #333;
      font-size: 12px;
      cursor: pointer;
      user-select: none;
      &:hover {
        background-color: #f5f7fa;
      }
      &.active {
        color: #409eff;
        background-color: #e9f2fe;
        .dr-icon {
          color: #409eff;
        }
      }
    }
    .no-columns {
      padding: 4px 16px;
      line-height: 24px;
      color: #999;
      font-size: 12px;
      text-align: center;
    }
  }

  ::v-deep {
    .workflow-name {
      padding: 0 8px;
      font-size: 14px;
      line-height: 20px;
      .id-label {
        padding: 0;
        font-size: 14px;
        line-height: 20px;
        color: inherit;
        border-radius: unset;
        background-color: transparent;
      }
    }

    .el-table-column--selection {
      .cell {
        padding-left: 8px !important;
      }
    }
  }

  .btn-refresh {
    padding: 7px;
  }

  .list-loading-mask {
    position: absolute;
    z-index: 2000;
    // background-color: rgba(255, 255, 255, 0.7);
    margin: 0;
    top: 30px;
    right: 0;
    bottom: 32px;
    left: 0;
    transition: opacity 0.3s;
  }
</style>
