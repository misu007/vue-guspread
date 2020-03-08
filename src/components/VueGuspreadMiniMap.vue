<template >
  <div class="guspread-minimap" v-if="show">
    <div
      ref="minimap"
      class="guspread-minimap-background"
      :style="`width:${map.w}px;height:${map.h}px;cursor: ${dragging ? 'grabbing' : 'grab'};`"
      @mousedown="mDown"
      @mouseup="mDone"
      @mousemove="mMove"
    ></div>
    <div ref="area" class="guspread-minimap-box" :style="boxStyle"></div>
  </div>
</template>

<script>
let moving = false;
export default {
  props: {
    showing: {
      type: Object,
      default: null
    },
    wholeWorld: {
      type: Object,
      default: null
    },
    world: {
      type: Object,
      default: null
    }
  },
  data: () => ({
    dragging: false,
    box: {
      w: 100,
      h: 75,
      x: 0,
      y: 0
    },
    start: {
      x: 0,
      y: 0
    }
  }),
  methods: {
    mDown({ clientX, clientY }) {
      this.dragging = true;
      const x = clientX - this.box.x;
      const y = clientY - this.box.y;

      this.$set(this, "start", { x, y });
    },
    mDone() {
      this.dragging = false;
    },
    moveBox({ x, y }) {
      if (!moving) {
        moving = true;
        window.requestAnimationFrame(() => {
          this.box.x = x;
          this.box.y = y;
          moving = false;
        });
      }
    },
    mMove({ clientX, clientY }) {
      if (this.dragging && clientX >= 0 && clientY >= 0) {
        const map = this.map;
        const box = this.box;
        const tx = clientX - this.start.x;
        const ty = clientY - this.start.y;
        const x = tx < 0 ? 0 : tx > map.w - box.w ? map.w - box.w : tx;
        const y = ty < 0 ? 0 : ty > map.h - box.h ? map.h - box.h : ty;
        this.moveBox({ x, y });
      }
    }
  },
  computed: {
    boxStyle() {
      return `width:${this.box.w}px;height:${this.box.h}px;right:${11 +
        this.map.w -
        this.box.w}px;bottom:${11 +
        this.map.h -
        this.box.h}px;transform: translate(${this.box.x}px,${this.box.y}px);`;
    },
    show() {
      const map = this.map;
      const box = this.box;
      if (map && box) {
        return map.w > box.w || map.h > box.h;
      }
      return false;
    },
    map() {
      const ww = this.wholeWorld;
      const tw = this.world;
      if (ww && tw) {
        const w = ww.w > tw.w ? 250 : this.box.w;
        const h = ww.h > tw.h ? 220 : this.box.h;
        return { w, h };
      }
      return {
        w: 0,
        h: 0
      };
    }
  },
  watch: {
    box: {
      handler({ x, y }) {
        if (this.dragging) {
          const map = this.map;
          const box = this.box;
          const ww = this.wholeWorld;
          const tw = this.world;
          const ax = map.w > box.w ? ((ww.w - tw.w) * x) / (map.w - box.w) : 0;
          const ay = map.h > box.h ? ((ww.h - tw.h) * y) / (map.h - box.h) : 0;
          if (ax >= 0 && ay >= 0) {
            this.$emit("scrollbox", { x: Math.round(ax), y: Math.round(ay) });
          }
        }
      },
      deep: true
    },
    world: {
      handler({ x1, y1, w, h }) {
        if (!this.dragging) {
          const map = this.map;
          const ww = this.wholeWorld;
          const box = this.box;
          if (map && map.w > 0 && map.h > 0) {
            const x = map.w > box.w ? ((map.w - box.w) * x1) / (ww.w - w) : 0;
            const y = map.h > box.h ? ((map.h - box.h) * y1) / (ww.h - h) : 0;
            this.moveBox({ x, y });
          }
        }
      },
      deep: true
    }
  }
};
</script>
<style scoped lang="stylus">
.guspread-minimap {
  opacity: 0;
  transition: opacity 0.2s;
  will-change: opacity;

  .guspread-minimap-background {
    position: absolute;
    border: 1px solid #ddd;
    border-radius: 3px;
    bottom: 10px;
    right: 10px;
    z-index: 7;
    opacity: 0.9;
    background-color: #bbb;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='100' height='20' viewBox='0 0 100 20'%3E%3Cpath fill='white' d='M0,0 L99,0 99,19 0,19'%3E%3C/path%3E%3C/svg%3E");
    background-size: 25px 5px;
    background-repeat: repeat;
    pointer-events: all;
    box-shadow: 0 0 15px -5px #aaa;
  }

  .guspread-minimap-box {
    position: absolute;
    border-radius: 3px;
    z-index: 8;
    pointer-events: all;
    opacity: 0;
    background-color: var(--brand-color);
    pointer-events: none;
    opacity: 0.2;
    will-change: transform;
  }

  &:hover {
    opacity: 1;
  }
}
</style>
