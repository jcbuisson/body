<template>
   <AnatomyDetail></AnatomyDetail>

   <VersionUpdater></VersionUpdater>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'

import VersionUpdater from "/src/components/VersionUpdater.vue";
import AnatomyDetail from './components/AnatomyDetail.vue';

</script>
