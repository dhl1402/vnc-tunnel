<template>
  <div id="app">
    <VNCViewer client-name="test-vnc" />
  </div>
</template>

<script>
import VNCViewer from "./components/VNCViewer";

export default {
  name: "App",
  components: {
    VNCViewer,
  },
};
</script>
