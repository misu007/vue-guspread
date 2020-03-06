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
export default {
  props: {
    showing: {
      type: Object,
      default: null
    },
    whole: {
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
    map: {
      w: 250,
      h: 220
    },

    box: {
      x: 0,
      y: 0,
      w: 100,
      h: 75
    },
    start: {
      x: 0,
      y: 0
    }
  }),
  methods: {
    _mDown(evt) {
      console.log(evt);
    },
    mDown({ clientX, clientY }) {
      this.dragging = true;
      const x = clientX - this.box.x;
      const y = clientY - this.box.y;
      this.$set(this, "start", { x, y });
    },
    mDone() {
      this.dragging = false;
    },
    mMove({ clientX, clientY }) {
      if (this.dragging && clientX >= 0 && clientY >= 0) {
        const map = this.map;
        const box = this.box;
        const x = clientX - this.start.x;
        const y = clientY - this.start.y;
        const tx = x < 0 ? 0 : x > map.w - box.w ? map.w - box.w : x;
        const ty = y < 0 ? 0 : y > map.h - box.h ? map.h - box.h : y;
        this.box.x = tx;
        this.box.y = ty;
      }
    }
  },
  computed: {
    boxStyle() {
      return `width:${this.box.w}px;height:${this.box.h}px;right:${10 +
        this.map.w -
        this.box.w}px;bottom:${10 +
        this.map.h -
        this.box.h}px;transform: translate(${this.box.x}px,${this.box.y}px);`;
    },
    show() {
      return true;
    }
  },
  watch: {
    box: {
      handler({ x, y }) {
        if (this.dragging) {
          const map = this.map;
          const box = this.box;
          const xm = map.w - box.w;
          const ym = map.h - box.h;
          const r = Math.round((this.whole.r * y) / ym);
          const c = Math.round((this.whole.c * x) / xm);
          if (r >= 0 && c >= 0) {
            this.$emit("scrollbox", { r, c });
          }
        }
      },
      deep: true
    },
    showing: {
      handler({ c, r }) {
        if (!this.dragging) {
          const x =
            ((this.map.w - this.box.w) * c.c1) / (this.whole.c - (c.c2 - c.c1));
          const y =
            ((this.map.h - this.box.h) * r.r1) / (this.whole.r - (r.r2 - r.r1));
          this.box.x = x;
          this.box.y = y;
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
  transition: opacity 0.4s;
  will-change: opacity;

  .guspread-minimap-background {
    position: absolute;
    border: 1px solid var(--brand-color);
    border-radius: 2px;
    bottom: 10px;
    right: 10px;
    z-index: 7;
    opacity: 0.7;
    background: rgba(255, 255, 255, 1);
    pointer-events: all;
    box-shadow: 0 0 15px -5px #888;
  }

  .guspread-minimap-box {
    position: absolute;
    border-radius: 2px;
    z-index: 8;
    pointer-events: all;
    opacity: 0;
    background-color: var(--brand-color);
    pointer-events: none;
    opacity: 0.7;
  }

  &:hover {
    opacity: 1;
  }
}
</style>
