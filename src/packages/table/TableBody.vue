<!-- Created by 337547038 on 2021/6/24 0024. -->
<template>
  <tbody>
    <template v-for="(row, rowIndex) in dataList" :key="rowIndex">
      <tr
        :class="{
          warning: selectedRows.indexOf(row) !== -1,
          [row.trClass]: row.trClass
        }"
        @click="rowClick(row, rowIndex)"
      >
        <table-td
          v-for="(column, indexTd) in colsNoExtend"
          :key="indexTd"
          :checked="selectedRows.indexOf(row) !== -1"
          :column="column"
          :row="row"
          :index="rowIndex"
          :column-index="indexTd"
          :title="title"
          :toggle="getToggle(rowIndex)"
          :row-col-span="rowColSpan"
          :rowspan-colspan-list="state.rowspanColspanList"
          @toggle-extend="toggleExtend(rowIndex, row)"
          @cellClick="cellClick"
        />
      </tr>
      <tr
        v-if="getToggle(rowIndex) && colsExtend.length > 0"
        :key="'tr' + rowIndex"
        class="extend"
        :class="{ warning: selectedRows.indexOf(row) !== -1 }"
      >
        <TableTd
          :column="colsExtend[0]"
          :row="row"
          :index="rowIndex"
          :colspan="colsNoExtend.length"
        />
      </tr>
      <template v-if="hasChild">
        <tr
          v-for="(item, index) in row.children"
          v-show="getToggle(rowIndex)"
          :key="'child' + index"
          :class="{ [row.trClass]: row.trClass }"
          class="tr-child"
          @cllick="rowClick(item, index)"
        >
          <TableTd
            v-for="(child, childIndex) in colsNoExtend"
            :key="'childTd' + childIndex"
            :column="child"
            :row="item"
            :index="index"
            :column-index="childIndex"
            :title="title"
            :parent-row="row"
            @cell-click="cellClick"
          />
        </tr>
      </template>
    </template>
  </tbody>
</template>

<script lang="ts" setup>
  import { reactive, inject, computed, ref } from 'vue'
  import TableTd from './TableTd.vue'
  import prefixCls from '../prefix'
  const props = withDefaults(
    defineProps<{
      data?: any
      rowColSpan?: Function
      hasChild?: boolean
      lazyLoad?: Function
      extendToggle?: boolean // 默认展开或收起状态
      title?: boolean
      selectedRows?: any
    }>(),
    {}
  )
  const emits = defineEmits<{
    (e: 'rowClick', row: any, index: number): void
    (
      e: 'cellClick',
      row: any,
      column: any,
      rowIndex: number,
      columnIndex: number
    ): void
  }>()
  const getColumns = inject(`${prefixCls}GetColumns`) as any
  const state = reactive<any>({
    toggle: {}, // {1: true, 2: false, 0: false} // 对应每行展开或收起状态
    rowspanColspanList: []
  })
  const dataList = ref(props.data) // 这里要转一下，加载子级时才能同步展示
  const colsExtend = computed(() => {
    return getColumns.value.filter((item: any) => {
      return item.type === 'extend' && !item.children
    })
  })
  const colsNoExtend = computed(() => {
    // 不带扩展的
    return getColumns.value.filter((item: any) => {
      return item.type !== 'extend' && !item.children
    })
  })
  const getToggle = (rowIndex: number) => {
    return state.toggle[rowIndex] === undefined
      ? props.extendToggle
      : state.toggle[rowIndex]
  }
  // 展开或收起扩展行
  const toggleExtend = (index: number, row: any) => {
    // 存在扩展时或有子级时
    if (colsExtend.value.length > 0 || props.hasChild) {
      if (typeof state.toggle[index] === 'undefined') {
        state.toggle[index] = !props.extendToggle
      } else {
        state.toggle[index] = !state.toggle[index]
      }
      // 展开时，如果是懒加载
      console.log(state.toggle[index])
      if (state.toggle[index] && props.lazyLoad) {
        props.lazyLoad(row, (child: any) => {
          if (child && child.length > 0) {
            row.children = child
          }
        })
      }
    }
  }
  const rowClick = (row: any, index: number) => {
    emits('rowClick', row, index)
  }
  const cellClick = (
    row: any,
    column: any,
    rowIndex: number,
    columnIndex: number
  ) => {
    emits('cellClick', row, column, rowIndex, columnIndex)
  }
</script>
