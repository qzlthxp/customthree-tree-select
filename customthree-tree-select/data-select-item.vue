<template>
  <view
    class="custom-tree-select-content"
    :class="{
      border: border && node[dataChildren] && node[dataChildren].length && node.showChildren
    }"
    :style="{
      marginLeft: `${level * 5 + 5}px`
    }"
  >
    <view v-if="node.visible" class="custom-tree-select-item">
      <view class="item-content">
        <view class="left" @click.stop="handleNameClick(node)">
          <view class="icon-group">
            <view
              v-if="node[dataChildren] && node[dataChildren].length"
              :class="['icon', { active: node.showChildren }]"
            >
              <uni-icons type="right" size="14" color="#333"></uni-icons>
            </view>
            <view v-else class="icon smallcircle-filled"></view>
          </view>
          <view v-if="loadingArr.includes(node[props.dataValue].toString())" class="loading-icon-box">
            <uni-icons class="loading-icon" type="spinner-cycle" size="14" color="#333"></uni-icons>
          </view>
          <view class="name" :style="node.disabled ? 'color: #999' : ''">
            <text>{{ node[dataLabel] }}</text>
          </view>
        </view>
        <checkbox
          v-if="
            (choseParent ||
              (!choseParent && !node[dataChildren]) ||
              (!choseParent && node[dataChildren] && !node[dataChildren].length)) &&
            !node.partChecked
          "
          :disabled="node.disabled"
          :value="node[dataValue].toString()"
          :checked="node.checked"
          @click.stop="nodeClick(node)"
        />
        <view class="check-section" v-else>
          <view class="check-content-section"> </view>
        </view>
      </view>
    </view>
    <view v-if="node.showChildren && node[dataChildren] && node[dataChildren].length">
      <data-select-item
        v-for="item in listData"
        :key="item[dataValue]"
        :node="item"
        :dataLabel="dataLabel"
        :dataValue="dataValue"
        :dataChildren="dataChildren"
        :choseParent="choseParent"
        :lazyLoadChildren="lazyLoadChildren"
        :border="border"
        :level="level + 1"
      ></data-select-item>
    </view>
  </view>
</template>
<script lang="ts" setup>
import dataSelectItem from './data-select-item.vue'
import { paging } from './utils'
import { ref, inject, watchEffect } from 'vue'

const { nodeClick, nameClick, loadNode, initData, addNode } = inject('nodeFn')

const props = defineProps({
  node: {
    type: Object,
    default: () => ({})
  },
  choseParent: {
    type: Boolean,
    default: true
  },
  dataLabel: {
    type: String,
    default: 'name'
  },
  dataValue: {
    type: String,
    default: 'value'
  },
  dataChildren: {
    type: String,
    default: 'children'
  },
  border: {
    type: Boolean,
    default: false
  },
  lazyLoadChildren: {
    type: Boolean,
    default: false
  },
  level: {
    type: Number,
    default: 0
  }
})

const listData = ref([])
const clearTimerList = ref([])
const loadingArr = ref([])

watchEffect(() => {
  if (props.node.showChildren && props.node[props.dataChildren] && props.node[props.dataChildren].length) {
    resetClearTimerList()
    renderTree(props.node[props.dataChildren])
  }
})

// 中断懒加载
function resetClearTimerList() {
  const list = [...clearTimerList.value]
  clearTimerList.value = []
  list.forEach((fn) => fn())
}

// 懒加载
function renderTree(arr: any[]) {
  const pagingArr = paging(arr)
  listData.value = pagingArr?.[0] || []
  lazyRenderList(pagingArr, 1)
}

// 懒加载具体逻辑
function lazyRenderList(arr: any[], startIndex: number) {
  for (let i = startIndex; i < arr.length; i++) {
    let timer: any = null
    timer = setTimeout(() => {
      listData.value.push(...arr[i])
    }, i * 500)
    clearTimerList.push(() => clearTimeout(timer))
  }
}

// 点击名称懒加载子元素
function handleNameClick(node: any) {
  if (node.disabled || !node.visible) return

  if (!node[props.dataChildren]?.length && props.lazyLoadChildren) {
    loadingArr.value.push(node[props.dataValue].toString())

    loadNode(node)
      .then((res: any) => {
        addNode(node, initData(res, node.visible))
      })
      .finally(() => {
        loadingArr.value = []
      })
  } else {
    nameClick(node)
  }
}
</script>

<style lang="scss" scoped>
.custom-tree-select-content {
  &.border {
    border-left: 1px dashed #c8c7cc;
  }

  ::v-deep .uni-checkbox-input {
    margin: 0 !important;
  }

  .item-content {
    margin: 0 0 16px;
    display: flex;
    justify-content: space-between;
    align-items: center;

    &:first-child {
      margin-top: 0;
    }

    .left {
      padding-right: 10px;
      flex: 1;
      display: flex;
      align-items: center;
      position: relative;

      .icon-group {
        position: absolute;
        transform: translateX(-50%);
        background-color: #fff;

        .icon {
          transition: 0.2s ease;

          &.active {
            transform: rotate(90deg);
          }

          &.smallcircle-filled {
            width: 5px;
            height: 5px;
            border-radius: 5px;
            background-color: #333;
          }
        }
      }

      .loading-icon-box {
        padding: 0 0 0 8px;

        .loading-icon {
          display: inline-block;
          animation: rotating infinite 0.4s linear;
        }
      }
    }

    .name {
      padding-left: 8px;
      flex: 1;
      height: auto;
      word-break: break-all;
    }
  }
}
.check-section {
  padding: 4px;
  border: 1px solid #007aff;
}
.check-content-section {
  width: 14px;
  height: 14px;
  background: #007aff;
}

@keyframes rotating {
  from {
    transform: rotate(0);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>
