<template>
  <div
    class="q-tabs flex no-wrap"
    :class="classes"
  >
    <div
      class="q-tabs-head row"
      ref="tabs"
      :class="innerClasses"
    >
      <div ref="scroller" class="q-tabs-scroller row no-wrap">
        <slot name="title"></slot>
        <div
          v-if="$q.theme !== 'ios'"
          class="relative-position self-stretch q-tabs-global-bar-container"
          :class="posbarClasses"
        >
          <div
            ref="posbar"
            class="q-tabs-bar q-tabs-global-bar"
            @transitionend="__updatePosbarTransition"
          ></div>
        </div>
      </div>
      <div
        ref="leftScroll"
        class="row flex-center q-tabs-left-scroll"
        @mousedown="__animScrollTo(0)"
        @touchstart="__animScrollTo(0)"
        @mouseup="__stopAnimScroll"
        @mouseleave="__stopAnimScroll"
        @touchend="__stopAnimScroll"
      >
        <q-icon :name="$q.icon.tabs.left"></q-icon>
      </div>
      <div
        ref="rightScroll"
        class="row flex-center q-tabs-right-scroll"
        @mousedown="__animScrollTo(9999)"
        @touchstart="__animScrollTo(9999)"
        @mouseup="__stopAnimScroll"
        @mouseleave="__stopAnimScroll"
        @touchend="__stopAnimScroll"
      >
        <q-icon :name="$q.icon.tabs.right"></q-icon>
      </div>
    </div>

    <div class="q-tabs-panes">
      <slot></slot>
    </div>
  </div>
</template>

<script>
import { width, css, cssTransform } from '../../utils/dom'
import { debounce } from '../../utils/debounce'
import { QIcon } from '../icon'
import { listenOpts } from '../../utils/event'

const
  scrollNavigationSpeed = 5, // in pixels
  debounceDelay = 50 // in ms

export default {
  name: 'QTabs',
  provide () {
    return {
      data: this.data,
      selectTab: this.selectTab,
      selectTabRouter: this.selectTabRouter
    }
  },
  components: {
    QIcon
  },
  props: {
    value: String,
    align: {
      type: String,
      default: __THEME__ === 'ios' ? 'center' : 'left',
      validator: v => ['left', 'center', 'right', 'justify'].includes(v)
    },
    position: {
      type: String,
      default: 'top',
      validator: v => ['top', 'bottom'].includes(v)
    },
    color: {
      type: String,
      default: 'primary'
    },
    textColor: String,
    inverted: Boolean,
    twoLines: Boolean,
    noPaneBorder: Boolean,
    glossy: Boolean
  },
  data () {
    return {
      currentEl: null,
      posbar: {
        width: 0,
        left: 0
      },
      data: {
        highlight: true,
        tabName: this.value || '',
        color: this.color,
        textColor: this.textColor,
        inverted: this.inverted
      }
    }
  },
  watch: {
    value (name) {
      this.selectTab(name)
    },
    color (v) {
      this.data.color = v
    },
    textColor (v) {
      this.data.textColor = v
    },
    inverted (v) {
      this.data.inverted = v
    }
  },
  computed: {
    classes () {
      return [
        `q-tabs-position-${this.position}`,
        `q-tabs-${this.inverted ? 'inverted' : 'normal'}`,
        this.noPaneBorder ? 'q-tabs-no-pane-border' : '',
        this.twoLines ? 'q-tabs-two-lines' : ''
      ]
    },
    innerClasses () {
      const cls = [ `q-tabs-align-${this.align}` ]
      this.glossy && cls.push('glossy')
      if (this.inverted) {
        cls.push(`text-${this.textColor || this.color}`)
      }
      else {
        cls.push(`bg-${this.color}`)
        cls.push(`text-${this.textColor || 'white'}`)
      }
      return cls
    },
    posbarClasses () {
      const cls = []
      this.inverted && cls.push(`text-${this.textColor || this.color}`)
      this.data.highlight && cls.push('highlight')
      return cls
    }
  },
  methods: {
    selectTab (value) {
      if (this.data.tabName === value) {
        return
      }

      this.data.tabName = value
      this.$emit('select', value)

      this.$emit('input', value)
      this.$nextTick(() => {
        if (JSON.stringify(value) !== JSON.stringify(this.value)) {
          this.$emit('change', value)
        }
      })

      const el = this.__getTabElByName(value)

      if (el) {
        this.__scrollToTab(el)
      }

      if (__THEME__ !== 'ios') {
        this.currentEl = el
        this.__repositionBar()
      }
    },
    selectTabRouter (params) {
      const
        { value, selectable, exact, selected, priority } = params,
        first = !this.buffer.length,
        existingIndex = first ? -1 : this.buffer.findIndex(t => t.value === value)

      if (existingIndex > -1) {
        const buffer = this.buffer[existingIndex]
        exact && (buffer.exact = exact)
        selectable && (buffer.selectable = selectable)
        selected && (buffer.selected = selected)
        priority && (buffer.priority = priority)
      }
      else {
        this.buffer.push(params)
      }

      if (first) {
        this.bufferTimer = setTimeout(() => {
          let tab = this.buffer.find(t => t.exact && t.selected) ||
            this.buffer.find(t => t.selectable && t.selected) ||
            this.buffer.find(t => t.exact) ||
            this.buffer.filter(t => t.selectable).sort((t1, t2) => t2.priority - t1.priority)[0] ||
            this.buffer[0]

          this.buffer.length = 0
          this.selectTab(tab.value)
        }, 100)
      }
    },
    __repositionBar () {
      clearTimeout(this.timer)

      let needsUpdate = false
      const
        ref = this.$refs.posbar,
        el = this.currentEl

      if (this.data.highlight !== false) {
        this.data.highlight = false
        needsUpdate = true
      }

      if (!el) {
        this.finalPosbar = {width: 0, left: 0}
        this.__setPositionBar(0, 0)
        return
      }

      const offsetReference = ref.parentNode.offsetLeft

      if (needsUpdate && this.oldEl) {
        this.__setPositionBar(
          this.oldEl.getBoundingClientRect().width,
          this.oldEl.offsetLeft - offsetReference
        )
      }

      this.timer = setTimeout(() => {
        const
          width = el.getBoundingClientRect().width,
          left = el.offsetLeft - offsetReference

        ref.classList.remove('contract')
        this.oldEl = el
        this.finalPosbar = {width, left}
        this.__setPositionBar(
          this.posbar.left < left
            ? left + width - this.posbar.left
            : this.posbar.left + this.posbar.width - left,
          this.posbar.left < left
            ? this.posbar.left
            : left
        )
      }, 20)
    },
    __setPositionBar (width = 0, left = 0) {
      if (this.posbar.width === width && this.posbar.left === left) {
        this.__updatePosbarTransition()
        return
      }
      this.posbar = {width, left}
      const xPos = this.$q.i18n.rtl
        ? left + width
        : left
      css(this.$refs.posbar, cssTransform(`translateX(${xPos}px) scaleX(${width})`))
    },
    __updatePosbarTransition () {
      if (
        this.finalPosbar.width === this.posbar.width &&
        this.finalPosbar.left === this.posbar.left
      ) {
        this.posbar = {}
        if (this.data.highlight !== true) {
          this.data.highlight = true
        }
        return
      }

      this.$refs.posbar.classList.add('contract')
      this.__setPositionBar(this.finalPosbar.width, this.finalPosbar.left)
    },
    __redraw () {
      if (!this.$q.platform.is.desktop) {
        return
      }
      this.scrollerWidth = width(this.$refs.scroller)
      if (this.scrollerWidth === 0 && this.$refs.scroller.scrollWidth === 0) {
        return
      }
      if (this.scrollerWidth + 5 < this.$refs.scroller.scrollWidth) {
        this.$refs.tabs.classList.add('scrollable')
        this.scrollable = true
        this.__updateScrollIndicator()
      }
      else {
        this.$refs.tabs.classList.remove('scrollable')
        this.scrollable = false
      }
    },
    __updateScrollIndicator () {
      if (!this.$q.platform.is.desktop || !this.scrollable) {
        return
      }
      let action = this.$refs.scroller.scrollLeft + width(this.$refs.scroller) + 5 >= this.$refs.scroller.scrollWidth ? 'add' : 'remove'
      this.$refs.leftScroll.classList[this.$refs.scroller.scrollLeft <= 0 ? 'add' : 'remove']('disabled')
      this.$refs.rightScroll.classList[action]('disabled')
    },
    __getTabElByName (value) {
      const tab = this.$children.find(child => child.name === value && child.$el && child.$el.nodeType === 1)
      if (tab) {
        return tab.$el
      }
    },
    __findTabAndScroll (name, noAnimation) {
      setTimeout(() => {
        this.__scrollToTab(this.__getTabElByName(name), noAnimation)
      }, debounceDelay * 4)
    },
    __scrollToTab (tab, noAnimation) {
      if (!tab || !this.scrollable) {
        return
      }

      let
        contentRect = this.$refs.scroller.getBoundingClientRect(),
        rect = tab.getBoundingClientRect(),
        tabWidth = rect.width,
        offset = rect.left - contentRect.left

      if (offset < 0) {
        if (noAnimation) {
          this.$refs.scroller.scrollLeft += offset
        }
        else {
          this.__animScrollTo(this.$refs.scroller.scrollLeft + offset)
        }
        return
      }

      offset += tabWidth - this.$refs.scroller.offsetWidth
      if (offset > 0) {
        if (noAnimation) {
          this.$refs.scroller.scrollLeft += offset
        }
        else {
          this.__animScrollTo(this.$refs.scroller.scrollLeft + offset)
        }
      }
    },
    __animScrollTo (value) {
      this.__stopAnimScroll()
      this.__scrollTowards(value)

      this.scrollTimer = setInterval(() => {
        if (this.__scrollTowards(value)) {
          this.__stopAnimScroll()
        }
      }, 5)
    },
    __stopAnimScroll () {
      clearInterval(this.scrollTimer)
    },
    __scrollTowards (value) {
      let
        scrollPosition = this.$refs.scroller.scrollLeft,
        direction = value < scrollPosition ? -1 : 1,
        done = false

      scrollPosition += direction * scrollNavigationSpeed
      if (scrollPosition < 0) {
        done = true
        scrollPosition = 0
      }
      else if (
        (direction === -1 && scrollPosition <= value) ||
        (direction === 1 && scrollPosition >= value)
      ) {
        done = true
        scrollPosition = value
      }

      this.$refs.scroller.scrollLeft = scrollPosition
      return done
    }
  },
  created () {
    this.timer = null
    this.scrollTimer = null
    this.bufferTimer = null
    this.buffer = []
    this.scrollable = !this.$q.platform.is.desktop

    // debounce some costly methods;
    // debouncing here because debounce needs to be per instance
    this.__redraw = debounce(this.__redraw, debounceDelay)
    this.__updateScrollIndicator = debounce(this.__updateScrollIndicator, debounceDelay)
  },
  mounted () {
    this.$nextTick(() => {
      if (!this.$refs.scroller) {
        return
      }
      this.$refs.scroller.addEventListener('scroll', this.__updateScrollIndicator, listenOpts.passive)
      window.addEventListener('resize', this.__redraw, listenOpts.passive)

      if (this.data.tabName !== '' && this.value) {
        this.selectTab(this.value)
      }

      this.__redraw()
      this.__findTabAndScroll(this.data.tabName, true)
    })
  },
  beforeDestroy () {
    clearTimeout(this.timer)
    clearTimeout(this.bufferTimer)
    this.__stopAnimScroll()
    this.$refs.scroller.removeEventListener('scroll', this.__updateScrollIndicator, listenOpts.passive)
    window.removeEventListener('resize', this.__redraw, listenOpts.passive)
    this.__redraw.cancel()
    this.__updateScrollIndicator.cancel()
  }
}
</script>
