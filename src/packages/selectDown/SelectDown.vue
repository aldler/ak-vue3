<!-- Created by 337547038 on 2021/11/25. -->
<template>
  <div
    ref="el"
    :class="{
      'is-down': state.visible,
      [prefixCls + '-select-down']: true,
      disabled: disabledOk
    }"
    :style="{ width: width }"
    @click="downToggle"
  >
    <div class="select-control" @click="selectControlClick">
      <div v-if="isRange" class="select-range" :class="inputCls">
        <input
          v-model="state.valueLabel[0]"
          :readonly="!filterable"
          :placeholder="placeholder"
          :disabled="disabledOk"
          @input="inputInput"
          @focus="inputFocus"
          @blur="inputBlur"
        />
        <span>{{ rangeSeparator }}</span>
        <input
          v-model="state.valueLabel[1]"
          :readonly="!filterable"
          :placeholder="endPlaceholder"
          :disabled="disabledOk"
          @input="inputInput"
          @focus="inputFocus"
          @blur="inputBlur"
        />
      </div>
      <div v-else-if="multiple" :class="inputCls" class="multiple-text">
        <ul :placeholder="placeholder">
          <template v-if="collapseTags">
            <li v-if="state.valueLabel.length > 0">
              <span v-text="state.valueLabel[0]"></span>
              <i class="icon-error" @click.stop="deleteText(0)"></i>
            </li>
            <li v-if="state.valueLabel.length > 1">
              <tag size="mini" type="info"> +{{ state.valueLabel.length }}</tag>
            </li>
          </template>
          <template v-else>
            <li v-for="(item, index) in state.valueLabel" :key="index">
              <span v-text="item"></span>
              <i class="icon-error" @click.stop="deleteText(index)"></i>
            </li>
          </template>
          <li v-if="filterable" class="input">
            <input
              v-model="state.searchValueM"
              type="text"
              :disabled="disabledOk"
              :placeholder="state.valueLabel.length === 0 ? placeholder : ''"
              @input="inputInput"
              @focus="inputFocus"
              @blur="inputBlur"
            />
          </li>
        </ul>
      </div>
      <input
        v-else
        v-model="state.valueLabel[0]"
        :readonly="!filterable"
        :placeholder="placeholder"
        :class="inputCls"
        :disabled="disabledOk"
        @input="inputInput"
        @focus="inputFocus"
        @blur="inputBlur"
      />
      <span class="group-icon">
        <i
          v-if="clear && modelValue.length > 0"
          class="icon-close"
          title="清空"
          @click="clearClick"
        ></i>
        <i
          :class="{ down: state.visible && !fixedIcon, [`icon-${icon}`]: true }"
        ></i>
      </span>
    </div>
    <transition
      :name="state.direction2 === 2 ? 'slide-toggle-top' : 'slide-toggle'"
    >
      <div
        v-show="state.visible"
        ref="selectDown"
        :class="{
          [prefixCls + '-select-down-pane']: true,
          [downClass]: downClass,
          top: state.direction2 === 2
        }"
        :style="downPanelStyle"
        @click.stop=""
      >
        <div :style="downHeightStyle" class="scroll-pane">
          <slot></slot>
        </div>
        <span class="down-arrow" :class="{ 'is-range': isRange }"></span>
      </div>
    </transition>
  </div>
</template>

<script lang="ts" setup>
  import prefixCls from '../prefix'
  import {
    reactive,
    ref,
    computed,
    onMounted,
    onBeforeUnmount,
    nextTick,
    watch
  } from 'vue'
  import { getFormDisabled } from '../util/form'
  import Tag from '../tag/Tag.vue'
  import { getOffset, getWindow } from '../util/dom'

  const props = withDefaults(
    defineProps<{
      modelValue: string[] | number[]
      width?: string
      multiple?: boolean
      collapseTags?: boolean
      clear?: boolean
      filterable?: boolean
      size?: string // 尺寸
      placeholder?: string
      disabled?: boolean
      direction?: number //0自动　1向下　2向上
      downClass?: string
      downStyle?: object
      appendToBody?: boolean
      downHeight?: number // 显示下拉最大高度，超出显示滚动条
      icon?: string
      fixedIcon?: boolean
      isRange?: boolean // 区间选择，此时multiple无效
      rangeSeparator?: string
      endPlaceholder?: string // isRange时第二个输入框
    }>(),
    {
      multiple: false,
      collapseTags: false,
      clear: false,
      filterable: false,
      disabled: false,
      appendToBody: false,
      direction: 0,
      icon: 'arrow',
      isRange: false,
      rangeSeparator: 'To',
      downClass: '',
      downHeight: 0
    }
  )
  const emits = defineEmits<{
    (e: 'update:modelValue', modelValue: string[]): void
    (e: 'blur', value: string | string[]): void
    (e: 'toggleClick', value: boolean, evt: MouseEvent): void
    (e: 'clear'): void
    (e: 'delete', value: number): void
    (e: 'input', value: string | string[]): void
    (e: 'focus', value: string | string[]): void
  }>()
  const el = ref()
  const selectDown = ref()
  const state = reactive({
    valueLabel: ref(JSON.parse(JSON.stringify(props.modelValue || []))),
    visible: false,
    appendStyle: {
      top: '',
      minWidth: '',
      bottom: '',
      left: ''
    },
    direction2: props.direction,
    stopPropagation: false,
    searchValueM: '' // 多选输入框的值
  })
  watch(
    () => props.modelValue,
    (val: any) => {
      state.valueLabel = JSON.parse(JSON.stringify(val))
    }
  )
  const disabledOk = computed(() => {
    return getFormDisabled(props.disabled)
  })
  // 输入框类名
  const inputCls = computed(() => {
    const cls = [`${prefixCls}-input-control`]
    if (props.size) {
      cls.push(props.size)
    }
    if (disabledOk.value) {
      cls.push('disabled')
    }
    return cls.join(' ')
  })
  const downToggle = (evt: MouseEvent) => {
    if (disabledOk.value) {
      return
    }
    state.visible = true
    nextTick(() => {
      setPosition(evt)
      setAppendToBodyStyle()
    })
    emits('toggleClick', state.visible, evt)
    // 设置为true不触发document点击的关闭事件，解决一个页面多次使用组件时，点击时可将展开中的收起
    state.stopPropagation = true
    setTimeout(() => {
      state.stopPropagation = false
    }, 50)
    // evt.stopPropagation()
  }
  // 添加一个事件，在展开的时候点击收起
  const selectControlClick = (evt: MouseEvent) => {
    if (state.visible && !props.filterable) {
      // 可搜索时不能关上
      slideUp(evt)
      evt.stopPropagation()
    }
  }
  const deleteText = (index: number) => {
    state.valueLabel.splice(index, 1)
    updateModel()
    emits('delete', index)
  }
  // 清空
  const clearClick = (evt: MouseEvent) => {
    state.valueLabel = []
    updateModel()
    emits('clear')
    evt.stopPropagation()
  }
  const slideUp = (evt: MouseEvent) => {
    if (state.visible && !state.stopPropagation) {
      // =true就没必要执行了
      evt && emits('toggleClick', false, evt)
      state.visible = false
      // 清空多选可输入时输入框的值
      state.searchValueM = ''
    }
  }
  const updateModel = () => {
    emits('update:modelValue', state.valueLabel)
  }
  const mouseEvent = (evt: MouseEvent | Event, type: any) => {
    if (props.filterable) {
      if (props.isRange) {
        emits(type, state.valueLabel)
        updateModel()
        return
      }
      const { value } = evt.target as HTMLInputElement
      emits(type, value)
      updateModel()
    }
  }
  const inputInput = (e: Event) => {
    mouseEvent(e, 'input')
  }
  const inputBlur = (e: Event) => {
    mouseEvent(e, 'blur')
  }
  const inputFocus = (e: Event) => {
    mouseEvent(e, 'focus')
  }
  // 计算插入body的位置样式
  const setAppendToBodyStyle = () => {
    const offset = getOffset(el.value)
    if (props.appendToBody) {
      const ww = getWindow()
      state.appendStyle = {
        bottom: 'auto',
        minWidth: offset.width + 'px',
        left: offset.left + 'px',
        top: offset.top + offset.height + 8 + 'px'
      }
      if (state.direction2 === 2) {
        // 向上
        state.appendStyle.top = 'auto'
        state.appendStyle.bottom = ww.height - offset.top + 'px'
      }
    } else {
      state.appendStyle.top = offset.height + 8 + 'px'
      state.appendStyle.bottom = 'auto'
      if (state.direction2 === 2) {
        // 向上
        state.appendStyle.top = 'auto'
        state.appendStyle.bottom = offset.height + 8 + 'px'
      }
    }
  }
  const setPosition = (evt: MouseEvent) => {
    if (props.direction === 0) {
      state.direction2 = props.direction
      // 计算弹出方向
      const wh =
        document.documentElement.clientHeight || document.body.clientHeight
      const clientY = evt.clientY // 当鼠标事件发生时，鼠标相对于浏览器（这里说的是浏览器的有效区域）y轴的位置；
      // 最大下拉高度
      let downMaxHeight = props.downHeight || selectDown.value.offsetHeight || 0
      if (downMaxHeight > wh - clientY && clientY > downMaxHeight) {
        // 向上
        state.direction2 = 2
      }
    }
  }
  // 下拉面板style样式
  const downHeightStyle: any = computed(() => {
    if (props.downHeight) {
      return {
        'max-height': props.downHeight + 'px',
        'overflow-y': 'auto'
      }
    }
    return {}
  })
  const downPanelStyle = computed(() => {
    /*let style = {}*/
    /*if (props.downHeight) {
      style = {
        'max-height': props.downHeight + 'px',
        overflowY: 'auto'
      }
    }*/
    // style = Object.assign({}, state.appendStyle, props.downStyle || {})
    return Object.assign({}, state.appendStyle, props.downStyle || {})
  })
  onMounted(() => {
    nextTick(() => {
      document.addEventListener('click', slideUp)
      if (props.appendToBody) {
        document.body.appendChild(selectDown.value)
      }
    })
  })
  onBeforeUnmount(() => {
    document.removeEventListener('click', slideUp)
    if (props.appendToBody && selectDown.value) {
      document.body.removeChild(selectDown.value)
    }
  })
  /** 提供一个方法用于改变显示的值
   * @param value
   * @return
   */
  const setValue = (val: string[]) => {
    state.valueLabel = JSON.parse(JSON.stringify(val))
  }
  defineExpose({ slideUp, setValue })
</script>
