<template>
  <div class="scroll-wrapper" ref="wrapper">
    <div ref="listWrapper" class="scroll-list-wrapper">
      <div v-if="pullDownRefresh" class="pulldown" ref="pulldown">
        <slot
          name="pulldown"
          :pullDownRefresh="pullDownRefresh"
          :pullDownStyle="pullDownStyle"
          :beforePullDown="beforePullDown"
          :isPullingDown="isPullingDown"
          :bubbleY="bubbleY">
          <div class="pulldown-wrapper" :style="pullDownStyle">
            <div class="before-trigger" v-show="beforePullDown">
              <bubble :y="bubbleY" class="bubble"></bubble>
            </div>
            <div class="after-trigger" v-show="!beforePullDown">
              <div v-show="isPullingDown" class="loading">
                <van-loading type="spinner" size="24px"/>
              </div>
              <div v-show="!isPullingDown" class="pulldown-loaded"><span>{{ refreshTxt }}</span></div>
            </div>
          </div>
        </slot>
      </div>
      <slot></slot>
    </div>
  </div>
</template>

<script>
  import BScroll from '@better-scroll/core'
  import PullDown from '@better-scroll/pull-down'// pulldown 插件为 BetterScroll 扩展下拉刷新的能力
  import Pullup from '@better-scroll/pull-up'// pullup 插件为 BetterScroll 扩展上拉加载的能力
  import ScrollBar from '@better-scroll/scroll-bar'// scrollbar 插件为 BetterScroll 提供了样式美观的滚动条
  import NestedScroll from '@better-scroll/nested-scroll'// 协调 BetterScroll 双层嵌套的滚动行为
  import Bubble from './components/bubble'
  import { getRect } from '../../common/helpers/dom'
  import { camelize } from '../../common/lang/string'

  BScroll.use(PullDown)
  BScroll.use(Pullup)
  BScroll.use(ScrollBar)
  BScroll.use(NestedScroll)

  const DIRECTION_H = 'horizontal'
  const DIRECTION_V = 'vertical'
  const DEFAULT_REFRESH_TXT = 'Refresh success'
  const DEFAULT_STOP_TIME = 600

  // eslint-disable-next-line no-unused-vars
  const EVENT_CLICK = 'click'
  const EVENT_PULLING_DOWN = 'pullingDown'

  const EVENT_SCROLL = 'scroll'
  const EVENT_BEFORE_SCROLL_START = 'beforeScrollStart'
  const EVENT_SCROLL_END = 'scrollEnd'

  const NEST_MODE_NONE = 'none'
  const NEST_MODE_NATIVE = 'native'

  const SCROLL_EVENTS = [EVENT_SCROLL, EVENT_BEFORE_SCROLL_START, EVENT_SCROLL_END]

  const DEFAULT_OPTIONS = {
    observeDOM: true,
    click: true,
    probeType: 1,
    scrollbar: false,
    pullDownRefresh: false,
    pullUpLoad: false
  }

  export default {
    name: 'BaseScroll',
    components: { Bubble },
    provide () {
      return {
        parentScroll: this
      }
    },
    inject: {
      parentScroll: {
        default: null
      }
    },
    props: {
      options: {
        type: Object,
        default () {
          return {}
        }
      },
      // 列表的数据
      data: {
        type: Array,
        default: null
      },
      scrollEvents: {
        type: Array,
        default () {
          return []
        },
        validator (arr) {
          return arr.every((item) => {
            return SCROLL_EVENTS.indexOf(item) !== -1
          })
        }
      },
      direction: {
        type: String,
        default: DIRECTION_V
      },
      nestMode: {
        type: String,
        default: NEST_MODE_NONE
      },
      // 是否派发列表滚动开始的事件
      beforeScroll: {
        type: Boolean,
        default: false
      },
      // 当数据更新后，刷新scroll的延时
      refreshDelay: {
        type: Number,
        default: 20
      }
    },
    data () {
      return {
        beforePullDown: true,
        isPullingDown: false,
        pullUpNoMore: false,
        bubbleY: 0,
        pullDownStyle: '',
        pullDownStop: 40,
        pullDownHeight: 60,
        pullUpHeight: 0
      }
    },
    computed: {
      pullDownRefresh () {
        let pullDownRefresh = this.options.pullDownRefresh
        if (!pullDownRefresh) {
          return pullDownRefresh
        }
        if (pullDownRefresh === true) {
          pullDownRefresh = {}
        }
        return Object.assign({ stop: this.pullDownStop }, pullDownRefresh)
      },
      refreshTxt () {
        const pullDownRefresh = this.pullDownRefresh
        return (pullDownRefresh && pullDownRefresh.txt) || DEFAULT_REFRESH_TXT
      },
      finalScrollEvents () {
        const finalScrollEvents = this.scrollEvents.slice()

        // if (!finalScrollEvents.length) {
        //   this.listenScroll && finalScrollEvents.push(EVENT_SCROLL)
        //   this.listenBeforeScroll && finalScrollEvents.push(EVENT_BEFORE_SCROLL_START)
        // }
        return finalScrollEvents
      },
      needListenScroll () {
        return this.finalScrollEvents.indexOf(EVENT_SCROLL) !== -1 || this.parentScroll
      }
    },
    mounted () {
      // 在 DOM 渲染完毕后初始化 better-scroll
      this.$nextTick(() => {
        this.initScroll()
      })
    },
    methods: {
      initScroll () {
        if (!this.$refs.wrapper) {
          return
        }

        // 最小高度计算
        this._calculateMinHeight()

        const options = Object.assign({}, DEFAULT_OPTIONS, {
          scrollY: this.direction === DIRECTION_V,
          scrollX: this.direction === DIRECTION_H,
          probeType: this.needListenScroll ? 3 : 1,
          scrollbar: this.scrollbar,
          nestedScroll: false
        }, this.options)

        // better-scroll 初始化
        this.bscroll = new BScroll(this.$refs.wrapper, options)

        this.parentScroll && this.nestMode !== NEST_MODE_NONE && this._handleNestScroll()

        this._listenScrollEvents()
        // 上拉
        if (this.pullup) {
          this.bscroll.on('scrollEnd', () => {
            // 滚动到底部
            if (this.bscroll.y <= (this.bscroll.maxScrollY + 50)) {
              // 派发滚动到底部的事件
              this.$emit('pullingUp')
            }
          })
        }

        // 下拉
        if (this.pullDownRefresh) {
          this._onPullDownRefresh()
          this._pullDownRefreshChangeHandler()
        }

        if (this.beforeScroll) {
          this.bscroll.on('beforeScrollStart', () => {
            // 列表滚动前触发
            this.$emit('beforeScroll')
          })
        }
      },
      disable () {
        // 代理 better-scroll 的 disable 方法
        this.bscroll && this.bscroll.disable()
      },
      enable () {
        // 代理 better-scroll 的 enable 方法
        this.bscroll && this.bscroll.enable()
      },
      refresh () {
        // 代理 better-scroll 的 refresh 方法
        this.bscroll && this.bscroll.refresh()
      },
      scrollTo () {
        // 代理 better-scroll 的 scrollTo 方法
        this.bscroll && this.bscroll.scrollTo.apply(this.bscroll, arguments)
      },
      scrollToElement () {
        // 代理 better-scroll 的 scrollToElement 方法
        this.bscroll && this.bscroll.scrollToElement.apply(this.bscroll, arguments)
      },
      async forceUpdate (dirty = false, nomore = false) {
        if (this.isPullDownUpdating) {
          return
        }

        if (this.pullDownRefresh && this.isPullingDown) {
          this.isPullingDown = false
          this.isPullDownUpdating = true
          await this._waitFinishPullDown()
          this.isPullDownUpdating = false
          await this._waitResetPullDown(dirty)
        } else if (this.pullUpLoad && this.isPullUpLoad) {
          this.isPullUpLoad = false
          this.scroll.finishPullUp()
          this.pullUpNoMore = !dirty || nomore
        }

        dirty && this.refresh()
      },
      _listenScrollEvents () {
        this.finalScrollEvents.forEach((event) => {
          this.bscroll.on(camelize(event), (...args) => {
            this.$emit(event, ...args)
          })
        })
      },
      _handleNestScroll () {
        // waiting scroll initial
        this.$nextTick(() => {
          const innerScroll = this.bscroll
          const outerScroll = this.parentScroll.bscroll
          const scrolls = [innerScroll, outerScroll]
          scrolls.forEach((scroll, index, arr) => {
            // scroll ended always enable them
            scroll.on('touchEnd', () => {
              outerScroll.enable()
              innerScroll.enable()
              // when inner scroll reaching boundary, we will disable inner scroll, so when 'touchend' event fire,
              // the inner scroll will not reset initiated within '_end' method in better-scroll.
              // then lead to inner and outer scrolls together when we touch and move on the outer scroll element,
              // so here we reset inner scroll's 'initiated' manually.
              innerScroll.initiated = false
            })

            scroll.on('beforeScrollStart', () => {
              this.touchStartMoment = true
              const anotherScroll = arr[(index + 1) % 2]
              anotherScroll.stop()
              anotherScroll.resetPosition()
            })
          })

          innerScroll.on('scroll', (pos) => {
            // if scroll event triggered not by touch event, such as by 'scrollTo' method
            if (!innerScroll.initiated || innerScroll.isInTransition) {
              return
            }

            if (this.nestMode === NEST_MODE_NATIVE && !this.touchStartMoment) {
              return
            }

            const reachBoundary = this._checkReachBoundary(pos)
            if (reachBoundary) {
              innerScroll.resetPosition()
              innerScroll.disable()
              // Prevent outer scroll have a bouncing step when enabled in 'free' nestMode.
              outerScroll.pointX = innerScroll.pointX
              outerScroll.pointY = innerScroll.pointY
              outerScroll.enable()
            } else {
              outerScroll.disable()
            }
            this.touchStartMoment = false
          })
        })
      },
      _checkReachBoundary(pos) {
        const distX = this.bscroll.distX
        const distY = this.bscroll.distY
        const reachBoundaryX = distX > 0 ? pos.x >= this.scroll.minScrollX : distX < 0 ? pos.x <= this.scroll.maxScrollX : false
        const reachBoundaryY = distY > 0 ? pos.y >= this.scroll.minScrollY : distY < 0 ? pos.y <= this.scroll.maxScrollY : false
        const freeScroll = this.scroll.freeScroll
        const hasHorizontalScroll = this.scroll.hasHorizontalScroll
        const hasVerticalScroll = this.scroll.hasVerticalScroll

        if (!hasHorizontalScroll && !hasVerticalScroll) {
          return true
        }

        if (freeScroll) {
          return reachBoundaryX || reachBoundaryY
        }

        let reachBoundary
        if (this.scroll.movingDirectionX) {
          reachBoundary = reachBoundaryX
        } else if (this.scroll.movingDirectionY) {
          reachBoundary = reachBoundaryY
        }
        return reachBoundary
      },
      _calculateMinHeight () {
        const { wrapper, listWrapper } = this.$refs
        const pullUpLoad = this.pullUpLoad
        const pullDownRefresh = this.pullDownRefresh
        let minHeight = 0

        if (pullDownRefresh || pullUpLoad) {
          const wrapperHeight = getRect(wrapper).height
          minHeight = wrapperHeight + 1
          if (pullUpLoad && pullUpLoad.visible) {
            // minHeight = wrapperHeight + 1 - pullUpHeight, then pullUpLoad text is visible
            // when content's height is not larger than wrapper height
            minHeight -= this.pullUpHeight
          }
        }

        listWrapper.style.minHeight = `${minHeight}px`
      },
      _onPullDownRefresh () {
        this.bscroll.on('pullingDown', this._pullDownHandle)
        this.bscroll.on('scroll', this._pullDownScrollHandle)
      },
      _offPullDownRefresh () {
        this.bscroll.off('pullingDown', this._pullDownHandle)
        this.bscroll.off('scroll', this._pullDownScrollHandle)
      },
      _pullDownRefreshChangeHandler () {
        this.$nextTick(() => {
          this._getPullDownEleHeight()
          this._calculateMinHeight()
        })
      },
      _pullDownHandle () {
        if (this.resetPullDownTimer) {
          clearTimeout(this.resetPullDownTimer)
        }
        this.beforePullDown = false
        this.isPullingDown = true
        this.$emit(EVENT_PULLING_DOWN)
      },
      _pullDownScrollHandle (pos) {
        if (this.beforePullDown) {
          this.bubbleY = Math.max(0, pos.y - this.pullDownHeight)
          this.pullDownStyle = `top:${Math.min(pos.y - this.pullDownHeight, 0)}px`
          // console.log(this.pullDownStyle)
        } else {
          this.bubbleY = 0
          this.pullDownStyle = `top:${Math.min(pos.y - this.pullDownStop, 0)}px`
          // console.log(this.pullDownStyle)
        }
      },
      _waitFinishPullDown (next) {
        const { stopTime = DEFAULT_STOP_TIME } = this.pullDownRefresh
        return new Promise(resolve => {
          setTimeout(() => {
            this.bscroll.finishPullDown()
            resolve()
          }, stopTime)
        })
      },
      _waitResetPullDown (dirty) {
        return new Promise(resolve => {
          this.resetPullDownTimer = setTimeout(() => {
            this.pullDownStyle = `top: -${this.pullDownHeight}px`
            this.beforePullDown = true
            resolve()
          }, this.bscroll.options.bounceTime)
        })
      },
      _getPullDownEleHeight () {
        let pulldown = this.$refs.pulldown
        if (!pulldown) {
          return
        }
        pulldown = pulldown.firstChild
        // console.log(pulldown)
        this.pullDownHeight = getRect(pulldown).height
        // console.log(this.pullDownHeight)
        this.beforePullDown = false
        this.isPullingDown = true
        this.$nextTick(() => {
          this.pullDownStop = getRect(pulldown).height

          this.beforePullDown = true
          this.isPullingDown = false
        })
      }
    },
    watch: {
      // 监听数据的变化，重新计算高度
      data () {
        setTimeout(() => {
          this.forceUpdate(true)
        }, this.refreshDelay)
      },
      pullDownRefresh: {
        handler (newVal, oldVal) {
          if (newVal) {
            this.bscroll.openPullDown(newVal)
            if (!oldVal) {
              this._onPullDownRefresh()
              this._pullDownRefreshChangeHandler()
            }
          }

          if (!newVal && oldVal) {
            this.bscroll.closePullDown()
            this._offPullDownRefresh()
            this._pullDownRefreshChangeHandler()
          }
        },
        deep: true
      }
    }
  }
</script>

<style scoped lang="less">
  .scroll-wrapper {
    position: relative;
    width: 100%;
    height: 100%;
    overflow: hidden;

    .scroll-list-wrapper {
      min-height: 1px;
    }

    .pulldown-wrapper {
      position: absolute;
      width: 100%;
      left: 0;
      z-index: 9999;
      display: -webkit-box;
      display: -webkit-flex;
      display: flex;
      -webkit-box-pack: center;
      -webkit-justify-content: center;
      justify-content: center;
      -webkit-box-align: center;
      -webkit-align-items: center;
      align-items: center;
      -webkit-transition: all;
      transition: all;
      transform: translateY(-100%) translateZ(0);
      text-align: center;
      color: #999;
      font-size: @font-size-md;
    }
  }
</style>
