<template>
  <div id="app">
    <div style="width: 100%; ">
      <v-guspread v-model="dataset" :fields="fields" :cellReadonly="cellReadonly"></v-guspread>
    </div>
  </div>
</template>

<script>
import VGuspread from "./components/VueGuspread.vue";

export default {
  name: "App",
  data: () => ({
    fields: [],
    dataset: [],
    cellReadonly: ({ col }) => {
      if (col == 2 || col == 4 || col == 5) {
        return true;
      }
      return false;
    }
  }),
  components: {
    VGuspread
  },
  mounted() {
    let fields = [];
    for (var i = 1; i < 400; i++) {
      fields.push({
        name: "c" + i,
        label: "c" + i
      });
    }
    let dataset = [];
    for (var i = 0; i < 400; i++) {
      let obj = {};
      fields.forEach((field, idx) => {
        obj[field.name] = "test-" + (i + 1) + "-" + (idx + 1);
      });
      dataset.push(obj);
    }
    this.fields = fields;
    this.dataset = dataset;
  }
};
</script>

<style lang="stylus">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
