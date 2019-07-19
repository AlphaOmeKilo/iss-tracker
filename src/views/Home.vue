<template>
  <div class="home">
    <img alt="Vue logo" src="../assets/logo.png">
    <Globe />
  </div>
</template>

<script>
// @ is an alias to /src
import Globe from '@/components/Globe.vue'

export default {
  name: 'home',
  components: {
    Globe
  }
}
</script>
