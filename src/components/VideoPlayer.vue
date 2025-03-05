<template>
  <div class="container mx-auto px-4 max-w-md min-h-screen flex flex-col justify-center">
    <div class="bg-black shadow-lg rounded-lg overflow-hidden">

      <!-- Sezione caricamento file -->
      <div class="p-4 bg-gradient-to-b from-gray-800 to-gray-900 rounded-lg shadow-lg">
        <input type="file" accept="video/*" @change="handleFileChange" class="hidden" ref="fileInput" />
        <button @click="triggerFileInput" class="w-full py-3 bg-blue-500 text-white rounded-md 
                 hover:bg-blue-600 transition-colors 
                 focus:outline-none focus:ring-2 focus:ring-blue-300">
          Carica un video
        </button>
      </div>

      <!-- Player video -->
      <div v-if="videoUrl" class="relative" :class="{ 'fullscreen-video-container': isFullScreen }">
        <video ref="videoPlayer" controls playsinlin webkit-playsinlinee class="w-full h-full">
          <source :src="videoUrl" type="video/mp4" />
        </video>

        <!-- Pulsanti di controllo -->
        <div class="absolute inset-0 flex justify-between items-center px-4 pointer-events-none">
          <!-- Sinistra: Start Loop & Riduci Velocità -->
          <div class="flex flex-col gap-2 pointer-events-auto">
            <button @click="setLoopStart">
              ⏮ Start Loop
            </button>
            <button @click="decreasePlaybackSpeed">
              ⏬ Velocità
            </button>
          </div>

          <!-- Destra: End Loop & Aumenta Velocità -->
          <div class="flex flex-col gap-2 pointer-events-auto">
            <button @click="setLoopEnd">
              ⏭ End Loop
            </button>
            <button @click="increasePlaybackSpeed">
              ⏫ Velocità
            </button>
          </div>

          <!-- Pulsante disattiva loop -->
          <div class="absolute bottom-16 right-4 pointer-events-auto">
            <button @click="disableLoop" class="bg-red-600 text-white px-3 py-2 rounded-md text-sm shadow-md hover:bg-red-700 
                           transition-opacity opacity-80 hover:opacity-100">
              ❌ Loop Off
            </button>
          </div>
        </div>

        <!-- Pulsante full screen -->
        <div class="absolute bottom-16 left-4 pointer-events-auto">
          <button @click="toggleFullScreen" class="bg-green-600 text-white px-3 py-2 rounded-md text-sm shadow-md hover:bg-green-700 
                         transition-opacity opacity-80 hover:opacity-100">
            🖥 Full Screen
          </button>
        </div>

        <!-- Hint gesture -->
        <div v-if="gestureHint" class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2
                  bg-black bg-opacity-70 text-white px-4 py-2 rounded-md">
          {{ gestureHint }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Hammer from 'hammerjs';

export default {
  data() {
    return {
      loopMode: false,
      loopStart: null,
      loopEnd: null,
      videoUrl: null,
      gestureHint: '',
      gestureTimeout: null,
      hammertime: null,
      playbackRate: 1,
      isFullScreen: false
    };
  },
  mounted() {
    window.addEventListener('fullscreenchange', this.handleFullscreenChange);
  },
  beforeUnmount() {
    window.removeEventListener('fullscreenchange', this.handleFullscreenChange);
    if (this.hammertime) {
      this.hammertime.destroy();
    }
  },
  methods: {
    triggerFileInput() {
      this.$refs.fileInput.click();
    },

    handleFileChange(event) {
      const file = event.target.files[0];
      if (file && file.type.startsWith('video/')) {
        this.videoUrl = URL.createObjectURL(file);
      } else {
        alert('Seleziona un file video valido.');
      }
    },

    toggleFullScreen() {
      const videoContainer = this.$refs.videoPlayer.parentElement;
      if (!this.isFullScreen) {
        if (videoContainer.requestFullscreen) {
          videoContainer.requestFullscreen();
        } else if (videoContainer.mozRequestFullScreen) { // Firefox
          videoContainer.mozRequestFullScreen();
        } else if (videoContainer.webkitRequestFullscreen) { // Chrome, Safari and Opera
          videoContainer.webkitRequestFullscreen();
        } else if (videoContainer.msRequestFullscreen) { // IE/Edge
          videoContainer.msRequestFullscreen();
        }
      } else if (document.exitFullscreen) {
        document.exitFullscreen();
      } else if (document.mozCancelFullScreen) { // Firefox
        document.mozCancelFullScreen();
      } else if (document.webkitExitFullscreen) { // Chrome, Safari and Opera
        document.webkitExitFullscreen();
      } else if (document.msExitFullscreen) { // IE/Edge
        document.msExitFullscreen();
      }
    },

    handleFullscreenChange() {
      this.isFullScreen = !!document.fullscreenElement;
    },

    setupFullscreenGestures() {
      const video = this.$refs.videoPlayer;

      // Rimuovi eventuale Hammer precedente
      if (this.hammertime) {
        this.hammertime.destroy();
      }

      if (document.fullscreenElement) {
        // Inizializza Hammer.js
        this.hammertime = new Hammer(video);

        // Abilita tap e swipe
        this.hammertime.get('tap').set({
          taps: 1,
          interval: 300,
          threshold: 10,
          posThreshold: 20
        });

        // Abilita swipe
        this.hammertime.get('swipe').set({
          direction: Hammer.DIRECTION_ALL
        });

        // Aggiungi listener per gesture
        this.hammertime.on('tap', this.handleFullscreenTap);
        this.hammertime.on('swipeleft', this.decreasePlaybackSpeed);
        this.hammertime.on('swiperight', this.increasePlaybackSpeed);
        this.hammertime.on('swipedown', this.disableLoop);
      }
    },

    handleFullscreenTap(event) {
      const video = this.$refs.videoPlayer;
      const videoWidth = video.clientWidth;
      const tapX = event.center.x - video.getBoundingClientRect().left;

      // Cancella timeout precedente per l'hint
      if (this.gestureTimeout) {
        clearTimeout(this.gestureTimeout);
      }

      // Lato sinistro: imposta inizio loop
      if (tapX < videoWidth / 2) {
        this.loopStart = video.currentTime;
        this.gestureHint = `Loop Start: ${this.formatTime(this.loopStart)}`;
      }
      // Lato destro: imposta fine loop
      else {
        this.loopEnd = video.currentTime;
        this.gestureHint = `Loop End: ${this.formatTime(this.loopEnd)}`;
      }

      // Se entrambi i punti sono impostati, attiva loop
      if (this.loopStart !== null && this.loopEnd !== null) {
        this.loopMode = true;
        video.addEventListener('timeupdate', this.handleLoop);
        this.gestureHint = 'Loop Attivato!';
      }

      // Rimuovi hint dopo 2 secondi
      this.gestureTimeout = setTimeout(() => {
        this.gestureHint = '';
      }, 2000);
    },

    increasePlaybackSpeed() {
      if (this.playbackRate < 2) {
        this.playbackRate = Math.min(this.playbackRate + 0.25, 2);
        this.$refs.videoPlayer.playbackRate = this.playbackRate;
        this.showGestureHint(`Velocità: ${this.playbackRate}x`);
      }
    },

    decreasePlaybackSpeed() {
      if (this.playbackRate > 0.25) {
        this.playbackRate = Math.max(this.playbackRate - 0.25, 0.25);
        this.$refs.videoPlayer.playbackRate = this.playbackRate;
        this.showGestureHint(`Velocità: ${this.playbackRate}x`);
      }
    },

    disableLoop() {
      const video = this.$refs.videoPlayer;
      this.loopMode = false;
      this.loopStart = null;
      this.loopEnd = null;
      video.removeEventListener('timeupdate', this.handleLoop);
      this.showGestureHint('Loop Disattivato');
    },

    handleLoop() {
      const video = this.$refs.videoPlayer;
      if (this.loopMode && video.currentTime >= this.loopEnd) {
        video.currentTime = this.loopStart;
      }
    },

    showGestureHint(message) {
      this.gestureHint = message;

      // Cancella timeout precedente per l'hint
      if (this.gestureTimeout) {
        clearTimeout(this.gestureTimeout);
      }

      // Rimuovi hint dopo 2 secondi
      this.gestureTimeout = setTimeout(() => {
        this.gestureHint = '';
      }, 2000);
    },

    formatTime(time) {
      const minutes = Math.floor(time / 60);
      const seconds = Math.floor(time % 60);
      return `${minutes}:${seconds.toString().padStart(2, '0')}`;
    },
    setLoopStart() {
      this.loopStart = this.$refs.videoPlayer.currentTime;
      this.showGestureHint(`Loop Start: ${this.formatTime(this.loopStart)}`);
    },

    setLoopEnd() {
      this.loopEnd = this.$refs.videoPlayer.currentTime;
      this.showGestureHint(`Loop End: ${this.formatTime(this.loopEnd)}`);

      if (this.loopStart !== null && this.loopEnd !== null) {
        this.loopMode = true;
        this.$refs.videoPlayer.addEventListener('timeupdate', this.handleLoop);
        this.showGestureHint('Loop Attivato!');
      }
    }
  }
};
</script>

<style scoped>
.fullscreen-video-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: black;
  z-index: 1000;
  display: flex;
  justify-content: center;
  align-items: center;
}

.fullscreen-video-container video {
  width: 100%;
  height: 100%;
  object-fit: contain;
  /* Mantiene le proporzioni del video */
}
</style>