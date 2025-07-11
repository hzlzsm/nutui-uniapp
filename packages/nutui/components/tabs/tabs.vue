<script setup lang="ts">
import type { ComponentInternalInstance, CSSProperties, Ref, VNode } from 'vue'
import { computed, defineComponent, getCurrentInstance, nextTick, onActivated, onMounted, ref, useSlots, watch } from 'vue'
import { CHANGE_EVENT, CLICK_EVENT, PREFIX, UPDATE_MODEL_EVENT } from '../_constants'
import { useProvide, useRect, useSelectorQuery } from '../_hooks'
import { getMainClass, getRandomId, pxCheck, TypeOfFun } from '../_utils'
import raf from '../_utils/raf'
import NutIcon from '../icon/icon.vue'
import { useTabContentTouch } from './hooks'
import { TAB_KEY, tabsEmits, tabsProps, Title } from './tabs'

const props = defineProps(tabsProps)

const emit = defineEmits(tabsEmits)

const slots = useSlots()

const instance = getCurrentInstance() as ComponentInternalInstance
const { getSelectorNodeInfo, getSelectorNodeInfos } = useSelectorQuery(instance)

const refRandomId = getRandomId()

const container = ref(null)

const { internalChildren } = useProvide(TAB_KEY, `${PREFIX}-tabs`)({
  activeKey: computed(() => props.modelValue || 0),
  autoHeight: computed(() => props.autoHeight),
  animatedTime: computed(() => props.animatedTime),
})

const titles: Ref<Title[]> = ref([])

function renderTitles(vnodes: VNode[]) {
  vnodes.forEach((vnode: VNode, index: number) => {
    let type = vnode.type
    type = (type as any).name || type
    if (type === 'nut-tab-pane') {
      const title = new Title()
      if (vnode.props?.title || vnode.props?.['pane-key'] || vnode.props?.paneKey) {
        const paneKeyType = TypeOfFun(vnode.props?.['pane-key'])
        const paneIndex
              = paneKeyType === 'number' || paneKeyType === 'string' ? String(vnode.props?.['pane-key']) : null
        const camelPaneKeyType = TypeOfFun(vnode.props?.paneKey)
        const camelPaneIndex
              = camelPaneKeyType === 'number' || camelPaneKeyType === 'string' ? String(vnode.props?.paneKey) : null
        title.title = vnode.props?.title
        title.paneKey = paneIndex || camelPaneIndex || String(index)
        title.disabled = vnode.props?.disabled
      }

      titles.value.push(title)
    }
    else {
      if (vnode.children === ' ')
        return

      renderTitles(vnode.children as VNode[])
    }
  })
}

const currentIndex = ref((props.modelValue as number) || 0)

function findTabsIndex(value: string | number) {
  const index = titles.value.findIndex(item => item.paneKey === String(value))
  // if (titles.value.length === 0)
  //   console.warn('[NutUI] <Tabs> 当前未找到 TabPane 组件元素 , 请检查 .')

  // else if (index === -1)
  //   console.warn('[NutUI] <Tabs> 请检查 v-model 值是否为 paneKey ,如 paneKey 未设置，请采用下标控制 .')

  // else
  currentIndex.value = index
}

const getScrollX = computed(() => {
  return props.titleScroll && props.direction === 'horizontal'
})

const getScrollY = computed(() => {
  return props.titleScroll && props.direction === 'vertical'
})

const titleRef = ref([]) as Ref<HTMLElement[]>
const scrollLeft = ref(0)
const scrollTop = ref(0)
const scrollWithAnimation = ref(false)
const navRectRef = ref()
const titleRectRef = ref<UniApp.NodeInfo[]>([])
const canShowLabel = ref(false)

function scrollIntoView() {
  if (!props.titleScroll || slots.titles)
    return
  raf(() => {
    Promise.all([
      getSelectorNodeInfo(`#nut-tabs__titles_${refRandomId}`),
      getSelectorNodeInfos(`#nut-tabs__titles_${refRandomId} .nut-tabs__titles-item`),
    ]).then(([navRect, titleRects]) => {
      navRectRef.value = navRect
      titleRectRef.value = titleRects

      if (navRectRef.value) {
        if (props.direction === 'vertical') {
          const titlesTotalHeight = titleRects.reduce((prev: number, curr: UniApp.NodeInfo) => prev + curr.height!, 0)
          if (titlesTotalHeight > navRectRef.value?.height)
            canShowLabel.value = true

          else
            canShowLabel.value = false
        }
        else {
          const titlesTotalWidth = titleRects.reduce((prev: number, curr: UniApp.NodeInfo) => prev + curr.width!, 0)
          if (titlesTotalWidth > navRectRef.value?.width)
            canShowLabel.value = true

          else
            canShowLabel.value = false
        }
      }

      const titleRect: UniApp.NodeInfo = titleRectRef.value[currentIndex.value]

      let to = 0
      if (props.direction === 'vertical') {
        const top = titleRects
          .slice(0, currentIndex.value)
          .reduce((prev: number, curr) => prev + curr.height!, 0)
        to = top - (navRectRef.value?.height - titleRect.height!) / 2
      }
      else {
        const left = titleRects
          .slice(0, currentIndex.value)
          .reduce((prev: number, curr) => prev + curr.width!, 0)
        // eslint-disable-next-line  ts/no-non-null-asserted-optional-chain
        to = left - (navRectRef.value?.width - titleRect?.width!) / 2
      }

      nextTick(() => {
        scrollWithAnimation.value = true
      })

      scrollDirection(to, props.direction)
    })
  })
}

function scrollDirection(to: number, direction: 'horizontal' | 'vertical') {
  let count = 0
  const from = direction === 'horizontal' ? scrollLeft.value : scrollTop.value
  const frames = 1

  function animate() {
    if (direction === 'horizontal')
      scrollLeft.value += (to - from) / frames
    else
      scrollTop.value += (to - from) / frames

    if (++count < frames)
      raf(animate)
  }

  animate()
}
function init(vnodes: VNode[] = internalChildren.map(item => item.vnode)) {
  titles.value = []
  vnodes = vnodes?.filter(item => typeof item.children !== 'string')

  if (vnodes && vnodes.length)
    renderTitles(vnodes)

  findTabsIndex(props.modelValue)
  setTimeout(() => {
    scrollIntoView()
  }, 500)
}

watch(
  () => internalChildren.map(item => item.props),
  (vnodes: any[]) => {
    init(internalChildren as any)
  },
  { deep: true, immediate: true },
)

watch(
  () => props.modelValue,
  (value: string | number) => {
    findTabsIndex(value)
    scrollIntoView()
  },
)
onMounted(init)
onActivated(init)
const tabMethods = {
  isBegin: () => {
    return currentIndex.value === 0
  },
  isEnd: () => {
    return currentIndex.value === titles.value.length - 1
  },
  next: () => {
    currentIndex.value += 1
    const nextDisabled = titles.value[currentIndex.value].disabled
    if (tabMethods.isEnd() && nextDisabled) {
      tabMethods.prev()
      return
    }
    if (nextDisabled && currentIndex.value < titles.value.length - 1) {
      tabMethods.next()
      return
    }

    tabMethods.updateValue(titles.value[currentIndex.value])
  },
  prev: () => {
    currentIndex.value -= 1
    const prevDisabled = titles.value[currentIndex.value].disabled
    if (tabMethods.isBegin() && prevDisabled) {
      tabMethods.next()
      return
    }
    if (prevDisabled && currentIndex.value > 0) {
      tabMethods.prev()
      return
    }
    tabMethods.updateValue(titles.value[currentIndex.value])
  },
  updateValue: (item: Title) => {
    emit(UPDATE_MODEL_EVENT, item.paneKey)
    emit(CHANGE_EVENT, item)
  },
  tabChange: (item: Title, index: number) => {
    emit(CLICK_EVENT, item)
    if (item.disabled || currentIndex.value === index)
      return

    currentIndex.value = index
    tabMethods.updateValue(item)
  },
  setTabItemRef: (el: HTMLElement, index: number) => {
    titleRef.value[index] = el
  },
}
const { tabChange } = tabMethods
const { touchState, touchMethods, tabsContentID, tabsContentRef } = useTabContentTouch(props, tabMethods, instance, useRect)
const contentStyle = computed(() => {
  let offsetPercent = currentIndex.value * 100
  if (touchState.moving)
    offsetPercent += touchState.offset

  let style: CSSProperties = {
    transform:
          props.direction === 'horizontal'
            ? `translate3d(-${offsetPercent}%, 0, 0)`
            : `translate3d( 0,-${offsetPercent}%, 0)`,
    transitionDuration: touchState.moving ? undefined : `${props.animatedTime}ms`,
  }
  if (props.animatedTime === 0)
    style = {}

  return style
})
const tabsNavStyle = computed(() => {
  return {
    background: props.background,
  }
})
const tabsActiveStyle = computed(() => {
  return {
    color: props.type === 'smile' ? props.customColor : '',
    background: props.type === 'line' ? props.customColor : '',
  }
})
const titleStyle = computed(() => {
  if (!props.titleGutter)
    return {}
  const px = pxCheck(props.titleGutter)
  if (props.direction === 'vertical')
    return { paddingTop: px, paddingBottom: px }

  return { paddingLeft: px, paddingRight: px }
})
const classes = computed(() => {
  return getMainClass(props, componentName, {
    [props.direction]: true,
  })
})
</script>

<script lang="ts">
const componentName = `${PREFIX}-tabs`

export default defineComponent({
  name: componentName,
  options: {
    virtualHost: true,
    addGlobalClass: true,
    styleIsolation: 'shared',
  },
})
</script>

<template>
  <view ref="container" :style="customStyle" :class="classes">
    <scroll-view
      :id="`nut-tabs__titles_${refRandomId}`"
      :scroll-x="getScrollX"
      :scroll-y="getScrollY"
      :scroll-with-animation="scrollWithAnimation"
      :scroll-left="scrollLeft"
      :scroll-top="scrollTop"
      :enable-flex="true"
      class="nut-tabs__titles"
      :class="{ [type]: type, scrollable: titleScroll, [size]: size }"
      :style="tabsNavStyle"
    >
      <view class="nut-tabs__list" :class="{ 'nut-tabs__titles-left': align === 'left' }">
        <slot v-if="$slots.titles" name="titles" />
        <template v-else>
          <view
            v-for="(item, index) in titles"
            :key="item.paneKey"
            class="nut-tabs__titles-item uni"
            :style="titleStyle"
            :class="{
              'nut-tabs-active': item.paneKey === String(modelValue),
              'disabled': item.disabled,
              'nut-tabs__titles-item-left': align === 'left',
            }"
            @click="tabChange(item, index)"
          >
            <view v-if="type === 'line'" class="nut-tabs__titles-item__line" :style="tabsActiveStyle" />
            <view v-if="type === 'smile'" class="nut-tabs__titles-item__smile" :style="tabsActiveStyle">
              <NutIcon name="joy-smile" :custom-color="customColor" />
            </view>
            <view class="nut-tabs__titles-item__text" :class="{ ellipsis }">
              {{ item.title }}
            </view>
          </view>
          <view v-if="canShowLabel && titleScroll" class="nut-tabs__titles-placeholder" />
        </template>
      </view>
    </scroll-view>
    <view
      :id="tabsContentID"
      ref="tabsContentRef"
      class="nut-tabs__content"
      :style="contentStyle"
      @touchstart="touchMethods.onTouchStart"
      @touchmove="touchMethods.onTouchMove"
      @touchend="touchMethods.onTouchEnd"
      @touchcancel="touchMethods.onTouchEnd"
    >
      <slot name="default" />
    </view>
  </view>
</template>

<style lang="scss">
@import './index';
</style>
