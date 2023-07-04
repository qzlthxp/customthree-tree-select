<template>
  <view :class="['select-list', { disabled }, { active: selectList.length }]" @click="open">
    <view class="left">
      <view v-if="selectList.length" class="select-items">
        <view class="select-item" v-for="item in selectedListBaseinfo" :key="item[dataValue]">
          <view class="name">
            <text>{{ item[dataLabel] }}</text>
          </view>
          <view v-if="!disabled" class="close" @click.stop="removeSelectedItem(item[dataValue])">
            <uni-icons type="closeempty" size="16" color="#999"></uni-icons>
          </view>
        </view>
      </view>
      <view v-else style="color: #6a6a6a" class="no-data">
        <text>{{ placeholder }}</text>
      </view>
    </view>
    <view class="right">
      <uni-icons v-if="!selectList.length || !clearable" type="bottom" size="14" color="#999"></uni-icons>
      <view @click.stop>
        <uni-icons
          v-if="selectList.length && clearable"
          type="clear"
          size="24"
          color="#c0c4cc"
          @click="clearSelectList"
        ></uni-icons>
      </view>
    </view>
  </view>
  <uni-popup
    ref="popup"
    :animation="animation"
    :is-mask-click="isMaskClick"
    :mask-background-color="maskBackgroundColor"
    :background-color="backgroundColor"
    :safe-area="safeArea"
    type="bottom"
    @change="change"
    @maskClick="maskClick"
  >
    <view class="popup-content" :style="{ height: contentHeight }">
      <view class="title">
        <view v-if="mutiple && canSelectAll" class="left" @click="handleSelectAll">
          <text>全选</text>
        </view>
        <view class="center">
          <text>{{ placeholder }}</text>
        </view>
        <view class="right" :style="{ color: confirmTextColor }" @click="close">
          <text>{{ confirmText }}</text>
        </view>
      </view>
      <view v-if="search" class="search-box">
        <uni-easyinput
          :maxlength="-1"
          prefixIcon="search"
          placeholder="搜索"
          v-model="searchStr"
          confirm-type="search"
          @confirm="handleSearch(false)"
          @clear="handleSearch(true)"
        >
        </uni-easyinput>
        <button type="primary" size="mini" class="search-btn" @click="handleSearch(false)">搜索</button>
      </view>
      <view v-if="treeData.length" class="select-content">
        <scroll-view class="scroll-view-box" :scroll-top="scrollTop" scroll-y="true" @touchmove.stop>
          <view v-if="!filterTreeData.length" class="no-data center">
            <text>暂无数据</text>
          </view>
          <data-select-item
            v-for="item in filterTreeData"
            :key="item[dataValue]"
            :node="item"
            :dataLabel="dataLabel"
            :dataValue="dataValue"
            :dataChildren="dataChildren"
            :choseParent="choseParent"
            :border="border"
            :lazyLoadChildren="lazyLoadChildren"
          ></data-select-item>
          <view class="sentry" />
        </scroll-view>
      </view>
      <view v-else class="no-data center">
        <text>暂无数据</text>
      </view>
    </view>
  </uni-popup>
</template>

<script lang="ts" setup>
import { ref, computed, watch, onMounted, nextTick, provide } from 'vue'
import dataSelectItem from './data-select-item.vue'
import { isString, paging } from './utils'

const props = defineProps({
  canSelectAll: {
    type: Boolean,
    default: false
  },
  safeArea: {
    type: Boolean,
    default: true
  },
  search: {
    type: Boolean,
    default: false
  },
  clearResetSearch: {
    type: Boolean,
    default: false
  },
  animation: {
    type: Boolean,
    default: true
  },
  'is-mask-click': {
    type: Boolean,
    default: true
  },
  'mask-background-color': {
    type: String,
    default: 'rgba(0,0,0,0.4)'
  },
  'background-color': {
    type: String,
    default: 'none'
  },
  'safe-area': {
    type: Boolean,
    default: true
  },
  choseParent: {
    type: Boolean,
    default: true
  },
  placeholder: {
    type: String,
    default: '请选择'
  },
  confirmText: {
    type: String,
    default: '完成'
  },
  confirmTextColor: {
    type: String,
    default: '#007aff'
  },
  listData: {
    type: Array,
    default: () => []
  },
  dataLabel: {
    type: String,
    default: 'name'
  },
  dataValue: {
    type: String,
    default: 'id'
  },
  dataChildren: {
    type: String,
    default: 'children'
  },
  linkage: {
    type: Boolean,
    default: false
  },
  clearable: {
    type: Boolean,
    default: false
  },
  mutiple: {
    type: Boolean,
    default: false
  },
  disabled: {
    type: Boolean,
    default: false
  },
  showChildren: {
    type: Boolean,
    default: true
  },
  border: {
    type: Boolean,
    default: false
  },
  lazyLoadChildren: {
    type: Boolean,
    default: false
  },
  load: {
    type: Function,
    default: function () {}
  },
  modelValue: {
    type: [Array, String],
    default: () => []
  }
})

const emits = defineEmits(['update:modelValue', 'change', 'maskClick', 'select-change'])

const contentHeight = ref('500px')
const treeData = ref([])
const filterTreeData = ref([])
const clearTimerList = ref([])
const showPopup = ref(false)
const isSelectedAll = ref(false)
const scrollTop = ref(0)
const searchStr = ref('')
const popup = ref<any>(null)

provide('nodeFn', {
  nodeClick: handleNodeClick,
  nameClick: handleHideChildren,
  loadNode: props.load,
  initData: initData,
  addNode: addNode
})

const selectList = computed(() => {
  const newVal = props.modelValue === null ? '' : props.modelValue
  return isString(newVal) ? (newVal.length ? newVal.split(',') : []) : newVal.map((item: any) => item.toString())
})
const selectedListBaseinfo = computed(() => {
  const ids = props.modelValue === null ? '' : props.modelValue
  return getselectedInfo(ids)
})

onMounted(() => {
  getContentHeight(uni.getSystemInfoSync())
})

function getContentHeight({ screenHeight }: { screenHeight: number }) {
  contentHeight.value = `${Math.floor(screenHeight * 0.7)}px`
}

watch(
  () => props.listData,
  (newVal: any[]) => {
    if (newVal) {
      isSelectedAll.value = true
      treeData.value = initData(newVal)
      if (showPopup.value) {
        resetClearTimerList()
        renderTree(treeData.value)
      }
    }
  },
  { immediate: true, deep: true }
)

watch(
  () => props.modelValue,
  (newVal: any[] | string) => {
    const ids = newVal ? (Array.isArray(newVal) ? newVal : newVal.split(',')) : []
    changeStatus(treeData.value, ids, true)
    changeStatus(filterTreeData.value, ids, true)
  },
  { immediate: true }
)

// 搜索完成返回顶部
function goTop() {
  scrollTop.val = 10
  nextTick(() => {
    scrollTop.value = 0
  })
}

// 处理搜索
function handleSearch(isClear = false) {
  resetClearTimerList()
  if (isClear) {
    // 点击清空按钮并且设置清空按钮会重置搜索
    if (props.clearResetSearch) {
      renderTree(treeData.value)
    }
  } else {
    renderTree(searchValue(searchStr.value, treeData.value))
  }
  goTop()
  uni.hideKeyboard()
}

// 具体搜索方法
function searchValue(str: any, arr: any[]) {
  const res: any = []
  arr.forEach((item) => {
    if (item.visible) {
      if (item[props.dataLabel].toString().toLowerCase().indexOf(str.toLowerCase()) > -1) {
        res.push(item)
      } else {
        if (item[props.dataChildren]?.length) {
          const data = searchValue(str, item[props.dataChildren])
          if (data?.length) {
            if (str && !item.showChildren && item[props.dataChildren]?.length) {
              item.showChildren = true
            }
            res.push({
              ...item,
              [props.dataChildren]: data
            })
          }
        }
      }
    }
  })
  return res
}

// 打开弹窗
async function open() {
  // disaled模式下禁止打开弹窗
  if (props.disabled) return
  showPopup.value = true
  popup.value.open()
  renderTree(treeData.value)
}

// 关闭弹窗
function close() {
  popup.value.close()
}

// 弹窗状态变化 包括点击回显框和遮罩
function change(data: any) {
  if (!data.show) {
    resetClearTimerList()
    searchStr.value = ''
    showPopup.value = false
  }
  emits('change', data)
}

// 点击遮罩
function maskClick() {
  emits('maskClick')
}

// 初始化数据
function initData(arr: any[], parentVisible?: undefined | boolean) {
  if (!Array.isArray(arr)) return []
  const res = []

  for (let i = 0; i < arr.length; i++) {
    const obj: any = {
      [props.dataLabel]: arr[i][props.dataLabel],
      [props.dataValue]: arr[i][props.dataValue]
    }

    obj.checked = selectList.value.includes(arr[i][props.dataValue].toString())

    obj.disabled = Boolean(arr[i].disabled)

    //半选
    obj.partChecked = Boolean(arr[i].partChecked === undefined ? false : arr[i].partChecked)

    const parentVisibleState = parentVisible === undefined ? true : parentVisible
    const curVisibleState = arr[i].visible === undefined ? true : Boolean(arr[i].visible)
    if (parentVisibleState === curVisibleState) {
      obj.visible = parentVisibleState
    } else if (!parentVisibleState || !curVisibleState) {
      obj.visible = false
    } else {
      obj.visible = true
    }

    obj.showChildren =
      'showChildren' in arr[i] && arr[i].showChildren != undefined ? arr[i].showChildren : props.showChildren

    if (arr[i].visible && !arr[i].disabled && !arr[i].checked) {
      isSelectedAll.value = false
    }

    if (arr[i][props.dataChildren]?.length) {
      obj[props.dataChildren] = initData(arr[i][props.dataChildren], obj.visible)
    }

    res.push(obj)
  }

  return res
}

function addNode(node: any, children: any[]) {
  getTruthNode(node)[props.dataChildren] = children
  handleHideChildren(node)
}

// 根据id展示内容
function getselectedInfo(ids: any[] | string) {
  if (!ids.length) return []
  const res = []
  const arr = treeData.value.length ? [...treeData.value] : [...props.listData]
  const data = typeof ids === 'string' ? ids.split(',') : ids.map((item) => item.toString())

  while (arr.length) {
    const item = arr.shift()
    if (data.includes(item[props.dataValue].toString())) {
      res.push({
        ...item,
        [props.dataLabel]: item[props.dataLabel],
        [props.dataValue]: item[props.dataValue].toString()
      })
    }
    if (item[props.dataChildren]?.length) {
      arr.push(...item[props.dataChildren])
    }
  }

  emits('select-change', res)

  return res
}

// 中断懒加载
function resetClearTimerList() {
  const list = [...clearTimerList.value]
  clearTimerList.value = []
  list.forEach((fn) => fn())
}

// 懒加载
function renderTree(arr: any[]) {
  const pagingArr = paging(arr)
  filterTreeData.value = pagingArr?.[0] || []
  lazyRenderList(pagingArr, 1)
}

// 懒加载具体逻辑
function lazyRenderList(arr: any[], startIndex: number) {
  for (let i = startIndex; i < arr.length; i++) {
    let timer: any = null
    timer = setTimeout(() => {
      filterTreeData.value.push(...arr[i])
    }, i * 500)
    clearTimerList.push(() => clearTimeout(timer))
  }
}

// 根据 dataValue 找节点
function changeStatus(list: any[], ids: any[] | string, checkedState: boolean) {
  const arr = [...list]
  const data = typeof ids === 'string' ? ids.split(',') : ids.map((item) => item.toString())

  while (arr.length) {
    const item = arr.shift()

    if (data.includes(item[props.dataValue].toString())) {
      item.checked = checkedState
    } else {
      item.checked = !checkedState
    }

    if (!item.checked) {
      isSelectedAll.value = false
    }

    if (item[props.dataChildren]?.length) {
      arr.push(...item[props.dataChildren])
    }
  }

  return null
}

// 移除选项
function removeSelectedItem(id: string | number) {
  changeStatus(treeData.value, [id], false)
  const emitData = selectList.value.filter((item: any) => item !== id)
  emits('update:modelValue', isString(props.modelValue) ? emitData.join(',') : emitData)
}

// 获取treeData中对应数据
function getTruthNode(node: any) {
  const arr = [...treeData.value]
  while (arr.length) {
    const item = arr.shift()
    if (item[props.dataValue] === node[props.dataValue]) {
      return item
    }
    if (item[props.dataChildren]?.length) {
      arr.push(...item[props.dataChildren])
    }
  }
  return null
}
// 获取 filterTreeData 中对应数据
function getFilterTreeNode(node: any) {
  const arr = [...filterTreeData.value]
  while (arr.length) {
    const item = arr.shift()
    if (item[props.dataValue] === node[props.dataValue]) {
      return item
    }
    if (item[props.dataChildren]?.length) {
      arr.push(...item[props.dataChildren])
    }
  }
  return null
}
// 获取某个节点所有子元素
function getChildren(node: any) {
  if (!node[props.dataChildren]?.length) return []
  const res = node[props.dataChildren].reduce((pre: any, val: any) => {
    if (val.visible) {
      return [...pre, val]
    }
    return pre
  }, [])
  for (let i = 0; i < node[props.dataChildren].length; i++) {
    res.push(...getChildren(node[props.dataChildren][i]))
  }
  return res
}
// 获取某个节点所有父元素
function getParentNode(target: any, arr: any[]) {
  let res: any[] = []

  for (let i = 0; i < arr.length; i++) {
    if (arr[i][props.dataValue] === target[props.dataValue]) {
      return true
    }

    if (arr[i][props.dataChildren]?.length) {
      const childRes = getParentNode(target, arr[i][props.dataChildren])
      if (typeof childRes === 'boolean' && childRes) {
        res = [arr[i]]
      } else if (Array.isArray(childRes) && childRes.length) {
        res = [...childRes, arr[i]]
      }
    }
  }

  return res
}
// 点击checkbox
function handleNodeClick(data: any) {
  const node = getTruthNode(data)
  node.checked = !node.checked
  // 如果是单选不考虑其他情况
  if (!props.mutiple) {
    let emitData: any[] = []
    if (node.checked) {
      emitData = [node[props.dataValue].toString()]
    }
    emits('update:modelValue', isString(props.modelValue) ? emitData.join(',') : emitData)
  } else {
    // 多选情况
    if (!props.linkage) {
      // 不需要联动
      let emitData = null
      if (node.checked) {
        emitData = Array.from(new Set([...selectList.value, node[props.dataValue].toString()]))
      } else {
        emitData = selectList.value.filter((id: number | string) => id !== node[props.dataValue].toString())
      }
      emits('update:modelValue', isString(props.modelValue) ? emitData.join(',') : emitData)
    } else {
      // 需要联动
      let emitData = [...selectList.value]
      const parentNodes: any = getParentNode(node, treeData.value)
      const childrenVal = getChildren(node).filter((item: any) => !item.disabled)
      if (node.checked) {
        // 选中
        emitData = Array.from(new Set([...emitData, node[props.dataValue].toString()]))
        if (childrenVal.length) {
          emitData = Array.from(
            new Set([...emitData, ...childrenVal.map((item: any) => item[props.dataValue].toString())])
          )
        }
        if (parentNodes.length) {
          // 有父元素 如果父元素下所有子元素全部选中，选中父元素
          while (parentNodes.length) {
            const item = parentNodes.shift()
            if (!item.disabled) {
              const allChecked = item[props.dataChildren]
                .filter((node: any) => node.visible && !node.disabled)
                .every((node: any) => node.checked)
              if (allChecked) {
                item.checked = true
                item.partChecked = false
                emitData = Array.from(new Set([...emitData, item[props.dataValue].toString()]))
              } else {
                break
              }
            }
          }
        }
      } else {
        // 取消选中
        emitData = emitData.filter((id) => id !== node[props.dataValue].toString())
        node.partChecked = false
        if (childrenVal.length) {
          // 取消选中全部子节点
          childrenVal.forEach((childNode: any) => {
            emitData = emitData.filter((id) => id !== childNode[props.dataValue].toString())
          })
        }
        if (parentNodes.length) {
          parentNodes.forEach((parentNode: any) => {
            emitData = emitData.filter((id) => id !== parentNode[props.dataValue].toString())
          })
          parentNodes[parentNodes.length - 1].partChecked = emitData.length === 0 ? false : true
        }
      }
      emits('update:modelValue', isString(props.modelValue) ? emitData.join(',') : emitData)
    }
  }
}
// 点击名称折叠或展开
function handleHideChildren(node: any) {
  const status = !node.showChildren
  getTruthNode(node).showChildren = status
  getFilterTreeNode(node).showChildren = status
}
// 全部选中
function handleSelectAll() {
  if (props.isSelectedAll) return
  isSelectedAll.value = true
  if (!props.mutiple) {
    uni.showToast({
      title: '单选模式下不能全选',
      icon: 'none',
      duration: 1000
    })
    return
  }
  let emitData: any[] = []
  treeData.value.forEach((item: any) => {
    if (item.visible && !item.disabled) {
      emitData = Array.from(new Set([...emitData, item[props.dataValue].toString()]))
      if (item[props.dataChildren]?.length) {
        emitData = Array.from(
          new Set([
            ...emitData,
            ...getChildren(item)
              .filter((item: any) => !item.disabled)
              .map((item: any) => item[props.dataValue].toString())
          ])
        )
      }
    }
  })
  emits('update:modelValue', isString(props.modelValue) ? emitData.join(',') : emitData)
}
// 清空选项
function clearSelectList() {
  if (props.disabled) return
  emits('update:modelValue', isString(props.modelValue) ? '' : [])
}
</script>

<style lang="scss" scoped>
.select-list {
  padding-left: 10px;
  min-height: 35px;
  border: 1px solid #e5e5e5;
  border-radius: 4px;
  display: flex;
  justify-content: space-between;
  align-items: center;

  &.active {
    padding: 4px 0 4px 10px;
  }

  .left {
    flex: 1;

    .select-items {
      display: flex;
      flex-wrap: wrap;
    }

    .select-item {
      margin: 4px 10px 4px 0;
      padding: 4px 5px;
      max-width: auto;
      height: auto;
      background-color: #eaeaea;
      border-radius: 4px;
      color: #333;
      display: flex;
      align-items: center;

      .name {
        flex: 1;
        padding-right: 10px;
        font-size: 14px;
      }

      .close {
        width: 18px;
        height: 18px;
        display: flex;
        justify-content: center;
        align-items: center;
        overflow: hidden;
      }
    }
  }

  .right {
    margin-right: 5px;
    display: flex;
    justify-content: flex-end;
    align-items: center;
  }

  &.disabled {
    background-color: #f5f7fa;

    .left {
      .select-item {
        .name {
          padding: 0;
        }
      }
    }
  }
}

.popup-content {
  flex: 1;
  background-color: #fff;
  border-top-left-radius: 20px;
  border-top-right-radius: 20px;
  display: flex;
  flex-direction: column;

  .title {
    padding: 8px 3rem;
    border-bottom: 1px solid $uni-border-color;
    font-size: 14px;
    display: flex;
    justify-content: space-between;
    position: relative;

    .left {
      position: absolute;
      left: 10px;
    }

    .center {
      flex: 1;
      text-align: center;
    }

    .right {
      position: absolute;
      right: 10px;
    }
  }

  .search-box {
    margin: 8px 10px 0;
    background-color: #fff;
    display: flex;
    align-items: center;

    .search-btn {
      margin-left: 10px;
      height: 35px;
      line-height: 35px;
    }
  }

  .select-content {
    margin: 8px 10px;
    flex: 1;
    overflow: hidden;
    position: relative;
  }

  .scroll-view-box {
    touch-action: none;
    flex: 1;
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
  }

  .sentry {
    height: 48px;
  }
}

.no-data {
  width: auto;
  color: #999;
  font-size: 12px;
}

.no-data.center {
  text-align: center;
}
</style>
