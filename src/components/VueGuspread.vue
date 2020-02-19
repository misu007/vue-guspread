<template>
  <div class="guspread-wrapper" ref="wr">
    <div class="guspread-container" :data-inactive="!active" ref="inner">
      <table class="guspread-table">
        <thead>
          <tr>
            <th></th>
            <th
              :key="'hd-' + hid"
              :data-select="cursors.active && cursors.c1 <= hid && cursors.c2 >= hid"
              v-for="(header, hid) in fields"
            >
              <slot name="field" :field="header">{{header[labelKey]}}</slot>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr :key="'row-' + rid" v-for="(row, rid) in value">
            <th :data-select="cursors.active && cursors.r1 <= rid && cursors.r2 >= rid">{{rid + 1}}</th>
            <template v-if="(worldRows.r1 - 2) < rid && (worldRows.r2 + 1) > rid">
              <template v-for="(column, cid) in fields">
                <td
                  v-if="(worldCols.c1 - 2) < cid && (worldCols.c2 + 1 )> cid"
                  :key="'row-' + rid + '-column-' + cid"
                  :ref="'td-' + rid + '-' + cid"
                  :class="`guspread-table-cell${cellClass ? ' ' + cellClass({
                    field: fields[cid], 
                    item:value[rid], 
                    row:rid, 
                    col:cid, 
                    value:value[rid][fields[cid][nameKey]]
                    }): ''}`"
                  @mousedown.exact="clickedDownCell($event, rid, cid)"
                  @mousedown.shift.exact.stop="clickedDownCellWithShift($event, rid, cid)"
                  @dblclick.stop="dblclickedCell(rid,cid)"
                  @mouseenter="enteredMouse($event, rid,cid)"
                  @mouseup="handleMouseUp()"
                >
                  <slot
                    name="cell"
                    :field="fields[cid]"
                    :rid="rid"
                    :cid="cid"
                    :item="value[rid]"
                  >{{value[rid][fields[cid][nameKey]]}}</slot>
                </td>
                <td :key="'row-' + rid + '-column-' + cid + '-dummy'" v-else></td>
              </template>
            </template>
            <template v-else>
              <td :colspan="fields.length"></td>
            </template>
          </tr>
        </tbody>
      </table>

      <div
        class="cursor"
        ref="cursor"
        v-if="cursors.active"
        :data-multi="cursors.multi"
        :data-editmode="isEditMode"
        :style="`padding:0;transform: translate(${cursors.x}px, ${cursors.y}px);width:${cursors.w}px;height:${cursors.h}px;top:-1px;left:-1px`"
      >
        <div v-show="isEditMode && cursors.active" ref="form" class="form-container">
          <form @submit="submitted">
            <slot name="input" :field="selectedField" :item="value[s.a.r]">
              <input type="text" v-model="value[s.a.r][selectedField[nameKey]]" />
            </slot>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
const calcMaxMix = (n1, d1, n2, d2) => {
  const max = Math.max(n1, n1 + d1, n2, n2 + d2);
  const min = Math.min(n1, n1 + d1, n2, n2 + d2);
  return { max, min };
};
const DELAY = 50;
export default {
  props: {
    value: {
      type: Array,
      default: []
    },
    fields: {
      type: Array,
      default: []
    },
    nameKey: {
      type: String,
      default: "name"
    },
    labelKey: {
      type: String,
      default: "label"
    },
    options: {
      type: Object,
      default: null
    },
    cellClass: {
      type: Function,
      default: null
    },
    cellReadonly: {
      type: Function,
      default: null
    }
  },
  data: () => ({
    active: false,
    widths: [],
    isEditMode: false,
    isSelecting: false,
    isCopyed: false,
    s: {
      a: {
        r: null,
        c: null,
        x: null,
        y: null,
        w: null,
        h: null
      },
      b: {
        r: null,
        c: null,
        x: null,
        y: null,
        w: null,
        h: null
      }
    },
    c: {
      active: false,
      r1: null,
      c1: null,
      r2: null,
      c2: null,
      x: null,
      y: null,
      w: null,
      h: null
    },
    world: null,
    scrolling: false,
    delayTimeout: null
  }),
  methods: {
    clicked(e) {
      if (!this.$refs.wr.contains(e.target)) {
        this.active = false;
      }
    },
    enteredMouse(e, r, c) {
      if (this.isSelecting) {
        if (!(this.s.b.r == r && this.s.b.c == c)) {
          const x = e.currentTarget.offsetLeft;
          const y = e.currentTarget.offsetTop;
          const w = e.currentTarget.offsetWidth;
          const h = e.currentTarget.offsetHeight;
          this.s.b = { r, c, x, y, w, h };
        }
      }
    },
    handleMouseUp() {
      this.isSelecting = false;
    },
    changedUserInput() {
      console.log("c");
    },
    onKeyDown(e) {
      if (this.active && !this.isEditMode) {
        if (e.key == "ArrowUp") {
          e.preventDefault();
          this.moveCursorUp(e.shiftKey);
        } else if (e.key == "ArrowRight") {
          e.preventDefault();
          this.moveCursorRight(e.shiftKey);
        } else if (e.key == "ArrowDown") {
          e.preventDefault();
          this.moveCursorDown(e.shiftKey);
        } else if (e.key == "ArrowLeft") {
          e.preventDefault();
          this.moveCursorLeft(e.shiftKey);
        } else if (e.key == "Enter") {
          if (this.s.a.r != null && this.s.a.c != null) {
            e.preventDefault();
            this.changeToEditmode(this.s.a.r, this.s.a.c);
          }
        } else if (e.key == "c" && (e.metaKey || e.ctrlKey)) {
          this.doCopy();
        } else if (e.key == "v" && (e.metaKey || e.ctrlKey)) {
          this.doPaste();
        }
      }
    },
    doCopy() {
      if (this.cursors && this.cursors.active) {
        this.isCopyed = true;
        const target = Object.assign({}, this.cursors);
        this.$set(this, "c", target);
        navigator.permissions
          .query({ name: "clipboard-write" })
          .then(result => {
            if (result.state == "granted" || result.state == "prompt") {
              const { r1, r2, c1, c2 } = target;

              const targetFields = this.fields
                .filter((_c, cid) => {
                  return cid >= c1 && cid <= c2;
                })
                .map(field => {
                  return field[this.nameKey];
                });
              const datasetText = this.value
                .filter((row, rid) => {
                  return rid >= r1 && rid <= r2;
                })
                .map(obj => {
                  return Object.keys(obj)
                    .filter(field => {
                      return targetFields.indexOf(field) >= 0;
                    })
                    .map(field => {
                      return obj[field];
                    })
                    .join("\t");
                })
                .join("\n");
              navigator.clipboard.writeText(datasetText);
            }
          });
      }
    },
    doPaste() {
      this.isCopyed = false;
      navigator.permissions.query({ name: "clipboard-read" }).then(result => {
        if (result.state == "granted" || result.state == "prompt") {
          navigator.clipboard.readText().then(rawText => {
            const rows = rawText.split(/\n/).map(row => {
              return row.split(/\t/);
            });
            const r = this.s.a.r;
            const c = this.s.a.c;
            this.doReplaceData({ r, c }, rows);
          });
        }
      });
    },
    doReplaceData(location, rows) {
      const rCount = rows.length;
      const cCount = rows[0].length;
      const rMax = this.value.length - 1;
      const cMax = this.fields.length - 1;
      const rToBe = location.r + rCount - 1;
      const cToBe = location.c + cCount - 1;

      for (let r = location.r; r < this.value.length; r++) {
        if (r >= location.r + rCount) {
          break;
        }
        for (let c = location.c; c < this.fields.length; c++) {
          if (c >= location.c + cCount) {
            break;
          }
          const type = this.fields[c].type;
          let val = rows[r - location.r][c - location.c];
          if (type == "check") {
            if (val == "true") {
              val = true;
            } else if (val == "false") {
              val = false;
            }
          }
          this.$set(this.value[r], this.fields[c][this.nameKey], val);
        }
      }

      const r = rToBe > rMax ? rMax : rToBe;
      const c = cToBe > cMax ? cMax : cToBe;
      const { x, y, w, h } = this.getPositions(r, c);
      this.s.b = { r, c, x, y, w, h };
    },
    submitted(e) {
      e.preventDefault();
      this.moveCursorDown();
    },
    moveCursorUp(shiftKey) {
      this.isEditMode = false;
      const t = shiftKey ? this.s.b : this.s.a;
      let r = t.r;
      let c = t.c;
      if (!(r == 0 && c == 0)) {
        r--;
        if (r < 0) {
          r = this.value.length - 1;
          c = c - 1;
        }
        const { x, y, w, h } = this.getPositions(r, c);
        if (!shiftKey) this.s.a = { r, c, x, y, w, h };
        this.s.b = { r, c, x, y, w, h };
      }
    },
    moveCursorLeft(shiftKey) {
      this.isEditMode = false;
      const t = shiftKey ? this.s.b : this.s.a;
      let r = t.r;
      let c = t.c;
      if (!(r == 0 && c == 0)) {
        c--;
        if (c < 0) {
          r = r - 1;
          c = this.fields.length - 1;
        }
        const { x, y, w, h } = this.getPositions(r, c);

        if (!shiftKey) this.s.a = { r, c, x, y, w, h };
        this.s.b = { r, c, x, y, w, h };
      }
    },
    moveCursorDown(shiftKey) {
      this.isEditMode = false;
      const t = shiftKey ? this.s.b : this.s.a;
      let r = t.r;
      let c = t.c;
      if (!(r == this.value.length - 1 && c == this.fields.length - 1)) {
        r++;
        if (r > this.value.length - 1) {
          r = 0;
          c = c + 1;
        }
        const { x, y, w, h } = this.getPositions(r, c);

        if (!shiftKey) this.s.a = { r, c, x, y, w, h };
        this.s.b = { r, c, x, y, w, h };
      }
    },
    moveCursorRight(shiftKey) {
      this.isEditMode = false;
      const t = shiftKey ? this.s.b : this.s.a;
      let r = t.r;
      let c = t.c;
      if (!(r == this.value.length - 1 && c == this.fields.length - 1)) {
        c++;
        if (c > this.fields.length - 1) {
          r = r + 1;
          c = 0;
        }
        const { x, y, w, h } = this.getPositions(r, c);

        if (!shiftKey) this.s.a = { r, c, x, y, w, h };
        this.s.b = { r, c, x, y, w, h };
      }
    },
    dblclickedCell(r, c) {
      this.changeToEditmode(r, c);
    },
    changeToEditmode(r, c) {
      const isReadonly = this.cellReadonly
        ? this.cellReadonly({
            field: this.fields[c],
            item: this.value[r],
            row: r,
            col: c,
            value: this.value[r][this.fields[c][this.nameKey]]
          })
        : false;
      if (!isReadonly) {
        this.isEditMode = true;
      } else {
        alert("This field can't be changed");
      }
    },
    getPositions(r, c) {
      const ref = this.$refs["td-" + r + "-" + c];
      const column = ref && ref.length > 0 ? ref[0] : ref;
      const x = column.offsetLeft;
      const y = column.offsetTop;
      const w = column.offsetWidth;
      const h = column.offsetHeight;
      return { x, y, w, h };
    },
    clickedDownCellWithShift(e, r, c) {
      const x = e.currentTarget.offsetLeft;
      const y = e.currentTarget.offsetTop;
      const w = e.currentTarget.offsetWidth;
      const h = e.currentTarget.offsetHeight;
      this.$set(this.s, "b", { r, c, x, y, w, h });
    },
    clickedDownCell(e, r, c) {
      if (!this.active) {
        this.active = true;
      }
      this.isSelecting = true;

      const x = e.currentTarget.offsetLeft;
      const y = e.currentTarget.offsetTop;
      const w = e.currentTarget.offsetWidth;
      const h = e.currentTarget.offsetHeight;

      if (this.isEditMode && !(this.s.a.r == r && this.s.a.c == c)) {
        this.$nextTick(() => {
          this.isEditMode = false;
          this.$set(this.s, "a", { r, c, x, y, w, h });
          this.$set(this.s, "b", {
            r: null,
            c: null,
            x: null,
            y: null,
            w: null,
            h: null
          });
        });
      } else {
        this.$set(this.s, "a", { r, c, x, y, w, h });
        this.$set(this.s, "b", {
          r: null,
          c: null,
          x: null,
          y: null,
          w: null,
          h: null
        });
      }
    },

    handleScroll(e) {
      this.scrolling = true;
      const xx = e.target.scrollWidth;
      const yy = e.target.scrollHeight;
      const x1 = e.target.scrollLeft;
      const y1 = e.target.scrollTop;
      const x2 = x1 + e.target.clientWidth;
      const y2 = y1 + e.target.clientHeight;
      window.clearTimeout(this.delayTimeout);
      this.delayTimeout = setTimeout(() => {
        this.$set(this, "world", { xx, yy, x1, y1, x2, y2 });
      }, DELAY);
    },
    scrollToTop() {
      this.$refs.wr.scrollTo(0, 0);
    }
  },
  computed: {
    worldRows() {
      if (this.world) {
        const length = this.value.length;
        const r1 = (length * this.world.y1) / this.world.yy;
        const r2 = (length * this.world.y2) / this.world.yy;

        return { r1, r2 };
      }
      return {
        r1: 0,
        r2: 10
      };
    },
    worldCols() {
      if (this.world) {
        const length = this.fields.length;
        const c1 = (length * this.world.x1) / this.world.xx;
        const c2 = (length * this.world.x2) / this.world.xx;
        return { c1, c2 };
      }
      return {
        c1: 0,
        c2: 10
      };
    },
    cursors() {
      if (
        this.s.a.r != null &&
        this.s.a.c != null &&
        this.s.b.r != null &&
        this.s.b.c != null
      ) {
        const a = this.s.a;
        const b = this.s.b;
        const xw = calcMaxMix(a.x, a.w, b.x, b.w);
        const yh = calcMaxMix(a.y, a.h, b.y, b.h);
        const x = xw.min;
        const w = xw.max - x;
        const y = yh.min;
        const h = yh.max - y;
        const r1 = Math.min(a.r, b.r);
        const r2 = Math.max(a.r, b.r);
        const c1 = Math.min(a.c, b.c);
        const c2 = Math.max(a.c, b.c);
        return { active: true, multi: true, x, y, w, h, r1, r2, c1, c2 };
      } else if (this.s.a.r != null && this.s.a.c != null) {
        const a = this.s.a;
        const x = a.x;
        const w = a.w;
        const y = a.y;
        const h = a.h;
        const r1 = a.r;
        const r2 = a.r;
        const c1 = a.c;
        const c2 = a.c;
        return { active: true, multi: false, x, y, w, h, r1, r2, c1, c2 };
      }
      return { active: false };
    },
    selectedField() {
      if (this.s.a.c != null) {
        return this.fields[this.s.a.c];
      }
      return null;
    },
    selectedRow() {
      if (this.s.a.r != null) {
        return this.value[this.s.a.r];
      }
      return null;
    },

    isMultiSelect() {
      if (this.select.from.active && this.select.to.active) {
        return (
          this.select.from.r != this.select.to.r ||
          this.select.from.c != this.select.to.c
        );
      }
      return false;
    }
  },
  created() {
    window.addEventListener("keydown", this.onKeyDown);
    window.addEventListener("click", this.clicked);
    this.widths = this.fields.map(() => {
      return 150;
    });
    this.$nextTick(() => {
      this.$refs.wr.addEventListener("scroll", this.handleScroll);
    });
  },
  destroyed() {
    window.removeEventListener("keydown", this.onKeyDown);
    window.removeEventListener("click", this.clicked);
    this.$refs.wr.removeEventListener("scroll", this.handleScroll);
  },
  watch: {
    isEditMode(val) {
      if (val) {
        this.isCopyed = false;
        this.$nextTick(() => {
          const input = this.$refs.form.getElementsByTagName("input");
          if (input && input.length > 0) {
            input[0].focus();
          }
        });
      }
    },
    fields(val) {
      if (val) {
        this.$set(this.s, "a", { r: null, c: null });
        this.$set(this.s, "b", { r: null, c: null });
        this.widths = this.fields.map(() => {
          return 150;
        });
      }
    },
    value(val) {
      this.$set(this, "world", null);
      this.scrollToTop();
      this.$nextTick(() => {
        const xx = this.$refs.wr.scrollWidth;
        const yy = this.$refs.wr.scrollHeight;
        const x1 = this.$refs.wr.scrollLeft;
        const y1 = this.$refs.wr.scrollTop;
        const x2 = x1 + this.$refs.wr.clientWidth;
        const y2 = y1 + this.$refs.wr.clientHeight;
        this.$set(this, "world", { xx, yy, x1, y1, x2, y2 });
      });
    }
  }
};
</script>
<style scoped lang="stylus">
.guspread-wrapper {
  width: 100%;
  height: 100%;
  overflow: scroll;

  .guspread-container {
    position: relative;
  }

  .guspread-table {
    border-collapse: collapse;
    table-layout: fixed;
    width: 100%;

    thead th, th:first-child {
      width: 50px;
      text-align: center;
      background-color: #F5F5F5;
    }

    thead th {
      position: -webkit-sticky;
      position: sticky;
      border-top: 0.1px solid #ddd;
      top: 0;
      z-index: 1;
      padding: 4px;
      font-size: 12px;
      font-weight: bold;
      text-align: center;
    }

    th:first-child {
      position: -webkit-sticky;
      position: sticky;
      left: 0;
      border-left: 0.1px solid #ddd;
    }

    thead th:first-child {
      z-index: 2;
    }

    tr {
      th, td {
        height: 27px;
        padding: 0;
        width: 150px;
        white-space: nowrap;
        overflow: hidden;
        font-size: 12px;
        border-bottom: 0.1px solid #ddd;
        border-right: 0.1px solid #ddd;
        -moz-user-select: none;
        -webkit-user-select: none;
        -ms-user-select: none;
      }
    }

    th[data-select=true] {
      background-color: #E0E0E0;
      color: #000;
    }

    tbody tr td {
      text-align: left;
      padding: 0 2px;
    }
  }

  .guspread-container[data-inactive=true] {
    filter: grayscale(1);
  }

  .form-container {
    width: calc(100% - 5px);
    padding: 0 2px;
    
    input, form {
      margin: 0;
      padding: 0;
      border: 0;
      outline: 0;
      color: #616161;
      font-weight: inherit;
      font-style: inherit;
      font-family: inherit;
      font-size: 12px;
      vertical-align: baseline;
      box-sizing: border-box;
      box-sizing: border-box;
      -moz-box-sizing: border-box;
      -webkit-box-sizing: border-box;
    }
  }

  .cursor {
    padding: 2px;
    z-index: 3;

    input, select {
      pointer-events: all;
      width: 100%;
      height: 27px;
      border: none;
    }

    input:focus, input[type]:focus {
      outline: 0;
      box-shadow: none;
      border: 0;
      background-color: #ffffff;
    }
  }

  .cursor {
    position: absolute;
    border: 1.5px solid #448AFF;
    pointer-events: none;
  }

  .cursor[data-editmode=true] {
    border-color: #FFC400;
  }

  .cursor[data-multi=true] {
    background-color: rgba(64, 196, 255, 0.06);
  }

  .header-boundary {
    position: absolute;
    background-color: #00B0FF;
    pointer-events: none;
  }

  .cursor-multi {
    position: absolute;
    border: 0.5px solid #448AFF;
    pointer-events: none;
  }

  .cursor-multi[data-dragging=true] {
    position: absolute;
    border: 0.5px solid #40C4FF;
    pointer-events: none;
  }

  .cursor[data-draggingheader=true], .cursor-multi[data-draggingheader=true], .cursor-multi-corner[data-draggingheader=true] {
    border: 0;
    background-color: transparent;
  }

  .cursor-multi-corner {
    position: absolute;
    background-color: #448AFF;
    border: 1px solid #fff;
    width: 5px;
    height: 5px;
    pointer-events: none;
  }

  .table-header .column {
    background-color: #F5F5F5;
    text-align: center;
  }

  .table-header .column[data-selectedallrows=true] {
    background-color: #666;
    color: #fff;
  }

  .header-title {
    padding: 4px;
    font-size: 12px;
    font-weight: bold;
  }
}
</style>
