<script>
import Vue from 'vue';
import VueMathField from '@/vue-math-field.vue';

export default Vue.extend({
  name: 'ServeDev',
  components: {
    VueMathField
  }
});
</script>

<template>
  <div id="app">
    <vue-math-field />
  </div>
</template>
