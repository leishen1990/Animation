<template>
  <div class="ion-range"
       :class="[modeClass,colorClass,{'range-pressed':pressed},{'range-disabled':disabled},{'range-has-pin':pin}]">
    <slot name="range-left"></slot>
    <div ref="slider"
         class="range-slider"
         @mouseup="pointerUp($event)"
         @touchend="pointerUp($event)"
         @mousemove="pointerMove($event)"
         @touchmove="pointerMove($event)"
         @mousedown="pointerDown($event)"
         @touchstart="pointerDown($event)">
      <div class="range-tick" v-for=" t of ticks" :style="{ left: t.left }" :class="{'range-tick-active':t.active}"
           role="presentation"></div>
      <div class="range-bar" role="presentation"></div>
      <div class="range-bar range-bar-active" :style="{ left: barL,right:barR }" ref="bar" role="presentation"></div>
      <RangeKnobHandle
        :ratio="ratioA"
        :val="valA"
        :pin="pin"
        :pressed="pressedA"
        :min="min"
        :max="max"
        :disabled="disabled">
      </RangeKnobHandle>
      <RangeKnobHandle
        :ratio="ratioB"
        :val="valB"
        :pin="pin"
        :pressed="pressedB"
        :min="min"
        :max="max"
        :disabled="disabled"
        v-if="dualKnobs">
      </RangeKnobHandle>
    </div>
    <slot name="range-right"></slot>
  </div>
</template>
<style lang="scss">

  @import "./range.scss";
  @import "./range.ios.scss";
  @import "./range.md.scss";
  @import "./range.wp.scss";

</style>
<script type="text/ecmascript-6">
  /**
   *
   * 对外事件:
   * onChange
   *
   * 如果debounce的值大于0, 则认为触发onChange事件是在松开knob之后,
   * 并在debounce时间之后触发
   * 否则随着knob的拖动而触发onChange事件
   *
   * */
  import RangeKnobHandle from './range-knob-handle.vue'
  import { setElementClass, pointerCoord } from '../../util/dom';
  import { isTrueProperty, clamp, isNumber, isObject, isString } from '../../util/util';

  export default{
    name: 'Range',
    data(){
      return {
        ticks: [], // 移动的标尺
        pressed: false, // 拖动knob

        barL: 0, // 左边bar的位置
        barR: 0, // 右边bar的位置

        ratioA: 0, // 左边bar的比例
        ratioB: 0, // 右边bar的比例
        valA: 0,
        valB: 0,
        pressedA: false,
        pressedB: false,

        _item: null, // 父组件item的句柄
        _sliderEl: null, // slider的DOM句柄
        _rect: null, // _sliderEl的尺寸对象
        _activeB: null, // 是否激活了B按钮

        _timer: null,

        valueInner: this.value, // value内部值
      }
    },
    props: {
      /**
       * 颜色: "primary", "secondary", "danger", "light", and "dark"
       */
      color: {
        type: String,
        default: '',
      },
      /**
       * 触发onChange的等待时间
       * */
      debounce: {
        type: Number,
        default: 0,
      },
      /**
       * 是否禁用
       * */
      disabled: {
        type: Boolean,
        default: false,
      },
      /**
       * 选择的拖动按钮, 默认是一个, true为两个
       * */
      dualKnobs: {
        type: Boolean,
        default: false,
      },
      /**
       * range的最大值
       * */
      max: {
        type: Number,
        default: 100,
      },
      /**
       *  range的最小值
       * */
      min: {
        type: Number,
        default: 0,
      },
      /**
       * 模式: "ios", "md", or "wp"
       * */
      mode: {
        type: String,
        default: VM.config.get('mode', 'ios'),
      },
      /**
       * 当拖动knob时显示大头针提示
       * */
      pin: {
        type: Boolean,
        default: false,
      },
      /**
       * 类似于卡槽, 如果为true, 则在range上画标尺,
       * 并且拖动中knob只能停留在标尺标记处
       * */
      snaps: {
        type: Boolean,
        default: false,
      },
      /**
       * 移动的步伐/粒度
       * */
      step: {
        type: Number,
        default: 1,
      },

      /**
       * v-model对应的值,
       * 需要出发input事件
       */
      value: {
        type: [String, Number, Object],
      },
    },
    watch: {},
    computed: {
      // 环境样式
      modeClass () {
        return `range range-${this.mode}`
      },
      // 颜色
      colorClass () {
        return !!this.color ? (`range-${this.mode}-${this.color}`) : ''
      },

      /**
       * 返回knob现在的位置比, ratio数值在0/1之间,
       * 如果使用了两个knob, 则返回最小那个
       */
      ratio(){
        if (this.dualKnobs) {
          return Math.min(this.ratioA, this.ratioB);
        }
        return this.ratioA;
      },

      /**
       * 在有两个knob的情况下, 返回最大的按个knob的ratio
       */
      ratioUpper(){
        if (this.dualKnobs) {
          return Math.max(this.ratioA, this.ratioB);
        }
        return null;
      },
    },
    methods: {
      /**
       * 拖动开始
       * @param {UIEvent} ev
       * @return {boolean}
       * */
      pointerDown(ev){
        // TODO: we could stop listening for events instead of checking this._disabled.
        // since there are a lot of events involved, this solution is
        // enough for the moment
        if (this.disabled) {
          return false;
        }

        // prevent default so scrolling does not happen
        ev.preventDefault();
        ev.stopPropagation();

        // get the start coordinates
        const current = pointerCoord(ev);

        // 获取slider元素的尺寸
        const rect = this._rect = this._sliderEl.getBoundingClientRect();

        // 判断点击的位置离那个knob近
        const ratio = clamp(0, (current.x - rect.left) / (rect.width), 1);
        this._activeB = (Math.abs(ratio - this.ratioA) > Math.abs(ratio - this.ratioB));

        // 更新激活状态的knob位置

        this.updateRange(current, rect, true);

        // trigger a haptic start
        // this._haptic.gestureSelectionStart();

        // return true so the pointer events
        // know everything's still valid
        return true;
      },

      /**
       * 拖动处理
       * @param {UIEvent} ev
       * */
      pointerMove(ev) {
        if (!this.disabled) {
          // prevent default so scrolling does not happen
          ev.preventDefault();
          ev.stopPropagation();

          // update the active knob's position
          const hasChanged = this.updateRange(pointerCoord(ev), this._rect, true);

          if (hasChanged && this.snaps) {
            // trigger a haptic selection changed event
            // if this is a snap range
          }
        }
      },

      /**
       * 拖动停止
       * @param {UIEvent} ev
       * */
      pointerUp(ev) {
        if (!this.disabled) {
          // prevent default so scrolling does not happen
          ev.preventDefault();
          ev.stopPropagation();

          if(this.debounce > 0){
            const _this = this;
            _this._timer && clearTimeout(_this._timer);
            _this._timer = setTimeout(function () {
              _this.$emit('onChange', _this.valueInner)
            }, _this.debounce)

          }



          // update the active knob's position
          this.updateRange(pointerCoord(ev), this._rect, false);
        }
      },

      /**
       * 更新knob的位置
       * @param {PointerCoordinates} current
       * @param {ClientRect} rect
       * @param {boolean} isPressed
       */
      updateRange(current, rect, isPressed) {
        // figure out where the pointer is currently at
        // update the knob being interacted with
        let ratio = clamp(0, (current.x - rect.left) / (rect.width), 1);
        let val = this.ratioToValue(ratio);

        if (this.snaps) {
          // snaps the ratio to the current value
          ratio = this.valueToRatio(val);
        }

        // update which knob is pressed
        this.pressed = isPressed;

        if (this._activeB) {
          // when the pointer down started it was determined
          // that knob B was the one they were interacting with
          this.pressedB = isPressed;
          this.pressedA = false;
          this.ratioB = ratio;

          if (val === this.valB) {
            // hasn't changed
            return false;
          }
          this.valB = val;

        } else {
          // interacting with knob A
          this.pressedA = isPressed;
          this.pressedB = false;
          this.ratioA = ratio;

          if (val === this.valA) {
            // hasn't changed
            return false;
          }
          this.valA = val;
        }

        // value has been updated
        if (this.dualKnobs) {
          // dual knobs have an lower and upper value
          if (!this.valueInner) {
            // ensure we're always updating the same object
            this.valueInner = {};
          }
          this.valueInner.lower = Math.min(this.valA, this.valB);
          this.valueInner.upper = Math.max(this.valA, this.valB);

          // console.debug(`range, updateKnob: ${ratio}, lower: ${this.valueInner.lower}, upper: ${this.valueInner.upper}`);

        } else {
          // single knob only has one value
          this.valueInner = this.valA;
          // console.debug(`range, updateKnob: ${ratio}, value: ${this.valueInner}`);
        }

        if(this.debounce === 0){
          this.$emit('onChange', this.valueInner)
        }

        this.updateBar();

        // 告知v-model
        this.$emit('input', this.valueInner)

        return true;
      },

      /**
       * @param {number} ratio
       */
      ratioToValue(ratio) {
        ratio = Math.round(((this.max - this.min) * ratio));
        ratio = Math.round(ratio / this.step) * this.step + this.min;
        return clamp(this.min, ratio, this.max);
      },
      /**
       * @param {number} ratio
       */
      valueToRatio(value) {
        value = Math.round((value - this.min) / this.step) * this.step;
        value = value / (this.max - this.min);
        return clamp(0, value, 1);
      },

      /**
       * 更新bar的位置
       */
      updateBar() {
        const ratioA = this.ratioA;
        const ratioB = this.ratioB;

        if (this.dualKnobs) {
          this.barL = `${(Math.min(ratioA, ratioB) * 100)}%`;
          this.barR = `${100 - (Math.max(ratioA, ratioB) * 100)}%`;

        } else {
          this.barL = '';
          this.barR = `${100 - (ratioA * 100)}%`;
        }

        this.updateTicks();
      },

      /**
       * 更新bar的位置
       */
      updateTicks() {
        const ticks = this.ticks;
        const ratio = this.ratio;

        if (this.snaps && ticks) {
          if (this.dualKnobs) {
            var upperRatio = this.ratioUpper;

            ticks.forEach(t => {
              t.active = (t.ratio >= ratio && t.ratio <= upperRatio);
            });

          } else {
            ticks.forEach(t => {
              t.active = (t.ratio <= ratio);
            });
          }
        }
      },

      /**
       * 在rangebar上画上移动的标尺
       * */
      createTicks() {
        if (this.snaps) {
          this.$nextTick(function () {
            this.ticks = [];
            for (var value = this.min; value <= this.max; value += this.step) {
              var ratio = this.valueToRatio(value);
              this.ticks.push({
                ratio: ratio,
                left: `${ratio * 100}%`,
              });
            }
            this.updateTicks();
          })

        }
      },

      /**
       * 更新标尺
       * */
      updateTicks() {
        const ticks = this.ticks;
        const ratio = this.ratio;

        if (this.snaps && ticks) {
          if (this.dualKnobs) {
            var upperRatio = this.ratioUpper;

            ticks.forEach(t => {
              t.active = (t.ratio >= ratio && t.ratio <= upperRatio);
            });

          } else {
            ticks.forEach(t => {
              t.active = (t.ratio <= ratio);
            });
          }
        }
      },

      /**
       * init
       * 为父元素item设置class需要等待mounted之后
       */
      initDOM(){
        // 在item父元素上添加类item-range
        if (!!this.$parent && !!this.$parent.$options._componentTag && this.$parent.$options._componentTag.toLowerCase() === 'item') {
          this._item = this.$parent;
          if (this._item.$el) {
            setElementClass(this._item.$el, 'item-range', true);
            // setElementClass(this._item.$el, 'item-range-disabled', this.disabled);
          }
        }

        // 获取slider的DOM
        this._sliderEl = this.$refs.slider;

        // 为range左右添加属性
        if (this.$slots && !!this.$slots['range-left']) {
          this.$slots['range-left'].forEach(function (item) {
            item.elm.setAttribute('range-left', '')
          })
        }
        if (this.$slots && !!this.$slots['range-right']) {
          this.$slots['range-right'].forEach(function (item) {
            item.elm.setAttribute('range-right', '')
          })
        }
      },
      /**
       * init
       * 初始化传入数据并更新视图
       */
      initData(){
        // 设置标尺
        this.createTicks();

        // 设置value的初始状态
        if (isString(this.valueInner)) {
          let val = Math.round(this.valueInner)
          if (!isNaN(val)) {
            this.valueInner = val;
          } else {
            this.valueInner = 0;
          }
        }

        if (isNumber(this.valueInner)) {
          this.ratioA = this.valueToRatio(this.valueInner);
        }

        if (isObject(this.valueInner)) {
          this.valA = Math.min(this.valueInner.lower, this.valueInner.upper);
          this.valB = Math.max(this.valueInner.lower, this.valueInner.upper);

          this.ratioA = this.valueToRatio(this.valA);
          this.ratioB = this.valueToRatio(this.valB);
        }
        // 更新bar
        this.updateBar();
      },
    },
    created () {
      this.initData();
    },
    mounted () {
      this.initDOM();
    },
    components: {
      RangeKnobHandle: RangeKnobHandle,
    }
  }
</script>
