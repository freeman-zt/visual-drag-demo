<template>
  <div
    id="editor"
    class="editor"
    :class="{ edit: isEdit }"
    :style="{
      ...getCanvasStyle(canvasStyleData),
      width: changeStyleWithScale(canvasStyleData.width) + 'px',
      height: changeStyleWithScale(canvasStyleData.height) + 'px',
    }"
    @contextmenu="handleContextMenu"
    @mousedown="handleMouseDown"
  >
    <!-- 网格线 -->
    <Grid :is-dark-mode="isDarkMode" />

    <!--页面组件列表展示-->
    <Shape
      v-for="(item, index) in componentData"
      :key="item.id"
      :default-style="item.style"
      :style="getShapeStyle(item.style)"
      :active="item.id === (curComponent || {}).id"
      :element="item"
      :index="index"
      :class="{ lock: item.isLock }"
    >
      <component
        :is="item.component"
        v-if="item.component.startsWith('SVG')"
        :id="'component' + item.id"
        :style="getSVGStyle(item.style)"
        class="component"
        :prop-value="item.propValue"
        :element="item"
        :request="item.request"
      />

      <component
        :is="item.component"
        v-else-if="item.component != 'VText'"
        :id="'component' + item.id"
        class="component"
        :style="getComponentStyle(item.style)"
        :prop-value="item.propValue"
        :element="item"
        :request="item.request"
      />

      <component
        :is="item.component"
        v-else
        :id="'component' + item.id"
        class="component"
        :style="getComponentStyle(item.style)"
        :prop-value="item.propValue"
        :element="item"
        :request="item.request"
        @input="handleInput"
      />
    </Shape>
    <!-- 右击菜单 -->
    <ContextMenu />
    <!-- 标线 -->
    <MarkLine />
    <!-- 选中区域 -->
    <Area v-show="isShowArea" :start="start" :width="width" :height="height" />
    <!-- 选择的组件包裹层 -->
    <GroupComponentBox v-if="areaData?.components?.length" v-bind="areaData.style" />
  </div>
</template>

<script>
import { mapState } from 'vuex'
import Shape from './Shape'
import { getStyle, getComponentRotatedStyle, getShapeStyle, getSVGStyle, getCanvasStyle } from '@/utils/style'
import { $, isPreventDrop } from '@/utils/utils'
import ContextMenu from './ContextMenu'
import MarkLine from './MarkLine'
import Area from './Area'
import GroupComponentBox from './GroupComponentBox'
import eventBus from '@/utils/eventBus'
import Grid from './Grid'
import { changeStyleWithScale } from '@/utils/translate'

export default {
  components: { Shape, ContextMenu, MarkLine, Area, Grid, GroupComponentBox },
  props: {
    isEdit: {
      type: Boolean,
      default: true,
    },
  },
  data() {
    return {
      editorX: 0,
      editorY: 0,
      start: {
        // 选中区域的起点
        x: 0,
        y: 0,
      },
      width: 0,
      height: 0,
      isShowArea: false,
      svgFilterAttrs: ['width', 'height', 'top', 'left', 'rotate'],
    }
  },
  computed: mapState(['componentData', 'curComponent', 'canvasStyleData', 'editor', 'isDarkMode', 'areaData']),
  mounted() {
    // 获取编辑器元素
    this.$store.commit('getEditor')

    eventBus.$on('hideArea', () => {
      this.hideArea()
    })
  },
  methods: {
    getShapeStyle,
    getCanvasStyle,
    changeStyleWithScale,

    handleMouseDown(e) {
      // 如果没有选中组件 在画布上点击时需要调用 e.preventDefault() 防止触发 drop 事件
      if (!this.curComponent || isPreventDrop(this.curComponent.component)) {
        e.preventDefault()
      }

      this.hideArea()

      // 获取编辑器的位移信息，每次点击时都需要获取一次。主要是为了方便开发时调试用。
      const rectInfo = this.editor.getBoundingClientRect()
      this.editorX = rectInfo.x
      this.editorY = rectInfo.y

      const startX = e.clientX
      const startY = e.clientY
      this.start.x = startX - this.editorX
      this.start.y = startY - this.editorY
      // 展示选中区域
      this.isShowArea = true

      const move = (moveEvent) => {
        this.width = Math.abs(moveEvent.clientX - startX)
        this.height = Math.abs(moveEvent.clientY - startY)
        if (moveEvent.clientX < startX) {
          this.start.x = moveEvent.clientX - this.editorX
        }

        if (moveEvent.clientY < startY) {
          this.start.y = moveEvent.clientY - this.editorY
        }
      }

      const up = (e) => {
        document.removeEventListener('mousemove', move)
        document.removeEventListener('mouseup', up)

        if (e.clientX == startX && e.clientY == startY) {
          this.hideArea()
          return
        }
        this.createGroup()
      }

      document.addEventListener('mousemove', move)
      document.addEventListener('mouseup', up)
    },

    hideArea() {
      this.isShowArea = false
      this.width = 0
      this.height = 0

      this.$store.commit('setAreaData', {
        style: {
          left: 0,
          top: 0,
          width: 0,
          height: 0,
        },
        components: [],
      })
    },

    createGroup() {
      // 获取选中区域的组件数据
      const areaData = this.getSelectArea()
      if (areaData.length <= 1) {
        this.hideArea()
        return
      }

      // 根据选中区域和区域中每个组件的位移信息来创建 Group 组件
      // 要遍历选择区域的每个组件，获取它们的 left top right bottom 信息来进行比较
      let top = Infinity,
        left = Infinity
      let right = -Infinity,
        bottom = -Infinity
      areaData.forEach((component) => {
        let style = {}
        if (component.component == 'Group') {
          component.propValue.forEach((item) => {
            const rectInfo = $(`#component${item.id}`).getBoundingClientRect()
            style.left = rectInfo.left - this.editorX
            style.top = rectInfo.top - this.editorY
            style.right = rectInfo.right - this.editorX
            style.bottom = rectInfo.bottom - this.editorY

            if (style.left < left) left = style.left
            if (style.top < top) top = style.top
            if (style.right > right) right = style.right
            if (style.bottom > bottom) bottom = style.bottom
          })
        } else {
          style = getComponentRotatedStyle(component.style)
        }

        if (style.left < left) left = style.left
        if (style.top < top) top = style.top
        if (style.right > right) right = style.right
        if (style.bottom > bottom) bottom = style.bottom
      })

      this.start.x = left
      this.start.y = top
      this.width = right - left
      this.height = bottom - top

      // 设置选中区域位移大小信息和区域内的组件数据
      this.$store.commit('setAreaData', {
        style: {
          left,
          top,
          width: this.width,
          height: this.height,
        },
        components: areaData,
      })
      this.isShowArea = false // 关闭选框
    },

    getSelectArea() {
      const result = []
      // 区域起点坐标
      const { x, y } = this.start
      // 计算所有的组件数据，判断是否在选中区域内
      this.componentData.forEach((component) => {
        if (component.isLock) return

        const { left, top, width, height } = getComponentRotatedStyle(component.style)
        if (x <= left && y <= top && left + width <= x + this.width && top + height <= y + this.height) {
          result.push(component)
        }
      })

      // 返回在选中区域内的所有组件
      return result
    },

    handleContextMenu(e) {
      e.stopPropagation()
      e.preventDefault()

      // 获取旋转后的实际位置
      // const targetRect = e.target.getBoundingClientRect();
      let editor = document.querySelector('.editor')

      let editorRect = editor.getBoundingClientRect()

      // 计算相对编辑器的精确位置（考虑旋转）
      let left = e.clientX - editorRect.left
      let top = e.clientY - editorRect.top

      let adjustedLeft = left > editorRect.width ? editorRect.width : left
      let adjustedTop = top > editorRect.height ? editorRect.height : top
      this.$store.commit('showContextMenu', { top: adjustedTop, left: adjustedLeft })
    },

    getComponentStyle(style) {
      return getStyle(style, this.svgFilterAttrs)
    },

    getSVGStyle(style) {
      return getSVGStyle(style, this.svgFilterAttrs)
    },

    handleInput(element, value) {
      // 根据文本组件高度调整 shape 高度
      this.$store.commit('setShapeStyle', { height: this.getTextareaHeight(element, value) })
    },

    getTextareaHeight(element, text) {
      let { lineHeight, fontSize, height } = element.style
      if (lineHeight === '') {
        lineHeight = 1.5
      }

      const newHeight = (text.split('<br>').length - 1) * lineHeight * (fontSize || this.canvasStyleData.fontSize)
      return height > newHeight ? height : newHeight
    },
  },
}
</script>

<style lang="scss" scoped>
.editor {
  position: relative;
  background: #fff;
  margin: auto;

  .lock {
    opacity: 0.5;

    &:hover {
      cursor: not-allowed;
    }
  }
}

.edit {
  .component {
    outline: none;
    width: 100%;
    height: 100%;
  }
}
</style>
