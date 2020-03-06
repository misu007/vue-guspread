<template>
  <div class="app-wrapper" :style="`--brand-color:${color}`" ref="app">
    <div class="guspread-container" @wheel="scrolled">
      <table
        class="guspread-table"
        :data-selectedallcol="isSelectedAllCol"
        :data-selectedallrow="isSelectedAllRow"
        :data-selectedall="isSelectedAll"
      >
        <thead>
          <tr>
            <th @click="clickedHeaderRoot"></th>
            <template v-for="(cid, cidx) in visibleWorldCol">
              <th
                :key="'hd-' + cid"
                :data-selectleft="cursors.active && (cursors.c1 - 1) == cid"
                :data-select="cursors.active && cursors.c1 <= cid && cursors.c2 >= cid"
                @mousedown.exact="clickedHeaderCell(cid)"
                @mousedown.shift.exact.stop="clickedHeaderCellWithShift(cid)"
                @mouseenter="enteredMouse({r:(itemCount - 1), c:cid})"
                @mouseup="handleMouseUp()"
              >
                <template v-if="thisField[cidx] && thisField[cidx].hasOwnProperty(labelKey)">
                  <slot name="field" :field="thisField[cidx]">{{thisField[cidx][labelKey]}}</slot>
                </template>
              </th>
            </template>
          </tr>
        </thead>
        <tbody>
          <!-- Visible Row-->

          <template v-for="(rid, ridx) in visibleWorldRow">
            <v-guspread-tr
              :key="`ttr${ridx}`"
              :item="thisValue[ridx]"
              :cursors="cursors"
              :thisField="thisField"
              :visibleWorldCol="visibleWorldCol"
              :cellClass="cellClass"
              :cellReadonly="cellReadonly"
              :nameKey="nameKey"
              :row="rid"
              @thmdown="clickedHeaderRow(rid)"
              @thmdownshift="clickedHeaderRowWithShift(rid)"
              @thmenter="enteredMouse({r:rid, c:(fieldCount - 1)})"
              @thmup="handleMouseUp()"
              @trmdown="clickedDownCell"
              @trmdownshift="clickedDownCellWithShift"
              @trdblc="dblclickedCell"
              @trmenter="enteredMouse"
              @trmup="handleMouseUp()"
            >
              <template #cell="{field, row, col, item, value}">
                <slot
                  name="cell"
                  :field="field"
                  :row="row"
                  :col="col"
                  :item="item"
                  :value="value"
                >{{value}}</slot>
              </template>
            </v-guspread-tr>
          </template>

          <!-- Visible Row-->
        </tbody>
      </table>
    </div>
    <div class="guspread-wrapper">
      <div
        :class="`cursor ${cs.class}`"
        v-if="cursors.active"
        :data-multi="cursors.multi"
        :data-editmode="isEditMode"
        :style="`padding:0;transform: translate(${cursors.x}px, ${cursors.y}px);width:${cursors.w + 1}px;height:${cursors.h + 1}px;top:-1px;left:-1px`"
      >
        <div v-if="isEditMode && cursors.active" ref="form" class="form-container">
          <form @submit="submitted">
            <slot name="input" :field="selectedField" :item="value[s.a.r]">
              <input type="text" v-model.lazy="value[s.a.r][selectedField[nameKey]]" />
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
import VGuspreadTr from "@/components/VueGuspreadTableTr.vue";

const calcMaxMix = (n1, d1, n2, d2) => {
  const max = Math.max(n1, n1 + d1, n2, n2 + d2);
  const min = Math.min(n1, n1 + d1, n2, n2 + d2);
  return { max, min };
};
const DELAY = 400;
const defaultCell = {
  w: 150,
  h: 27
};
let scrolling = false;
let world = {
  x: 0,
  y: 0,
  w: 0,
  h: 0,
  mx: 0,
  my: 0
};

const optimizeItem = item => {
  const ret = {};
  Object.keys(item).forEach(key => {
    Object.defineProperty(ret, key, {
      configurable: false,
      value: item[key]
    });
  });
  return ret;
};

export default {
  components: {
    VGuspreadTr
  },
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

    cs: {
      class: "",
      delayTimeout: null
    },
    scrolling: false,
    delayTimeout: null,
    optimizedFields: [],
    optimizedItems: []
  }),
  methods: {
    moveNext(callback) {
      this.isEditMode = false;
      this.$nextTick(callback);
    },
    clicked(e) {
      if (!this.$refs.app.contains(e.target)) {
        this.active = false;
      }
    },
    clickedHeaderCellWithShift(c) {
      const r = this.itemCount - 1;
      this.s.b = {
        r,
        c
      };
    },
    clickedHeaderCell(c) {
      this.active = true;
      this.isSelecting = true;
      const rMax = this.itemCount - 1;
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
      const c = this.fieldCount - 1;
      this.s.b = {
        r,
        c
      };
    },
    clickedHeaderRow(r) {
      this.active = true;
      this.isSelecting = true;
      const cMax = this.fieldCount - 1;
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
      const r = this.itemCount - 1;
      const c = this.fieldCount - 1;
      this.s.a = {
        r: 0,
        c: 0
      };
      this.s.b = {
        r,
        c
      };
    },
    enteredMouse({ r, c }) {
      if (this.isSelecting) {
        if (!(this.s.b.r == r && this.s.b.c == c)) {
          this.s.b = { r, c };
        }
      }
    },
    handleMouseUp() {
      this.isSelecting = false;
    },
    scrollCursorIntoView() {},
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
              const targetoptimizedFields = this.optimizedFields
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
                      return targetoptimizedFields.indexOf(field) >= 0;
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
      const rMax = this.itemCount - 1;
      const cMax = this.fieldCount - 1;
      const rToBe = location.r + rCount - 1;
      const cToBe = location.c + cCount - 1;

      for (let r = location.r; r < this.itemCount; r++) {
        if (r >= location.r + rCount) {
          break;
        }
        for (let c = location.c; c < this.fieldCount; c++) {
          if (c >= location.c + cCount) {
            break;
          }
          let val = rows[r - location.r][c - location.c];
          const isReadonly = this.cellReadonly
            ? this.cellReadonly({
                field: this.optimizedFields[c],
                item: this.value[r],
                row: r,
                col: c,
                value: this.value[r][this.optimizedFields[c][this.nameKey]]
              })
            : false;
          if (!isReadonly) {
            this.$set(
              this.value[r],
              this.optimizedFields[c][this.nameKey],
              val
            );
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
      this.moveNext(() => {
        const t = shiftKey ? this.s.b : this.s.a;
        let r = t.r;
        let c = t.c;
        if (!(r == 0 && c == 0)) {
          r--;
          if (r < 0) {
            r = this.itemCount - 1;
            c = c - 1;
          }
          if (!shiftKey) this.s.a = { r, c };
          this.s.b = { r, c };
        }
      });
    },
    moveCursorLeft(shiftKey) {
      this.moveNext(() => {
        const t = shiftKey ? this.s.b : this.s.a;
        let r = t.r;
        let c = t.c;
        if (!(r == 0 && c == 0)) {
          c--;
          if (c < 0) {
            r = r - 1;
            c = this.fieldCount - 1;
          }

          if (!shiftKey) this.s.a = { r, c };
          this.s.b = { r, c };
        }
      });
    },
    moveCursorDown(shiftKey) {
      this.moveNext(() => {
        const t = shiftKey ? this.s.b : this.s.a;
        let r = t.r;
        let c = t.c;
        if (!(r == this.itemCount - 1 && c == this.fieldCount - 1)) {
          r++;
          if (r > this.itemCount - 1) {
            r = 0;
            c = c + 1;
          }

          if (!shiftKey) this.s.a = { r, c };
          this.s.b = { r, c };
        }
      });
    },
    moveCursorRight(shiftKey) {
      this.moveNext(() => {
        const t = shiftKey ? this.s.b : this.s.a;
        let r = t.r;
        let c = t.c;
        if (!(r == this.itemCount - 1 && c == this.fieldCount - 1)) {
          c++;
          if (c > this.fieldCount - 1) {
            r = r + 1;
            c = 0;
          }

          if (!shiftKey) this.s.a = { r, c };
          this.s.b = { r, c };
        }
      });
    },
    dblclickedCell({ r, c }) {
      this.changeToEditmode(r, c);
    },
    changeToEditmode(r, c) {
      const isReadonly = this.cellReadonly
        ? this.cellReadonly({
            field: this.optimizedFields[c],
            item: this.value[r],
            row: r,
            col: c,
            value: this.value[r][this.optimizedFields[c][this.nameKey]]
          })
        : false;
      if (!isReadonly) {
        this.isEditMode = true;
      } else {
        window.clearTimeout(this.cs.delayTimeout);
        this.cs.class = "cursor-shake";
        this.cs.delayTimeout = window.setTimeout(() => {
          this.cs.class = "";
        }, DELAY);
      }
    },
    getPositions(r, c) {
      const x = 50 + c * defaultCell.w;
      const y = defaultCell.h + r * defaultCell.h;
      const w = defaultCell.w;
      const h = defaultCell.h;
      return { x, y, w, h };
    },
    clickedDownCellWithShift({ r, c }) {
      this.$set(this.s, "b", { r, c });
    },
    clickedDownCell({ r, c }) {
      if (!this.active) {
        this.active = true;
      }
      this.isSelecting = true;

      if (this.isEditMode && !(this.s.a.r == r && this.s.a.c == c)) {
        this.moveNext(() => {
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
    scrolled(evt) {
      if (!this.isEditMode && !scrolling) {
        scrolling = true;
        this.myRequestFrame(evt);
      }
      this.scrolling = true;
      window.clearTimeout(this.delayTimeout);
      this.delayTimeout = window.setTimeout(() => {
        this.scrolling = false;
      }, 200);
    },
    myRequestFrame(evt) {
      window.requestAnimationFrame(() => {
        const deltaX = evt ? evt.deltaX : 0;
        const deltaY = evt ? evt.deltaY : 0;
        const dx = Math.abs(deltaX);
        const dy = Math.abs(deltaY);
        const _x = dx > dy ? world.x + deltaX : world.x;
        const x =
          _x < 0 ? 0 : _x > world.mx - world.w ? world.mx - world.w : _x;
        const _y = dy >= dx ? world.y + deltaY : world.y;
        const y =
          _y < 0 ? 0 : _y > world.my - world.h ? world.my - world.h : _y;
        const x1 = x;
        const y1 = y;
        const x2 = x1 + world.w + 1;
        const y2 = y1 + world.h;
        world.x = x;
        world.y = y;
        this.$set(this, "world", { x1, y1, x2, y2 });
        scrolling = false;
      });
    },
    initWorld() {
      const t = this.$refs.app;
      world.x = 0;
      world.y = 0;
      world.w = t.clientWidth;
      world.h = t.clientHeight;
      if (this.itemCount > 0) {
        world.my = (this.itemCount + 2) * 27;
      }
      if (this.fieldCount > 0) {
        world.mx = this.fieldCount * 150 + 200;
      }
      const x1 = world.x;
      const x2 = x1 + t.clientWidth;
      const y1 = world.y;
      const y2 = y1 + t.clientHeight;
      this.$nextTick(() => {
        this.$set(this, "world", { x1, y1, x2, y2 });
      });
    },
    optimizeFields() {
      const fields = this.fields;
      if (fields) {
        this.optimizedFields = fields.map(item => {
          return optimizeItem(item);
        });
        world.mx = fields.length * 150 + 200;
      }
    },
    optimizeItems() {
      const val = this.value;
      if (val) {
        this.optimizedItems = val.map(item => {
          return optimizeItem(item);
        });
      }
      world.my = (val.length + 2) * 27;
    }
  },
  computed: {
    isSelectedAll() {
      return this.isSelectedAllRow && this.isSelectedAllCol;
    },
    isSelectedAllRow() {
      if (this.cursors) {
        const rMax = this.itemCount - 1;
        const cu = this.cursors;
        return cu.r1 == 0 && cu.r2 == rMax;
      }
      return false;
    },
    isSelectedAllCol() {
      if (this.cursors) {
        const cMax = this.fieldCount - 1;
        const cu = this.cursors;
        return cu.c1 == 0 && cu.c2 == cMax;
      }
      return false;
    },
    visibleWorldRow() {
      if (this.worldRows && this.itemCount > 0) {
        if (this.worldRows.r2 - this.worldRows.r1 > 0) {
          return [...Array(this.worldRows.r2 - this.worldRows.r1).keys()].map(
            i => i + this.worldRows.r1
          );
        }
      }
      return [];
    },
    thisValue() {
      if (this.visibleWorldRow.length > 0) {
        return this.value.slice(this.worldRows.r1, this.worldRows.r2);
      }
      return [];
    },
    visibleWorldCol() {
      if (this.worldCols && this.fieldCount > 0) {
        if (this.worldCols.c2 - this.worldCols.c1 > 0) {
          return [...Array(this.worldCols.c2 - this.worldCols.c1).keys()].map(
            i => i + this.worldCols.c1
          );
        }
      }
      return [];
    },
    thisField() {
      if (this.visibleWorldCol.length > 0) {
        return this.optimizedFields.slice(this.worldCols.c1, this.worldCols.c2);
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
    worldRows() {
      if (this.world && this.value) {
        const r1t = Math.floor(this.world.y1 / defaultCell.h);
        const diff = Math.floor(
          (this.world.y2 - this.world.y1) / defaultCell.h
        );
        const r2t = r1t + diff;
        const r2m = this.itemCount;
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
        const c1t = Math.floor(this.world.x1 / defaultCell.w);
        const diff = Math.floor(
          (this.world.x2 - this.world.x1) / defaultCell.w
        );
        const c1 = c1t > 0 ? c1t : 0;
        const c2t = c1 + diff + 1;
        const c2m = this.fieldCount;
        const c2 = c2t < c2m ? c2t : c2m - 1;
        return { c1, c2 };
      }
      return {
        c1: 0,
        c2: 0
      };
    },
    fieldCount() {
      if (this.optimizedFields) {
        return this.optimizedFields.length;
      }
      return 0;
    },
    itemCount() {
      if (this.optimizedItems) {
        return this.optimizedItems.length;
      }
      return 0;
    },
    cursors() {
      if (
        this.s.a.r != null &&
        this.s.a.c != null &&
        this.s.b.r != null &&
        this.s.b.c != null &&
        this.worldRows
      ) {
        const r1 = Math.min(this.s.a.r, this.s.b.r);
        const r2 = Math.max(this.s.a.r, this.s.b.r);
        const c1 = Math.min(this.s.a.c, this.s.b.c);
        const c2 = Math.max(this.s.a.c, this.s.b.c);
        const x = 50 + (c1 - this.worldCols.c1) * defaultCell.w;
        const y = defaultCell.h + (r1 - this.worldRows.r1) * defaultCell.h;
        const w = (c2 - c1 + 1) * defaultCell.w;
        const h = (r2 - r1 + 1) * defaultCell.h;

        const multi = r1 == r2 && c1 == c2 ? false : true;
        return { active: true, multi, x, y, w, h, r1, r2, c1, c2 };
      } else if (this.s.a.r != null && this.s.a.c != null) {
        const r1 = this.s.a.r;
        const r2 = this.s.a.r;
        const c1 = this.s.a.c;
        const c2 = this.s.a.c;
        const x = 50 + (c1 - this.worldCols.c1) * defaultCell.w;
        const y = defaultCell.h + (r1 - this.worldRows.r1) * defaultCell.h;
        const w = defaultCell.w;
        const h = defaultCell.h;
        return { active: true, multi: false, x, y, w, h, r1, r2, c1, c2 };
      }
      return { active: false };
    },
    selectedField() {
      if (this.s.a.c != null) {
        return this.optimizedFields[this.s.a.c];
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
    this.widths = this.optimizedFields.map(() => {
      return defaultCell.w;
    });
  },
  mounted() {
    window.addEventListener("keydown", this.onKeyDown);
    window.addEventListener("click", this.clicked);
    window.addEventListener("resize", this.initWorld);
    this.optimizeFields();
    this.optimizeItems();
    this.initWorld();
  },
  beforeDestroy() {
    window.removeEventListener("keydown", this.onKeyDown);
    window.removeEventListener("click", this.clicked);
    window.removeEventListener("resize", this.initWorld);
  },
  watch: {
    scrolling(val) {
      this.$emit("changeScrolling", val);
    },
    isEditMode(val) {
      this.$emit("changeEditMode", val);
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
        this.optimizeFields();
        this.$set(this.s, "a", { r: null, c: null });
        this.$set(this.s, "b", { r: null, c: null });
        this.widths = this.optimizedFields.map(() => {
          return defaultCell.w;
        });
      }
    },
    value(val) {
      this.optimizeItems();
      this.$set(this, "world", null);
      window.requestAnimationFrame(() => {
        const x1 = 0;
        const y1 = 0;
        world.x = 0;
        world.y = 0;
        const x2 = x1 + this.$refs.app.clientWidth;
        const y2 = y1 + this.$refs.app.clientHeight;
        this.$set(this, "world", { x1, y1, x2, y2 });
      });
    },
    cursors(val) {
      if (val && val.active) {
        this.$emit("changeFocused", {
          a: { row: val.r1, col: val.c1 },
          b: { row: val.r2, col: val.c2 }
        });
      }
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

.app-wrapper {
  width: 100%;
  height: 100%;
  max-height: 100vh;
  position: relative;
  overflow: hidden;
  background-color: #424242;

  .guspread-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;

    .guspread-table {
      border-collapse: collapse;
      table-layout: fixed;
      width: fit-content;

      thead {
        th {
          font-size: 12px;
          font-weight: bold;
          text-align: center;
          width: 150px;
          position: relative;
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
          position: relative;
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
        background-color: #dadada;
        color: #000;
      }

      th[data-selectleft=true]::before {
        content: '';
        position: absolute;
        top: 0;
        left: -1px;
        right: 0;
        bottom: 0;
        border-right: 1px solid #dadada;
        pointer-events: none;
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
            pointer-events: none;
          }

          &::after {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            pointer-events: none;
          }

          &[data-readonly=true] {
            background-color: #fafafa;
            color: #999;
          }
        }
      }
    }

    .guspread-table[data-selectedall=true] >>> tr th, .guspread-table[data-selectedallrow=true] thead tr th[data-select=true], .guspread-table[data-selectedallcol=true] tbody >>> tr th[data-select=true] {
      background-color: #777;
      color: #fff;
    }

    .guspread-table[data-selectedallrow=true] {
      >>> th[data-selectleft=true]::before {
        border-color: #777;
      }
    }

    .guspread-table[data-selectedallcol=true] {
      >>> th[data-selectabove=true]::before {
        border-color: #777;
      }
    }
  }

  .guspread-wrapper {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 2;
    pointer-events: none;
    overflow: scroll;

    &::-webkit-scrollbar {
      display: none;
    }

    .guspread-scroll-container {
      position: static;
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
      z-index: 3;
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
        z-index: 4;
        pointer-events: none;
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
          pointer-events: none;
        }
      }
    }

    .copy-cursor {
      z-index: 4;
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
        pointer-events: none;
      }

      &::after {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        border: 2px dashed var(--brand-color);
        pointer-events: none;
      }
    }
  }
}
</style>
