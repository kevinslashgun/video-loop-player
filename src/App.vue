<template>
  <div id="app" class="min-h-screen bg-gradient-to-b from-gray-900 to-black text-white antialiased">
    <VideoPlayer />
  </div>
</template>

<script>
import VideoPlayer from './components/VideoPlayer.vue';

export default {
  name: 'App',
  components: {
    VideoPlayer,
  },
};
</script>