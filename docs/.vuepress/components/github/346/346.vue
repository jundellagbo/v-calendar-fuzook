<template>
  <v-calendar :rows="2" :columns="2" />
  <!-- <v-date-picker
    v-model="date"
    popover-visibility="visible"
    is-inline
    mode="range"
  /> -->
</template>

<script>
export default {
  data() {
    return {
      date: null,
    };
  },
};
</script>
