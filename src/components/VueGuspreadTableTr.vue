<template>
  <tr class="guspread-item-tr">
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

