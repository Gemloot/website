<script setup>
import { ref } from 'vue'

const { item } = defineProps({
  item: Object
})

</script>

<template>
  <div>
    <p>{{item.json.name}}</p>
    <img :src="item.json.image" class="gem">
  </div>
</template>

<style scoped src="./index.less">
</style>
