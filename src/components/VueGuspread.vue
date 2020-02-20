<template>
  <div class="guspread-wrapper" ref="wr">
    <div class="guspread-container" :data-inactive="!active" ref="inner">
      <table
        class="guspread-table"
        :data-selectedallcol="isSelectedAllCol"
        :data-selectedallrow="isSelectedAllRow"
        :data-selectedall="isSelectedAll"
      >
        <thead>
          <tr>
            <th @click.stop="clickedHeaderRoot"></th>

            <th
              v-if="invisibleWorldPrependCol && invisibleWorldPrependCol.w"
              :style="`width:${invisibleWorldPrependCol.w}px;`"
            ></th>

            <th
              :key="'hd-' + cid"
              :data-selectleft="cursors.active && (cursors.c1 - 1) == cid"
              :data-select="cursors.active && cursors.c1 <= cid && cursors.c2 >= cid"
              v-for="cid in visibleWorldCol"
              @mousedown.exact="clickedHeaderCell(cid)"
              @mousedown.shift.exact.stop="clickedHeaderCellWithShift(cid)"
              @mouseenter="enteredMouse((value.length - 1), cid)"
              @mouseup="handleMouseUp()"
            >
              <template v-if="fields[cid]">
                <slot name="field" :field="fields[cid]">{{fields[cid][labelKey]}}</slot>
              </template>
            </th>

            <th
              v-if="invisibleWorldAppendCol && invisibleWorldAppendCol.w"
              :style="`width:${invisibleWorldAppendCol.w}px;`"
            ></th>
          </tr>
        </thead>
        <tbody>
          <!-- Virtual Row-->
          <tr
            v-if="invisibleWorldPrependRow && invisibleWorldPrependRow.h"
            :style="`height:${invisibleWorldPrependRow.h}px;`"
          ></tr>
          <!-- Virtual Row-->

          <!-- Visible Row-->
          <tr :key="'row-' + rid" v-for="rid in visibleWorldRow">
            <!-- #Row -->
            <th
              :data-selectabove="cursors.active && (cursors.r1 - 1) == rid"
              :data-select="cursors.active && cursors.r1 <= rid && cursors.r2 >= rid"
              @mousedown.exact="clickedHeaderRow(rid)"
              @mousedown.shift.exact.stop="clickedHeaderRowWithShift(rid)"
              @mouseenter="enteredMouse(rid, (fields.length - 1))"
              @mouseup="handleMouseUp()"
            >{{rid + 1}}</th>
            <!-- #Row -->

            <!-- Virtual Col-->
            <td
              v-if="invisibleWorldPrependCol && invisibleWorldPrependCol.w"
              :style="`width:${invisibleWorldPrependCol.w}px;`"
            ></td>
            <!-- Virtual Col-->

            <!-- Visible Col -->
            <td
              v-for="cid in visibleWorldCol"
              :key="'row-' + rid + '-column-' + cid"
              :class="`guspread-table-cell${cellClass ? ' ' + cellClass({
                    field: fields[cid], 
                    item:value[rid], 
                    row:rid, 
                    col:cid, 
                    value:value[rid][fields[cid][nameKey]]
                    }): ''}`"
              @mousedown.exact="clickedDownCell(rid, cid)"
              @mousedown.shift.exact.stop="clickedDownCellWithShift(rid, cid)"
              @dblclick.stop="dblclickedCell(rid, cid)"
              @mouseenter="enteredMouse(rid, cid)"
              @mouseup="handleMouseUp()"
            >
              <template v-if="fields[cid] && value[rid]">
                <slot
                  name="cell"
                  :field="fields[cid]"
                  :rid="rid"
                  :cid="cid"
                  :item="value[rid]"
                >{{value[rid][fields[cid][nameKey]]}}</slot>
              </template>
            </td>
            <!-- Visible Col -->

            <!-- Virtual Col-->
            <td
              v-if="invisibleWorldAppendCol && invisibleWorldAppendCol.w"
              :style="`width:${invisibleWorldAppendCol.w}px;`"
            ></td>
            <!-- Virtual Col-->
          </tr>
          <!-- Visible Row-->

          <!-- Virtual Row-->
          <tr
            v-if="invisibleWorldAppendRow && invisibleWorldAppendRow.h"
            :style="`height:${invisibleWorldAppendRow.h}px;`"
          ></tr>
          <!-- Virtual Row-->
        </tbody>
      </table>

      <div
        class="cursor"
        ref="cursor"
        v-if="cursors.active"
        :data-multi="cursors.multi"
        :data-editmode="isEditMode"
        :style="`padding:0;transform: translate(${cursors.x}px, ${cursors.y}px);width:${cursors.w + 1}px;height:${cursors.h + 1}px;top:-1px;left:-1px`"
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
    clickedHeaderCellWithShift(c) {
      const r = this.value.length - 1;
      this.s.b = {
        r,
        c
      };
    },
    clickedHeaderCell(c) {
      this.isSelecting = true;
      const rMax = this.value.length - 1;
      this.s.a = {
        r: 0,
        c
      };
      this.s.b = {
        r: rMax,
        c
      };
    },
    clickedHeaderRowWithShift(r) {
      const c = this.fields.length - 1;
      this.s.b = {
        r,
        c
      };
    },
    clickedHeaderRow(r) {
      this.isSelecting = true;

      const cMax = this.fields.length - 1;
      this.s.a = {
        r,
        c: 0
      };
      this.s.b = {
        r,
        c: cMax
      };
    },
    clickedHeaderRoot() {
      const r = this.value.length - 1;
      const c = this.fields.length - 1;
      this.s.a = {
        r: 0,
        c: 0
      };
      this.s.b = {
        r,
        c
      };
    },
    enteredMouse(r, c) {
      if (this.isSelecting) {
        if (!(this.s.b.r == r && this.s.b.c == c)) {
          this.s.b = { r, c };
        }
      }
    },
    handleMouseUp() {
      this.isSelecting = false;
    },
    scrollCursorIntoView() {
      this.$refs.cursor.scrollIntoView({
        behavior: "auto",
        block: "nearest",
        inline: "nearest"
      });
    },
    onKeyDown(e) {
      if (this.active && !this.isEditMode) {
        if (e.key == "ArrowUp") {
          e.preventDefault();
          this.moveCursorUp(e.shiftKey);
          this.scrollCursorIntoView();
        } else if (e.key == "ArrowRight") {
          e.preventDefault();
          this.moveCursorRight(e.shiftKey);
          this.scrollCursorIntoView();
        } else if (e.key == "ArrowDown") {
          e.preventDefault();
          this.moveCursorDown(e.shiftKey);
          this.scrollCursorIntoView();
        } else if (e.key == "ArrowLeft") {
          e.preventDefault();
          this.moveCursorLeft(e.shiftKey);
          this.scrollCursorIntoView();
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
      this.s.b = { r, c };
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
        if (!shiftKey) this.s.a = { r, c };
        this.s.b = { r, c };
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

        if (!shiftKey) this.s.a = { r, c };
        this.s.b = { r, c };
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

        if (!shiftKey) this.s.a = { r, c };
        this.s.b = { r, c };
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

        if (!shiftKey) this.s.a = { r, c };
        this.s.b = { r, c };
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
      const x = 50 + c * 150;
      const y = 27 + r * 27;
      const w = 150;
      const h = 27;
      return { x, y, w, h };
    },
    clickedDownCellWithShift(r, c) {
      this.$set(this.s, "b", { r, c });
    },
    clickedDownCell(r, c) {
      if (!this.active) {
        this.active = true;
      }
      this.isSelecting = true;

      if (this.isEditMode && !(this.s.a.r == r && this.s.a.c == c)) {
        this.$nextTick(() => {
          this.isEditMode = false;
          this.$set(this.s, "a", { r, c });
          this.$set(this.s, "b", {
            r: null,
            c: null
          });
        });
      } else {
        this.$set(this.s, "a", { r, c });
        this.$set(this.s, "b", {
          r: null,
          c: null
        });
      }
    },

    handleScroll(e) {
      window.clearTimeout(this.delayTimeout);
      this.scrolling = true;
      const x1 = e.target.scrollLeft;
      const y1 = e.target.scrollTop;
      const x2 = x1 + e.target.clientWidth;
      const y2 = y1 + e.target.clientHeight;
      this.$set(this, "world", { x1, y1, x2, y2 });
      this.delayTimeout = window.setTimeout(() => {
        this.scrolling = false;
      }, 800);
    },
    scrollToTop() {
      if (this.$refs && this.$refs.wr) {
        this.$refs.wr.scrollTo(0, 0);
      }
    }
  },
  computed: {
    isSelectedAll() {
      return this.isSelectedAllRow && this.isSelectedAllCol;
    },
    isSelectedAllRow() {
      if (this.cursors) {
        const rMax = this.value.length - 1;
        const cu = this.cursors;
        return cu.r1 == 0 && cu.r2 == rMax;
      }
      return false;
    },
    isSelectedAllCol() {
      if (this.cursors) {
        const cMax = this.fields.length - 1;
        const cu = this.cursors;
        return cu.c1 == 0 && cu.c2 == cMax;
      }
      return false;
    },
    visibleWorldRow() {
      if (this.worldRows && this.value) {
        return [...Array(this.worldRows.r2 - this.worldRows.r1).keys()].map(
          i => i + this.worldRows.r1
        );
      }
      return [];
    },
    invisibleWorldPrependRow() {
      if (this.worldRows && this.worldRows.r1 > 0) {
        const h = 27 * this.worldRows.r1;
        return {
          h
        };
      }
      return null;
    },
    invisibleWorldAppendRow() {
      if (
        this.worldRows &&
        this.value &&
        this.worldRows.r2 < this.value.length
      ) {
        const h = 27 * (this.value.length - this.worldRows.r2);
        return {
          h
        };
      }
      return null;
    },
    visibleWorldCol() {
      if (this.worldCols && this.value) {
        return [...Array(this.worldCols.c2 - this.worldCols.c1).keys()].map(
          i => i + this.worldCols.c1
        );
      }
      return [];
    },
    invisibleWorldPrependCol() {
      if (this.worldCols && this.worldCols.c1 > 0) {
        const w = 150 * this.worldCols.c1;
        return {
          w
        };
      }
      return null;
    },
    invisibleWorldAppendCol() {
      if (
        this.worldCols &&
        this.value &&
        this.worldCols.c2 < this.fields.length
      ) {
        const w = 150 * (this.fields.length - this.worldCols.c2);
        return {
          w
        };
      }
      return null;
    },
    worldRows() {
      if (this.world && this.value) {
        let r1t = Math.floor(this.world.y1 / 27);
        let r2t = Math.ceil(this.world.y2 / 27);
        if (this.scrolling) {
          r1t -= 3;
          r2t += 3;
        }
        const r2m = this.value.length;
        const r1 = r1t > 0 ? r1t : 0;
        const r2 = r2t < r2m ? r2t : r2m;

        return { r1, r2 };
      }
      return {
        r1: 0,
        r2: 10
      };
    },
    worldCols() {
      if (this.world) {
        const c1 = Math.floor((this.world.x1 - 0) / 150);
        const c2t = Math.ceil((this.world.x2 - 0) / 150);
        const c2m = this.fields.length;
        const c2 = c2t < c2m ? c2t : c2m;
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
        const a = this.getPositions(this.s.a.r, this.s.a.c);
        const b = this.getPositions(this.s.b.r, this.s.b.c);
        const xw = calcMaxMix(a.x, a.w, b.x, b.w);
        const yh = calcMaxMix(a.y, a.h, b.y, b.h);
        const x = xw.min;
        const w = xw.max - x;
        const y = yh.min;
        const h = yh.max - y;
        const r1 = Math.min(this.s.a.r, this.s.b.r);
        const r2 = Math.max(this.s.a.r, this.s.b.r);
        const c1 = Math.min(this.s.a.c, this.s.b.c);
        const c2 = Math.max(this.s.a.c, this.s.b.c);
        const multi = r1 == r2 && c1 == c2 ? false : true;
        return { active: true, multi, x, y, w, h, r1, r2, c1, c2 };
      } else if (this.s.a.r != null && this.s.a.c != null) {
        const a = this.getPositions(this.s.a.r, this.s.a.c);
        const x = a.x;
        const w = a.w;
        const y = a.y;
        const h = a.h;
        const r1 = this.s.a.r;
        const r2 = this.s.a.r;
        const c1 = this.s.a.c;
        const c2 = this.s.a.c;
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
        const x1 = this.$refs.wr.scrollLeft;
        const y1 = this.$refs.wr.scrollTop;
        const x2 = x1 + this.$refs.wr.clientWidth;
        const y2 = y1 + this.$refs.wr.clientHeight;
        this.$set(this, "world", { x1, y1, x2, y2 });
      });
    }
  }
};
</script>
<style scoped lang="stylus">
.guspread-wrapper::-webkit-scrollbar {
  display: none;
}

.guspread-wrapper {
  width: 100%;
  height: 100%;
  max-height: 100vh;
  overflow: scroll;

  .guspread-container {
    position: relative;
  }

  .guspread-table {
    border-collapse: collapse;
    table-layout: fixed;
    width: 100%;

    thead {
      th {
        position: -webkit-sticky;
        position: sticky;
        top: 0;
        z-index: 3;
        font-size: 12px;
        font-weight: bold;
        text-align: center;
        width: 150px;
      }

      th:first-child {
        width: 50px;
        text-align: center;
        z-index: 4;
      }
    }

    th:first-child {
      position: -webkit-sticky;
      position: sticky;
      left: 0;
    }

    tr {
      th {
        background-color: #F5F5F5;
      }

      th, td {
        height: 27px;
        padding: 0;
        white-space: nowrap;
        overflow: hidden;
        font-size: 12px;
        -moz-user-select: none;
        -webkit-user-select: none;
        -ms-user-select: none;
      }
    }

    th[data-select=true] {
      background-color: #E0E0E0;
      color: #000;
    }

    th[data-selectleft=true]::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border-right: 1px solid #E0E0E0;
    }

    th[data-selectabove=true]::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border-bottom: 1px solid #E0E0E0;
    }

    tbody tr {
      th {
        width: 50px;
        z-index: 2;
      }

      td {
        width: 146px;
        text-align: left;
        padding: 0 2px;
        position: relative;
      }

      td::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        border-bottom: 1px solid #eaeaea;
        border-right: 1px solid #eaeaea;
        /*
        bottom: -100%;
        border-bottom: 1px solid #ccc;
        transform: scaleY(0.5);
        transform-origin: 100% 0;
        */
      }

      td::after {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        /*
        right: -100%;
        bottom: 0;
        border-right: 1px solid #ccc;
        transform: scaleX(0.5);
        transform-origin: 0 100%;
        */
      }
    }
  }

  .guspread-table[data-selectedall=true] tr th, .guspread-table[data-selectedallrow=true] thead tr th[data-select=true], .guspread-table[data-selectedallcol=true] tbody tr th[data-select=true] {
    background-color: #666;
    color: #fff;
  }

  .guspread-table[data-selectedallrow=true] {
    th[data-selectleft=true]::before {
      border-color: #666;
    }
  }

  .guspread-table[data-selectedallcol=true] {
    th[data-selectabove=true]::before {
      border-color: #666;
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
    z-index: 1;
    position: absolute;
    pointer-events: none;
    transition: box-shadow 0.1s ease-out;
    caret-color: #41b883;

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

  .cursor::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    border: 2px solid #41b883;
    transition: all 0.1s ease-out;
  }

  .cursor[data-editmode=true] {
    box-shadow: 0px 0px 10px 0px rgba(0, 0, 0, 0.14);

    &::before {
      top: -2px;
      left: -2px;
      right: -2px;
      bottom: -2px;
      /* border: 2px solid #e48c24; */
    }
  }

  .cursor[data-multi=true] {
    background-color: rgba(84, 165, 129, 0.06);
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
