<template >
  <td
    :class="`guspread-table-cell ${extraCellClass}`"
    :data-readonly="dataReadOnly"
    @mousedown.exact="$emit('trmdown')"
    @mousedown.shift.exact.stop="$emit('trmdownshift')"
    @dblclick.stop="$emit('trdblc')"
    @mouseenter="$emit('trmenter')"
    @mouseup="$emit('trmup')"
  >
    <slot name="cell" :field="field" :row="row" :col="col" :item="item" :value="value"></slot>
  </td>
</template>

<script>
export default {
  props: {
    item: {
      type: Object,
      default: {}
    },
    nameKey: {
      type: String,
      default: {}
    },
    field: {
      type: Object,
      default: {}
    },
    row: {
      type: Number,
      default: 0
    },
    col: {
      type: Number,
      default: 0
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
  computed: {
    value() {
      const field = this.field;
      const item = this.item;
      const nameKey = this.nameKey;
      if (
        field &&
        nameKey &&
        field.hasOwnProperty(nameKey) &&
        item &&
        item.hasOwnProperty(field[nameKey])
      ) {
        return item[field[nameKey]];
      }
      return null;
    },
    dataReadOnly() {
      if (this.cellReadonly && this.field && this.item) {
        return this.cellReadonly({
          field: this.field,
          item: this.item,
          row: this.row,
          col: this.col,
          value: this.value
        });
      }
      return false;
    },
    extraCellClass() {
      if (this.cellClass && this.field && this.item) {
        return this.cellClass({
          field: this.field,
          item: this.item,
          row: this.row,
          col: this.col,
          value: this.value
        }).join(" ");
      }
      return "";
    }
  }
};
</script>
<style scoped lang="stylus">
.guspread-table-cell {
  height: 27px;
  padding: 0;
  white-space: nowrap;
  overflow: hidden;
  font-size: 12px;
  -moz-user-select: none;
  -webkit-user-select: none;
  -ms-user-select: none;
  width: 146px;
  text-align: left;
  padding: 0 2px;
  position: relative;
  background-color: #fff;

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
</style>
