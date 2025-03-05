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
      <div v-if="videoUrl" class="relative">
        <video ref="videoPlayer" class="" controls @fullscreenchange="handleFullscreenChange">
          <source :src="videoUrl" type="video/mp4" />
        </video>

        <!-- Pulsanti di controllo (visibili solo su schermi ≥ 768px) -->
        <div class="absolute inset-0 hidden md:flex justify-between items-center px-4 pointer-events-none">
          <!-- Sinistra: Start Loop & Riduci Velocità -->
          <div class="flex flex-col gap-2 pointer-events-auto">
            <button @click="setLoopStart" class="video-control-btn">
              ⏮ Start Loop
            </button>
            <button @click="decreasePlaybackSpeed" class="video-control-btn">
              ⏬ Velocità
            </button>
          </div>

          <!-- Destra: End Loop & Aumenta Velocità -->
          <div class="flex flex-col gap-2 pointer-events-auto">
            <button @click="setLoopEnd" class="video-control-btn">
              ⏭ End Loop
            </button>
            <button @click="increasePlaybackSpeed" class="video-control-btn">
              ⏫ Velocità
            </button>
          </div>
          <!-- Pulsante disattiva loop (visibile solo su PC) -->
          <div :class="fullscreen ? 'absolute top-4 right-4' : 'absolute bottom-16 right-4'">
            <button @click="disableLoop" class="bg-red-600 text-white px-3 py-2 rounded-md text-sm shadow-md hover:bg-red-700 
                           transition-opacity opacity-80 hover:opacity-100 pointer-events-auto">
              ❌ Loop Off
            </button>
          </div>  
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
      isFullscreen: false
    };
  },
  mounted() {
    window.addEventListener('fullscreenchange', this.setupFullscreenGestures);
  },
  beforeUnmount() {
    window.removeEventListener('fullscreenchange', this.setupFullscreenGestures);
    if (this.hammertime) {
      this.hammertime.destroy();
    }
  },
  methods: {
    toggleFullscreen() {
      this.isFullscreen = !this.isFullscreen;

      const videoContainer = this.$el.querySelector('.container');
      const video = this.$refs.videoPlayer;

      if (this.isFullscreen) {
        // Abilitare fullscreen personalizzato
        videoContainer.style.position = 'fixed';
        videoContainer.style.top = 0;
        videoContainer.style.left = 0;
        videoContainer.style.width = '100%';
        videoContainer.style.height = '100%';
        video.style.objectFit = 'cover'; // Copre tutto lo schermo
      } else {
        // Disabilitare fullscreen personalizzato
        videoContainer.style.position = 'relative';
        videoContainer.style.width = 'auto';
        videoContainer.style.height = 'auto';
        video.style.objectFit = 'contain'; // Torna alla dimensione originale
      }
    },
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

    handleVideoPause() {
      // Quando il video viene messo in pausa, disattiva il loop
      const video = this.$refs.videoPlayer;
      this.loopMode = false;
      this.loopStart = null;
      this.loopEnd = null;
      video.removeEventListener('timeupdate', this.handleLoop);
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
    },
    handleFullscreenChange() {
      // Verifica se il video è in modalità schermo intero
      this.fullscreen = document.fullscreenElement !== null;
    }
  }
};
</script>