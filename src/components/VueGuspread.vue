<template>
  <div class="guspread-wrapper" ref="wr">
    <div
      class="guspread-container"
      :style="`--brand-color:${color}`"
      :data-inactive="!active"
      ref="wri"
    >
      <table
        class="guspread-table"
        :data-selectedallcol="isSelectedAllCol"
        :data-selectedallrow="isSelectedAllRow"
        :data-selectedall="isSelectedAll"
      >
        <thead>
          <tr>
            <th @click="clickedHeaderRoot"></th>

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
              <template v-if="fields[cid] && fields[cid].hasOwnProperty(labelKey)">
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
              :class="`guspread-table-cell${fields[cid] && value[rid] && cellClass ? ' ' + cellClass({
                    field: fields[cid], 
                    item:value[rid], 
                    row:rid, 
                    col:cid, 
                    value:value[rid][fields[cid][nameKey]]
                    }).join(' '): ''}`"
              :data-readonly="fields[cid] && value[rid] && cellReadonly ? cellReadonly({
                    field: fields[cid], 
                    item: value[rid], 
                    row: rid, 
                    col: cid, 
                    value: value[rid][fields[cid][nameKey]]
                    }): false"
              @mousedown.exact="clickedDownCell(rid, cid)"
              @mousedown.shift.exact.stop="clickedDownCellWithShift(rid, cid)"
              @dblclick.stop="dblclickedCell(rid, cid)"
              @mouseenter="enteredMouse(rid, cid)"
              @mouseup="handleMouseUp()"
            >
              <template
                v-if="fields[cid] && fields[cid].hasOwnProperty(nameKey) && value[rid] && value[rid].hasOwnProperty(fields[cid][nameKey])"
              >
                <slot
                  name="cell"
                  :field="fields[cid]"
                  :row="rid"
                  :col="cid"
                  :item="value[rid]"
                  :value="value[rid][fields[cid][nameKey]]"
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
        :class="`cursor ${cs.class}`"
        ref="cursor"
        v-if="cursors.active"
        :data-multi="cursors.multi"
        :data-editmode="isEditMode"
        :style="`padding:0;transform: translate(${cursors.x}px, ${cursors.y}px);width:${cursors.w + 1}px;height:${cursors.h + 1}px;top:-1px;left:-1px`"
      >
        <div v-if="isEditMode && cursors.active" ref="form" class="form-container">
          <form @submit="submitted">
            <slot name="input" :field="selectedField" :item="value[s.a.r]">
              <input type="text" v-model="value[s.a.r][selectedField[nameKey]]" />
            </slot>
          </form>
        </div>
      </div>
      <div
        class="copy-cursor"
        v-if="c != null && c.active"
        :style="`padding:0;transform: translate(${c.x}px, ${c.y}px);width:${c.w + 1}px;height:${c.h + 1}px;top:-1px;left:-1px`"
      ></div>
    </div>
  </div>
</template>

<script>
import Papa from "papaparse";
const calcMaxMix = (n1, d1, n2, d2) => {
  const max = Math.max(n1, n1 + d1, n2, n2 + d2);
  const min = Math.min(n1, n1 + d1, n2, n2 + d2);
  return { max, min };
};
const DELAY = 50;
const defaultCell = {
  w: 150,
  h: 27
};
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
    color: {
      type: String,
      default: "#41b883"
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
    s: {
      a: {
        r: null,
        c: null
      },
      b: {
        r: null,
        c: null
      }
    },
    c: null,
    world: null,
    scrolling: false,
    delayTimeout: null,
    cs: {
      class: "",
      delayTimeout: null
    }
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
      this.active = true;
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
      this.active = true;
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
      this.active = true;
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
        this.$set(this, "c", { ...this.cursors });
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
              const arr = this.value
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
                    });
                });
              navigator.clipboard.writeText(
                Papa.unparse(arr, {
                  dynamicTyping: true,
                  delimiter: "\t"
                })
              );
            }
          });
      }
    },
    doPaste() {
      this.$set(this, "c", null);
      navigator.permissions.query({ name: "clipboard-read" }).then(result => {
        if (result.state == "granted" || result.state == "prompt") {
          navigator.clipboard.readText().then(tsv => {
            const rows = Papa.parse(tsv, {
              dynamicTyping: true,
              delimiter: "\t"
            });
            if (rows && rows.data && rows.data.length > 0) {
              const r = this.s.a.r;
              const c = this.s.a.c;
              this.doReplaceData({ r, c }, rows.data);
            }
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
          let val = rows[r - location.r][c - location.c];
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
            this.$set(this.value[r], this.fields[c][this.nameKey], val);
          }
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
        window.clearTimeout(this.cs.delayTimeout);
        this.cs.class = "cursor-shake";
        this.cs.delayTimeout = window.setTimeout(() => {
          this.cs.class = "";
        }, 400);
      }
    },
    getPositions(r, c) {
      const x = 50 + c * defaultCell.w;
      const y = defaultCell.h + r * defaultCell.h;
      const w = defaultCell.w;
      const h = defaultCell.h;
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
    initWorld() {
      window.clearTimeout(this.delayTimeout);
      this.scrolling = true;
      const t = this.$refs.wr;
      const x1 = t.scrollLeft;
      const y1 = t.scrollTop;
      const x2 = x1 + t.clientWidth;
      const y2 = y1 + t.clientHeight;
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
        const h = defaultCell.h * this.worldRows.r1;
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
        const h = defaultCell.h * (this.value.length - this.worldRows.r2);
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
        const w = defaultCell.w * this.worldCols.c1;
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
        const w = defaultCell.w * (this.fields.length - this.worldCols.c2);
        return {
          w
        };
      }
      return null;
    },
    worldRows() {
      if (this.world && this.value) {
        let r1t = Math.floor(this.world.y1 / defaultCell.h);
        let r2t = Math.ceil(this.world.y2 / defaultCell.h);
        if (this.scrolling) {
          r1t -= 5;
          r2t += 5;
        }
        const r2m = this.value.length;
        const r1 = r1t > 0 ? r1t : 0;
        const r2 = r2t < r2m ? r2t : r2m;

        return { r1, r2 };
      }
      return {
        r1: 0,
        r2: 0
      };
    },
    worldCols() {
      if (this.world) {
        const c1 = Math.floor((this.world.x1 - 0) / defaultCell.w);
        const c2t = Math.ceil((this.world.x2 - 0) / defaultCell.w);
        const c2m = this.fields.length;
        const c2 = c2t < c2m ? c2t : c2m;
        return { c1, c2 };
      }
      return {
        c1: 0,
        c2: 0
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
    this.widths = this.fields.map(() => {
      return defaultCell.w;
    });
    this.$nextTick(() => {
      this.$refs.wr.addEventListener("scroll", this.initWorld);
    });
  },
  mounted() {
    window.addEventListener("keydown", this.onKeyDown);
    window.addEventListener("click", this.clicked);
    window.addEventListener("resize", this.initWorld);
    this.initWorld();
  },
  beforeDestroy() {
    window.removeEventListener("keydown", this.onKeyDown);
    window.removeEventListener("click", this.clicked);
    window.removeEventListener("resize", this.initWorld);
    this.$refs.wr.removeEventListener("scroll", this.initWorld);
  },
  watch: {
    isEditMode(val) {
      if (val) {
        this.$set(this, "c", null);
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
          return defaultCell.w;
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
@keyframes shake {
  0% {
    transform: translateX(-1px) rotate(0);
  }

  25% {
    transform: translateX(2px) rotate(0.2deg);
  }

  50% {
    transform: translateX(-2px) rotate(-0.3deg);
  }

  75% {
    transform: translateX(2px) rotate(0.2deg);
  }

  100% {
    transform: translateX(-1px) rotate(0);
  }
}

.guspread-wrapper {
  width: 100%;
  height: 100%;
  max-height: 100vh;
  overflow: scroll;

  &::-webkit-scrollbar {
    display: none;
  }

  .guspread-container {
    position: relative;

    &[data-inactive=true] {
      filter: grayscale(1);
    }
  }

  .guspread-table {
    border-collapse: collapse;
    table-layout: fixed;
    width: fit-content;

    thead {
      th {
        position: -webkit-sticky;
        position: sticky;
        top: 0;
        z-index: 4;
        font-size: 12px;
        font-weight: bold;
        text-align: center;
        width: 150px;
      }

      th:first-child {
        width: 50px;
        text-align: center;
        z-index: 5;
      }
    }

    tr {
      th {
        background-color: #f2f2f2;
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

    th:first-child {
      position: -webkit-sticky;
      position: sticky;
      left: 0;
    }

    th[data-select=true] {
      background-color: #dadada;
      color: #000;
    }

    th[data-selectleft=true]::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border-right: 1px solid #dadada;
    }

    th[data-selectabove=true]::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border-bottom: 1px solid #dadada;
    }

    tbody tr {
      th {
        width: 50px;
        z-index: 3;
      }

      td {
        width: 146px;
        text-align: left;
        padding: 0 2px;
        position: relative;

        &::before {
          content: '';
          position: absolute;
          top: 0;
          left: 0;
          right: 0;
          bottom: 0;
          border-bottom: 1px solid #eaeaea;
          border-right: 1px solid #eaeaea;
        }

        &::after {
          content: '';
          position: absolute;
          top: 0;
          left: 0;
          right: 0;
          bottom: 0;
        }

        &[data-readonly=true] {
          background-color: #fafafa;
          color: #999;
        }
      }
    }
  }

  .guspread-table[data-selectedall=true] tr th, .guspread-table[data-selectedallrow=true] thead tr th[data-select=true], .guspread-table[data-selectedallcol=true] tbody tr th[data-select=true] {
    background-color: #777;
    color: #fff;
  }

  .guspread-table[data-selectedallrow=true] {
    th[data-selectleft=true]::before {
      border-color: #777;
    }
  }

  .guspread-table[data-selectedallcol=true] {
    th[data-selectabove=true]::before {
      border-color: #777;
    }
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
    transform-origin: center;
    padding: 2px;
    z-index: 1;
    position: absolute;
    pointer-events: none;
    caret-color: var(--brand-color);
    transition: box-shadow 0.2s ease;
    box-shadow: 0px 0px 12px -3px transparent;

    &.cursor-shake {
      &::before {
        animation: shake 0.2s;
        animation-iteration-count: infinite;
      }
    }

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

    &::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border: 2px solid var(--brand-color);
      transition: all 0.1s ease;
    }

    &[data-editmode=true] {
      box-shadow: 0px 0px 12px -3px var(--brand-color);

      &::before {
        top: -1px;
        left: -1px;
        right: -1px;
        bottom: -1px;
        border: 1px solid var(--brand-color);
      }
    }

    &[data-multi=true] {
      &::after {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: var(--brand-color);
        opacity: 0.05;
      }
    }
  }

  .copy-cursor {
    z-index: 2;
    position: absolute;
    pointer-events: none;
    transition: box-shadow 0.1s ease-out;

    &::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border: 2px solid #fff;
    }

    &::after {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border: 2px dashed var(--brand-color);
    }
  }
}
</style>
