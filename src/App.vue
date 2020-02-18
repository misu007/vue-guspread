<template>
  <div id="app">
    <div style="width: 100%; height:600px;">
      <v-guspread v-model="dataset" :fields="fields"></v-guspread>
    </div>
  </div>
</template>

<script>
import VGuspread from "./components/VueGuspread.vue";

export default {
  name: "App",
  data: () => ({
    fields: [
      { name: "c1", label: "A" },
      { name: "c2", label: "B" },
      { name: "c3", label: "C" }
    ],
    dataset: [
      { c1: "blue", c2: "banana", c3: "sky" },
      { c1: "red", c2: "apple", c3: "river" },
      { c1: "orange", c2: "orange", c3: "mountain" },
      { c1: "white", c2: "rasberry", c3: "lake" }
    ]
  }),
  components: {
    VGuspread
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
