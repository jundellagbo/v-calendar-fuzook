<template>
  <div>
    <div class="bg-gray-600 text-white px-2 py-3 text-center" v-if="visible">
      <p class="text font-bold mt-0 mb-0">
        This documentation applies for v1.0.0-beta.0
      </p>
      <p class="text-sm text-gray-200 my-0">
        <a href="../../../changelog/v1.0" class="underline text-gray-200"
          >See all breaking changes</a
        >, or
        <a
          href="https://vcalendar-legacy.netlify.com"
          class="underline text-gray-200"
          target="_blank"
          >reference previous documentation site</a
        >.
      </p>
    </div>
    <div>
      <v-date-picker
      is-inline 
      :value="value"
      :disabled-dates="[{ start: new Date('2019-09-11'), end: new Date('2019-09-14') }]"  
      :attributes="[{key: 'today', highlight: { backgroundColor: '#eaeaea', }, dates: new Date()}]"
      :min-date="new Date()"
      mode="range"
      />
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      visible: false,
      value: null
    };
  },
  mounted() {
    const el = document.querySelector('.hero');
    el.parentElement.insertBefore(this.$el, el);
    this.visible = true;
  },
};
</script>
