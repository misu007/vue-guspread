<template>
  <div class="app-wrapper" :style="`--brand-color:${color}`" ref="app">
    <div
      class="guspread-container"
      :data-selectedallcol="isSelectedAllCol"
      :data-selectedallrow="isSelectedAllRow"
      :data-selectedall="isSelectedAll"
      @wheel="scrolled"
    >
      <!-- Table Root -->
      <div
        class="guspread-table-root"
        :data-selectleft="cursors.active && cursors.c1 == 0"
        :data-selectabove="cursors.active && cursors.r1 == 0"
        :style="`width:${rootRect.w - 1}px;height:${rootRect.h - 1}px;`"
        @click="clickedHeaderRoot"
      ></div>
      <!-- Table Root -->

      <!-- Header Row -->
      <table
        class="guspread-table guspread-table-header-row"
        :style="`top:0;left:${rootRect.w}px;transform:translateX(-${offsetWorld.x}px);`"
      >
        <thead>
          <tr>
            <template v-for="(cid, cidx) in visibleWorldCol">
              <th
                :key="'hd-' + cid"
                style="height:26px;"
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
      </table>
      <!-- Header Row -->

      <!-- Header Col -->
      <table
        class="guspread-table guspread-table-header-col"
        :style="`left:0;top:${rootRect.h}px;transform:translateY(-${offsetWorld.y}px);`"
      >
        <tbody>
          <!-- Visible Row-->

          <template v-for="rid in visibleWorldRow">
            <tr :key="`hcttr${rid}`">
              <th
                style="height:27px;width:49px;"
                :data-selectabove="cursors.active && (cursors.r1 - 1) == rid"
                :data-select="cursors.active && cursors.r1 <= rid && cursors.r2 >= rid"
                @mousedown.exact="clickedHeaderRow(rid)"
                @mousedown.shift.exact.stop="clickedHeaderRowWithShift(rid)"
                @mouseenter="enteredMouse({r:rid, c:(fieldCount - 1)})"
                @mouseup="handleMouseUp()"
              >{{rid + 1}}</th>
            </tr>
          </template>

          <!-- Visible Row-->
        </tbody>
      </table>
      <!-- Header Col -->

      <!-- Table Body -->
      <table
        class="guspread-table guspread-table-body"
        :style="`top:${rootRect.h}px;left:${rootRect.w}px;transform:translate(-${offsetWorld.x}px,-${offsetWorld.y}px);`"
      >
        <tbody>
          <!-- Visible Row-->

          <template v-for="rid in visibleWorldRow">
            <v-guspread-tr
              :key="`tbttr${rid}`"
              :item="value[rid]"
              :cursors="cursors"
              :thisField="thisField"
              :visibleWorldCol="visibleWorldCol"
              :cellClass="cellClass"
              :rowClass="rowClass"
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
      <!-- Table Body -->

      <!-- Cursor -->
      <div
        :class="`cursor ${cs.class}`"
        v-if="cursors.active"
        :data-multi="cursors.multi"
        :data-editmode="isEditMode"
        :style="`padding:0;transform: translate(${cursors.x}px, ${cursors.y}px);width:${cursors.w + 1}px;height:${cursors.h + 1}px;top:-1px;left:-1px`"
      >
        <div v-if="isEditMode && cursors.active" ref="form" class="form-container">
          <form @submit="submitted">
            <slot name="input" :field="selectedField" :item="selectedRow">
              <input type="text" v-model.lazy="selectedRow[selectedField[nameKey]]" />
            </slot>
          </form>
        </div>
      </div>
      <!-- Cursor -->

      <!-- Cursor -->
      <div
        class="copy-cursor"
        v-if="copyCursors.active"
        :style="`padding:0;transform: translate(${copyCursors.x}px, ${copyCursors.y}px);width:${copyCursors.w + 1}px;height:${copyCursors.h + 1}px;top:-1px;left:-1px`"
      ></div>
      <!-- Cursor -->

      <!-- Mini Map -->
      <v-guspread-mini-map
        class="guspread-minimap"
        v-if="showMinimap"
        :wholeWorld="wholeWorld"
        :world="world"
        @scrollbox="scrolledBox"
      ></v-guspread-mini-map>
      <!-- Mini Map -->
    </div>
  </div>
</template>

<script>
import Papa from "papaparse";
import VGuspreadTr from "@/components/VueGuspreadTableTr.vue";
import VGuspreadMiniMap from "@/components/VueGuspreadMiniMap.vue";

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
    VGuspreadTr,
    VGuspreadMiniMap
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
    hideMinimap: {
      type: Boolean,
      default: false
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
    rowClass: {
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
      navigator.permissions
        .query({ name: "clipboard-read" })
        .then(({ state }) => {
          if (state == "granted" || state == "prompt") {
            navigator.clipboard.readText().then(tsv => {
              const { data } = Papa.parse(tsv, {
                dynamicTyping: true,
                delimiter: "\t"
              });
              if (data && data.length > 0) {
                const a = this.s.a;
                const b = this.s.b;
                let r = a.r;
                let c = a.c;
                let w = 1;
                let h = 1;
                if (b.r != null && b.c != null) {
                  if (b.r < a.r) {
                    r = b.r;
                  }
                  if (b.c < a.c) {
                    c = b.c;
                  }
                  h = Math.abs(a.r - b.r) + 1;
                  w = Math.abs(a.c - b.c) + 1;
                }
                this.doReplaceData({ r, c, w, h }, data);
              }
            });
          }
        });
    },
    doReplaceData(location, rows) {
      const selectedw = location.w;
      const selectedh = location.h;
      const rCount = rows.length;
      const cCount = rows[0].length;
      const rMax = this.itemCount - 1;
      const cMax = this.fieldCount - 1;
      const rToBe = location.r - 1 + (rCount > selectedh ? rCount : selectedh);
      const cToBe = location.c - 1 + (cCount > selectedw ? cCount : selectedw);
      for (let r = location.r; r < this.itemCount; r++) {
        if (r >= location.r + rCount && r >= location.r + selectedh) {
          break;
        }
        for (let c = location.c; c < this.fieldCount; c++) {
          if (c >= location.c + cCount && c >= location.c + selectedw) {
            break;
          }
          const rIdx = (r - location.r) % rCount;
          const cIdx = (c - location.c) % cCount;
          let val = rows[rIdx][cIdx];
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
    scrolledBox({ x, y }) {
      if (!this.isEditMode) {
        this.calcScrolledPosition({ tx: x, ty: y });
      }
      window.clearTimeout(this.delayTimeout);
      this.delayTimeout = window.setTimeout(() => {
        this.scrolling = false;
        scrolling = false;
      }, 800);
    },
    scrolled({ deltaX, deltaY }) {
      if (!this.isEditMode) {
        this.calcScrolledPosition({ deltaX, deltaY });
      }
      window.clearTimeout(this.delayTimeout);
      this.delayTimeout = window.setTimeout(() => {
        this.scrolling = false;
        scrolling = false;
      }, 800);
    },
    animateFrame() {
      window.requestAnimationFrame(() => {
        const x1 = world.x;
        const y1 = world.y;
        const x2 = x1 + world.w + 1;
        const y2 = y1 + world.h;
        this.world.x1 = x1;
        this.world.x2 = x2;
        this.world.y1 = y1;
        this.world.y2 = y2;
        if (scrolling) this.animateFrame();
      });
    },
    calcScrolledPosition({ deltaX, deltaY, tx, ty }) {
      let _x = world.x;
      let _y = world.y;
      if (deltaX != null && deltaY != null) {
        const dx = Math.abs(deltaX);
        const dy = Math.abs(deltaY);
        if (dx > dy) {
          _x = world.x + deltaX;
        } else if (dy >= dx) {
          _y = world.y + deltaY;
        }
      } else if (tx != null && ty != null) {
        _x = tx;
        _y = ty;
      }
      const x1 =
        _x < 0
          ? 0
          : _x > world.mx - world.w - 150
          ? world.mx - world.w - 150
          : _x;
      const y1 =
        _y < 0
          ? 0
          : _y > world.my - world.h - 27
          ? world.my - world.h - 27
          : _y;
      world.x = x1;
      world.y = y1;
      if (!scrolling) {
        scrolling = true;
        this.scrolling = true;
        this.animateFrame();
      }
    },
    initWorld() {
      this.changeWorld(true);
    },
    resizeWorld() {
      this.changeWorld();
    },
    changeWorld(init) {
      const t = this.$refs.app;
      if (init) {
        world.x = 0;
        world.y = 0;
      }
      const w = t.clientWidth;
      const h = t.clientHeight;
      world.w = w;
      world.h = h;
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
        this.$set(this, "world", { x1, y1, x2, y2, w, h });
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
    wholeWorld() {
      const rc = this.itemCount;
      const cc = this.fieldCount;
      if (rc && cc) {
        const h = defaultCell.h * (rc + 1);
        const w = defaultCell.w * cc + 50;
        return { w, h };
      }
      return null;
    },
    rootRect() {
      return {
        w: 50,
        h: defaultCell.h
      };
    },
    showMinimap() {
      return (
        !this.hideMinimap &&
        this.world &&
        !(this.cursors.active && this.cursors.multi) &&
        !this.isEditMode
      );
    },
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
    offsetWorld() {
      const world = this.world;
      const ret = { x: 0, y: 0 };
      if (world && world.x1 && world.x1 > 0) {
        ret.x = world.x1 % defaultCell.w;
      }
      if (world && world.y1 && world.y1 > 0) {
        ret.y = world.y1 % defaultCell.h;
      }
      return ret;
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
      if (this.world) {
        const r1t = Math.floor(this.world.y1 / defaultCell.h);
        const diff = Math.floor(
          (this.world.y2 - this.world.y1) / defaultCell.h
        );
        const r2t = r1t + diff + 1;
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
        const c2t = c1 + diff + 2;
        const c2m = this.fieldCount;
        const c2 = c2t < c2m ? c2t : c2m;
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
    copyCursors() {
      if (
        this.c &&
        this.c.r1 != null &&
        this.c.c1 != null &&
        this.c.r2 != null &&
        this.c.c2 != null &&
        this.worldRows
      ) {
        const r1 = this.c.r1;
        const r2 = this.c.r2;
        const c1 = this.c.c1;
        const c2 = this.c.c2;
        const x =
          50 + (c1 - this.worldCols.c1) * defaultCell.w - this.offsetWorld.x;
        const y =
          defaultCell.h +
          (r1 - this.worldRows.r1) * defaultCell.h -
          this.offsetWorld.y;
        const w = (c2 - c1 + 1) * defaultCell.w;
        const h = (r2 - r1 + 1) * defaultCell.h;

        return { active: true, x, y, w, h, r1, r2, c1, c2 };
      }
      return { active: false };
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
        const x =
          50 + (c1 - this.worldCols.c1) * defaultCell.w - this.offsetWorld.x;
        const y =
          defaultCell.h +
          (r1 - this.worldRows.r1) * defaultCell.h -
          this.offsetWorld.y;
        const w = (c2 - c1 + 1) * defaultCell.w;
        const h = (r2 - r1 + 1) * defaultCell.h;

        const multi = r1 == r2 && c1 == c2 ? false : true;
        return { active: true, multi, x, y, w, h, r1, r2, c1, c2 };
      } else if (this.s.a.r != null && this.s.a.c != null) {
        const r1 = this.s.a.r;
        const r2 = this.s.a.r;
        const c1 = this.s.a.c;
        const c2 = this.s.a.c;
        const x =
          50 + (c1 - this.worldCols.c1) * defaultCell.w - this.offsetWorld.x;
        const y =
          defaultCell.h +
          (r1 - this.worldRows.r1) * defaultCell.h -
          this.offsetWorld.y;
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
    window.addEventListener("resize", this.resizeWorld);
    this.optimizeFields();
    this.optimizeItems();
    this.initWorld();
  },
  beforeDestroy() {
    window.removeEventListener("keydown", this.onKeyDown);
    window.removeEventListener("click", this.clicked);
    window.removeEventListener("resize", this.resizeWorld);
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
  min-height: 297px;
  max-height: 100vh;
  position: relative;
  overflow: hidden;
  background-color: #eaeaea;

  .guspread-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;

    &[data-selectedall=true] {
      .guspread-table-root {
        background-color: #777;
        color: #fff;
      }
    }

    &::-webkit-scrollbar {
      display: none;
    }

    .guspread-table-root {
      z-index: 6;
      position: absolute;
      top: 0;
      left: 0;
      background-color: #f2f2f2;
      will-change: transform;

      &::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: -1px;
        bottom: 0;
        border-right: 1px solid #f2f2f2;
        pointer-events: none;
      }

      &[data-selectleft=true] {
        &::before {
          border-color: #dadada;
        }
      }

      &::after {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: -1px;
        border-bottom: 1px solid #f2f2f2;
        pointer-events: none;
      }

      &[data-selectabove=true] {
        &::after {
          border-color: #dadada;
        }
      }
    }

    .guspread-table {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      border-collapse: collapse;
      table-layout: fixed;
      width: fit-content;
      will-change: transform;

      &.guspread-table-header-row {
        z-index: 5;
      }

      &.guspread-table-header-col {
        z-index: 4;
      }

      &.guspread-table-body {
        z-index: 1;
      }

      thead {
        th {
          font-size: 12px;
          font-weight: bold;
          text-align: center;
          width: 150px;
          top: 0;
        }
      }

      tr {
        th {
          background-color: #f2f2f2;
          position: -webkit-sticky;
          position: sticky;
        }

        th, td {
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

      tbody {
        height: fit-content;

        tr {
          th {
            padding: 0;
            white-space: nowrap;
            overflow: hidden;
            font-size: 12px;
            -moz-user-select: none;
            -webkit-user-select: none;
            -ms-user-select: none;
            background-color: #f2f2f2;
            position: relative;
            z-index: 3;
          }

          th[data-select=true] {
            background-color: #dadada;
            color: #000;
          }

          th[data-selectabove=true]::after {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            border-bottom: 1px solid #dadada;
            pointer-events: none;
          }

          td {
            width: 146px;
            height: 27px;
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
    }

    &[data-selectedallrow=true] {
      >>> [data-selectleft=true]::before {
        border-color: #777 !important;
      }
    }

    &[data-selectedallcol=true] {
      >>> [data-selectabove=true]::after {
        border-color: #777 !important;
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

    .guspread-minimap {
      position: absolute;
      z-index: 7;
      bottom: 10px;
      right: 10px;
    }
  }

  .guspread-container[data-selectedall=true] >>> tr th, .guspread-container[data-selectedallrow=true] thead tr th[data-select=true], .guspread-container[data-selectedallcol=true] tbody >>> tr th[data-select=true] {
    background-color: #777;
    color: #fff;
  }
}
</style>
