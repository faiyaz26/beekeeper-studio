<template>
  <div class="community-overlay">
    <span class="title">
      <i class="material-icons">stars</i> JSON Row Viewer
    </span>
    <div class="body">
      <p>
        The JSON Row Viewer is a powerful tool that allows you to view and edit
        JSON data in a structured and organized way.
      </p>
    </div>
    <upsell-buttons />
  </div>
</template>
<script lang="js">
import UpsellButtons from './common/UpsellButtons.vue';

  export default {
    components: { UpsellButtons },
    data: () => ({}),
    methods: {
      startTrial() {
        this.$store.dispatch('licenses/add', { trial: true })
      }
    }
  }

</script>
