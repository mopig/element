<template>
  <div class="el-slider">
    <el-input-number
      v-model="inputValue"
      v-if="showInput"
      class="el-slider__input"
      @keyup.native="onInputChange"
      ref="input"
      :step="step"
      :min="min"
      :max="max"
      size="small">
    </el-input-number>
    <div class="el-slider__runway"
      :class="{ 'show-input': showInput }"
      @click="onSliderClick" ref="slider">
      <div class="el-slider__bar" :style="{ width: currentPosition }"></div>
      <div
        class="el-slider__button-wrapper"
        @mouseenter="hovering = true"
        @mouseleave="hovering = false"
        @mousedown="onButtonDown"
        :style="{left: currentPosition}" ref="button">
        <div class="el-slider__button" :class="{ 'hover': hovering, 'dragging': dragging }"></div>
      </div>
      <transition name="popper-fade">
        <div class="el-slider__pop" v-show="showTip" ref="pop">{{ value }}</div>
      </transition>
      <div class="el-slider__stop" v-for="item in stops" :style="{ 'left': item + '%' }" v-if="showStops"></div>
    </div>
  </div>
</template>

<script type="text/babel">
  import Popper from 'element-ui/src/utils/popper';
  import ElInputNumber from 'element-ui/packages/input-number/index.js';
  import { getStyle } from 'wind-dom/src/style';
  import { addClass, removeClass } from 'wind-dom/src/class';

  export default {
    name: 'ElSlider',

    props: {
      min: {
        type: Number,
        default: 0
      },
      max: {
        type: Number,
        default: 100
      },
      step: {
        type: Number,
        default: 1
      },
      defaultValue: {
        type: Number,
        default: 0
      },
      value: {
        type: Number,
        default: 0
      },
      showInput: {
        type: Boolean,
        default: false
      },
      showStops: {
        type: Boolean,
        default: false
      }
    },

    components: {
      ElInputNumber
    },

    data() {
      return {
        inputValue: null,
        timeout: null,
        showTip: false,
        hovering: false,
        dragging: false,
        startX: 0,
        currentX: 0,
        startPos: 0,
        popper: null,
        newPos: null,
        oldValue: this.value,
        currentPosition: (this.value - this.min) / (this.max - this.min) * 100 + '%'
      };
    },

    watch: {
      inputValue(val) {
        this.$emit('input', Number(val));
      },

      showTip(val) {
        if (val) {
          this.$nextTick(() => {
            this.updatePopper();
          });
        } else {
          this.timeout = setTimeout(() => {
            if (this.popper) {
              this.popper.destroy();
              this.popper = null;
            }
          }, 300);
        }
      },

      value(val) {
        this.$nextTick(() => {
          this.updatePopper();
        });
        if (val < this.min) {
          this.$emit('input', this.min);
          return;
        }
        if (val > this.max) {
          this.$emit('input', this.max);
          return;
        }
        this.inputValue = val;
        this.setPosition((val - this.min) * 100 / (this.max - this.min));
      }
    },

    methods: {
      handlePopperStyle() {
        let placementMap = { top: 'bottom', bottom: 'top' };
        let placement = this.popper._popper.getAttribute('x-placement').split('-')[0];
        let origin = placementMap[placement];
        addClass(this.popper._popper, placement);
        removeClass(this.popper._popper, placementMap[placement]);
        this.popper._popper.style.transformOrigin = `center ${ origin }`;
      },

      updatePopper() {
        if (this.popper) {
          clearTimeout(this.timeout);
          this.popper.update();
          this.handlePopperStyle();
        } else {
          this.popper = new Popper(this.$refs.button, this.$refs.pop, { gpuAcceleration: false, placement: 'top' });
          this.popper.onCreate(() => {
            this.handlePopperStyle();
          });
          this.updatePopper();
        }
      },

      setPosition(newPos) {
        if (newPos >= 0 && (newPos <= 100)) {
          var lengthPerStep = 100 / ((this.max - this.min) / this.step);
          var steps = Math.round(newPos / lengthPerStep);
          this.$emit('input', Math.round(steps * lengthPerStep * (this.max - this.min) * 0.01 + this.min));
          this.currentPosition = (this.value - this.min) / (this.max - this.min) * 100 + '%';
          if (!this.dragging) {
            if (this.value !== this.oldValue) {
              this.$emit('change', this.value);
              this.oldValue = this.value;
            }
          }
        }
      },

      onSliderClick(event) {
        var currentX = event.clientX;
        var sliderOffsetLeft = this.$refs.slider.getBoundingClientRect().left;
        var newPos = (currentX - sliderOffsetLeft) / this.$sliderWidth * 100;
        this.setPosition(newPos);
      },

      onInputChange() {
        if (this.value === '') {
          return;
        }
        if (!isNaN(this.value)) {
          this.setPosition((this.value - this.min) * 100 / (this.max - this.min));
        }
      },

      onDragStart(event) {
        this.dragging = true;
        this.startX = event.clientX;
        this.startPos = parseInt(this.currentPosition, 10);
      },

      onDragging(event) {
        if (this.dragging) {
          this.currentX = event.clientX;
          var diff = (this.currentX - this.startX) / this.$sliderWidth * 100;
          this.newPos = this.startPos + diff;
          this.setPosition(this.newPos);
        }
      },

      onDragEnd() {
        if (this.dragging) {
          this.dragging = false;
          this.setPosition(this.newPos);
          window.removeEventListener('mousemove', this.onDragging);
          window.removeEventListener('mouseup', this.onDragEnd);
        }
      },

      onButtonDown(event) {
        this.onDragStart(event);
        window.addEventListener('mousemove', this.onDragging);
        window.addEventListener('mouseup', this.onDragEnd);
      }
    },

    computed: {
      $sliderWidth() {
        return parseInt(getStyle(this.$refs.slider, 'width'), 10);
      },

      showTip() {
        return this.dragging || this.hovering;
      },

      stops() {
        let stopCount = (this.max - this.value) / this.step;
        let result = [];
        let currentLeft = parseFloat(this.currentPosition);
        let stepWidth = 100 * this.step / (this.max - this.min);
        for (let i = 1; i < stopCount; i++) {
          result.push(currentLeft + i * stepWidth);
        }
        return result;
      }
    },

    created() {
      if (typeof this.value !== 'number' || this.value < this.min) {
        this.$emit('input', this.min);
      } else if (this.value > this.max) {
        this.$emit('input', this.max);
      }
      this.$nextTick(() => {
        this.inputValue = this.inputValue || this.value;
      });
    },

    beforeDestroy() {
      if (this.popper) {
        this.popper.destroy();
      }
    }
  };
</script>
