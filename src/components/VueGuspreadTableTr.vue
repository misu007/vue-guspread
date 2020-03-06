<template >
  <tr class="guspread-item-tr">
    <th
      :data-selectabove="cursors.active && (cursors.r1 - 1) == row"
      :data-select="cursors.active && cursors.r1 <= row && cursors.r2 >= row"
      @mousedown.exact="$emit('thmdown')"
      @mousedown.shift.exact.stop="$emit('thmdownshift')"
      @mouseenter="$emit('thmenter')"
      @mouseup="$emit('thmup')"
    >{{row + 1}}</th>
    <template v-for="(cid, cidx) in visibleWorldCol">
      <v-guspread-td
        :key="'row-' + row + '-column-' + cid"
        :cellClass="cellClass"
        :cellReadonly="cellReadonly"
        :field="thisField[cidx]"
        :item="item"
        :row="row"
        :col="cid"
        :nameKey="nameKey"
        @trmdown="$emit('trmdown', {r:row, c:cid})"
        @trmdownshift="$emit('trmdownshift', {r:row, c:cid})"
        @trdblc="$emit('trdblc', {r:row, c:cid})"
        @trmenter="$emit('trmenter', {r:row, c:cid})"
        @trmup="$emit('trmup')"
      >
        <template #cell="{field, row, col, item, value}">
          <slot name="cell" :field="field" :row="row" :col="col" :item="item" :value="value"></slot>
        </template>
      </v-guspread-td>
    </template>
  </tr>
</template>

<script>
import VGuspreadTd from "@/components/VueGuspreadTableTrTd.vue";

export default {
  components: {
    VGuspreadTd
  },
  props: {
    item: {
      type: Object,
      default: {}
    },
    cursors: {
      type: Object,
      default: {}
    },
    nameKey: {
      type: String,
      default: {}
    },
    thisField: {
      type: Array,
      default: []
    },
    visibleWorldCol: {
      type: Array,
      default: []
    },
    row: {
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
  }
};
</script>
<style scoped lang="stylus">
.guspread-item-tr {
  th {
    height: 27px;
    padding: 0;
    white-space: nowrap;
    overflow: hidden;
    font-size: 12px;
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
    background-color: #f2f2f2;
    position: relative;
    width: 50px;
    z-index: 3;
  }

  th[data-select=true] {
    background-color: #dadada;
    color: #000;
  }

  th[data-selectabove=true]::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    border-bottom: 1px solid #dadada;
    pointer-events: none;
  }
}
</style>
