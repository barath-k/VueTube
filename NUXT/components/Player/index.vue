<template>
  <!-- // TODO: down: () => minimize, -->
  <div
    ref="vidcontainer"
    v-touch="{
      up: () => (contain = false),
      down: () => (contain = true),
    }"
    class="d-flex flex-column"
    style="position: relative; background-color: black"
    :style="{ height: isFullscreen ? '100vh' : 'auto' }"
  >
    <video
      ref="player"
      autoplay
      width="100%"
      :src="vidSrc"
      :height="isFullscreen ? '100%' : 'auto'"
      style="transition: filter 0.15s ease-in-out"
      :class="controls || seeking || skipping ? 'dim' : ''"
      :style="{
        objectFit: contain ? 'contain' : 'cover',
        borderRadius:
          $store.state.tweaks.roundWatch && !isFullscreen
            ? `${$store.state.tweaks.roundTweak / 3}rem ${$store.state.tweaks.roundTweak / 3}rem 0rem 0rem !important`
            : '0',
      }"
      :poster="thumbnails[thumbnails.length - 1].url"
      @click="controlsHandler()"
      @loadedmetadata="checkDimensions()"
    />
    <!-- // NOTE: replace poster URL with "none" -->

    <!-- // TODO: merge the bottom 2 into 1 reusable component -->
    <v-btn
      text
      tile
      color="white"
      :class="skipping == -10 ? '' : 'invisible'"
      style="
        top: 0;
        left: 0;
        width: 50%;
        height: 100%;
        position: absolute;
        transition: opacity 0.15s;
        border-radius: 0 100vh 100vh 0;
        text-transform: none;
        font-size: 0.5rem;
      "
      @click="controlsHandler()"
      @dblclick="skipHandler(-10)"
    >
      <v-icon>mdi-rewind</v-icon>
      <!-- {{ skipping }} seconds -->
    </v-btn>

    <v-btn
      text
      tile
      color="white"
      :class="skipping == 10 ? '' : 'invisible'"
      style="
        top: 0;
        left: 50%;
        width: 50%;
        height: 100%;
        position: absolute;
        transition: opacity 0.15s;
        border-radius: 100vh 0 0 100vh;
        text-transform: none;
        font-size: 0.5rem;
      "
      @click="controlsHandler()"
      @dblclick="skipHandler(10)"
    >
      <v-icon>mdi-fast-forward</v-icon>
      <!-- {{ skipping }} seconds] -->
    </v-btn>

    <div
      v-if="seeking"
      class="d-flex justify-center"
      style="width: 100%; top: 0.5rem; position: absolute; font-size: 0.66rem"
    >
      <v-icon small class="pr-2">mdi-rewind</v-icon>
      Double tap left or right to skip 10 seconds
      <v-icon small class="pl-2">mdi-fast-forward</v-icon>
    </div>

    <!-- controls container -->
    <div
      style="transition: opacity 0.15s ease-in-out"
      :style="
        controls && !seeking
          ? 'opacity: 1;'
          : 'opacity: 0; pointer-events: none'
      "
    >
      <!-- top controls row -->
      <div
        style="position: absolute; width: 100%; top: 0.25rem"
        class="d-flex justify-center px-2"
      >
        <minimize />
        <div v-if="isFullscreen" class="pt-2" @click.self="controlsHandler()">
          <h4>{{ video.title }}</h4>
          <div style="color: #aaa; font-size: 0.75rem">
            {{ video.channelName }}
          </div>
        </div>
        <v-spacer />
        <captions />
        <loop
          v-if="$refs.player"
          class="mx-2"
          :loop="$refs.player.loop"
          @loop="$refs.player.loop = !$refs.player.loop"
        />
        <close />
      </div>
      <!-- top controls row end -->

      <!-- center controls row -->
      <div
        class="d-flex justify-center align-center"
        style="
          transform: translate(-50%, -50%);
          position: absolute;
          left: 50%;
          top: 50%;
        "
      >
        <v-btn fab text color="white" class="px-8" disabled>
          <v-icon size="2rem" color="white">mdi-skip-previous</v-icon>
        </v-btn>
        <playpause
          v-if="$refs.player"
          :video="$refs.player"
          @play="$refs.player.play()"
          @pause="$refs.player.pause()"
        />
        <v-btn fab text color="white" class="px-8" disabled>
          <v-icon size="2rem" color="white">mdi-skip-next</v-icon>
        </v-btn>
      </div>
      <!-- center controls row end -->

      <!-- time & fullscreen row -->
      <div
        :style="isFullscreen ? 'bottom: 4.25rem' : 'bottom: 0.5rem'"
        class="d-flex justify-between align-center pl-4 pr-2"
        style="position: absolute; width: 100%"
        @click.self="controlsHandler()"
      >
        <watchtime
          v-if="$refs.player"
          :current-time="$refs.player.currentTime"
          :duration="$refs.player.duration"
        />
        <v-spacer />
        <fullscreen
          style="z-index: 2"
          :fullscreen="isFullscreen"
          @fullscreen="fullscreenHandler(true)"
        />
      </div>
      <!-- time & fullscreen row end -->

      <!-- bottom controls row -->
      <div
        style="position: absolute; width: 100%; bottom: 0.5rem"
        class="d-flex justify-between align-center px-2"
        @click.self="controlsHandler()"
      >
        <div v-if="isFullscreen">
          <v-btn fab text small color="white" class="mr-2" disabled>
            <v-icon>mdi-thumb-up-outline</v-icon>
          </v-btn>
          <v-btn fab text small color="white" class="mr-2" disabled>
            <v-icon>mdi-thumb-down-outline</v-icon>
          </v-btn>
          <v-btn fab text small color="white" class="mr-2" disabled>
            <v-icon>mdi-share-outline</v-icon>
          </v-btn>
          <v-btn fab text small color="white" class="mr-2" disabled>
            <v-icon>mdi-plus-box-multiple-outline</v-icon>
          </v-btn>
          <v-btn fab text small color="white" class="mr-2" disabled>
            <v-icon>mdi-comment-text-outline</v-icon>
          </v-btn>
        </div>
        <v-spacer />
        <!-- // TODO: merge the bottom 2 into 1 reusable component -->
        <quality
          v-if="$refs.player"
          :sources="sources"
          style="z-index: 77777"
          :current-source="$refs.player"
          @quality="qualityHandler($event)"
        />
        <speed
          v-if="$refs.player"
          class="mx-2"
          style="z-index: 77777"
          :current-speed="$refs.player.playbackRate"
          @speed="$refs.player.playbackRate = $event"
        />
        <v-btn v-if="isFullscreen" fab text small disabled @click.stop="">
          <v-icon>mdi-cards-outline</v-icon>
        </v-btn>
        <!-- placeholder for moving fullscreen button above -->
        <v-btn v-else fab text small disabled> </v-btn>
      </div>
      <!-- bottom controls row -->
    </div>
    <!-- controls container end -->

    <progressbar
      v-if="$refs.player"
      :current-time="$refs.player.currentTime"
      :duration="$refs.player.duration"
      :fullscreen="isFullscreen"
      :controls="controls"
      :buffered="buffered"
      :seeking="seeking"
    />

    <sponsorblock
      v-if="$refs.player && blocks.length > 0"
      :duration="$refs.player.duration"
      :fullscreen="isFullscreen"
      :controls="controls"
      :seeking="seeking"
      :blocks="blocks"
    />

    <seekbar
      v-if="$refs.player"
      v-show="!isFullscreen || controls"
      :duration="$refs.player.duration"
      :fullscreen="isFullscreen"
      :current-time="progress"
      :video="$refs.player"
      :controls="controls"
      :sources="sources"
      :seeking="seeking"
      @seeking="seeking = !seeking"
      @scrub="$refs.player.currentTime = $event"
    />
  </div>
</template>

<script>
import loop from "~/components/Player/loop.vue";
import close from "~/components/Player/close.vue";
import speed from "~/components/Player/speed.vue";
import seekbar from "~/components/Player/seekbar.vue";
import quality from "~/components/Player/quality.vue";
import minimize from "~/components/Player/minimize.vue";
import captions from "~/components/Player/captions.vue";
import playpause from "~/components/Player/playpause.vue";
import watchtime from "~/components/Player/watchtime.vue";
import fullscreen from "~/components/Player/fullscreen.vue";
import progressbar from "~/components/Player/progressbar.vue";
import sponsorblock from "~/components/Player/sponsorblock.vue";
export default {
  components: {
    sponsorblock,
    progressbar,
    fullscreen,
    watchtime,
    playpause,
    captions,
    minimize,
    quality,
    seekbar,
    speed,
    close,
    loop,
  },
  props: {
    sources: {
      type: Array,
      required: true,
    },
    thumbnails: {
      type: Object,
      required: true,
    },
    video: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      isFullscreen: false,
      fullscreenLock: false,
      verticalFullscreen: false,
      midRotation: false,
      controls: false,
      seeking: false,
      contain: true,
      skipping: 0,
      progress: 0,
      buffered: 0,
      watched: 0,
      blocks: [],
      vidSrc: "",
      isVerticalVideo: false,
    };
  },
  mounted() {
    console.log("sources", this.sources);
    this.vidSrc = this.sources[this.sources.length - 1].url;
    let vid = this.$refs.player;

    this.$youtube.getSponsorBlock(this.video.id, (data) => {
      console.log("sbreturn", data);
      if (Array.isArray(data)) {
        this.blocks = data;
      }
    });

    vid.addEventListener("loadeddata", (e) => {
      // TODO: detect video loading state and send this.loading to play button :loading = loading here
      // console.log(e);
      if (vid.readyState >= 3) {
        vid.addEventListener("timeupdate", () => {
          if (!this.seeking) this.progress = vid.currentTime; // for seekbar

          // console.log("sb check", this.blocks);
          // iterate over data.segments array
          // for sponsorblock
          if (this.blocks.length > 0)
            this.blocks.forEach((sponsor) => {
              let vidTime = vid.currentTime;

              if (
                vidTime >= sponsor.segment[0] &&
                vidTime <= sponsor.segment[1]
              ) {
                console.log("Skipping the sponsor");
                this.$youtube.showToast("Skipped sponsor");
                vid.currentTime = sponsor.segment[1] + 1;
              }
            });
        });
        // TODO: detect video loading state and send this.loading to play button :loading = loading here
        vid.addEventListener("progress", () => {
          this.buffered = (vid.buffered.end(0) / vid.duration) * 100;
        });
      }
    });
  },
  created() {
    screen.orientation.addEventListener("change", () =>
      this.fullscreenHandler(false)
    );
  },
  beforeDestroy() {
    if (this.isFullscreen) this.exitFullscreen();
    screen.orientation.removeEventListener("change");
  },
  methods: {
    // TODO: make accumulative onclick after first dblclick (don't set timeout untill stopped clicking)
    skipHandler(time) {
      this.skipping = time;
      setTimeout(() => {
        this.skipping = false;
      }, 500);

      this.$refs.player.currentTime += time;
    },
    controlsHandler() {
      if (!this.seeking)
        this.controls
          ? (clearTimeout(this.controls), (this.controls = false))
          : setTimeout(() => {
              if (!this.skipping) {
                this.controls = setTimeout(() => {
                  if (!this.seeking && !this.$refs.player.paused)
                    this.controls = false;
                }, 2345);
              }
            }, 250);
    },
    qualityHandler(q) {
      console.log(q);
      let time = this.$refs.player.currentTime;
      this.$refs.player.src = q;
      this.$refs.player.currentTime = time;
    },
    checkDimensions() {
      if (this.$refs.player.videoHeight > this.$refs.player.videoWidth) {
        this.isVerticalVideo = true;
      } else {
        this.isVerticalVideo = false;
      }
    },
    fullscreenHandler(pressedFullscreenBtn) {
      // Prevent fullscreen button press from being handled twice
      // (once by pressing fullscreen button, another by the resulting rotation)
      if (this.midRotation) {
        this.midRotation = false;
        return;
      }
      // Toggle fullscreen state
      if (this.isFullscreen) {
        this.exitFullscreen(pressedFullscreenBtn);
      } else {
        this.enterFullscreen(pressedFullscreenBtn);
      }
    },
    exitFullscreen(unlock) {
      if (unlock) {
        if (this.verticalFullscreen) {
          // Unset vertical fullscreen mode
          screen.orientation.unlock();
          this.fullscreenLock = false;
          this.verticalFullscreen = false;
        } else {
          // Unset standard fullscreen mode
          this.midRotation = true;
          screen.orientation.lock("portrait");
          this.fullscreenLock = true;
          // Locks the rotation to portrait for two seconds,
          // and will then rotate back to landscape if the
          // user doesn't rotate first
          setTimeout(() => {
            this.fullscreenLock = false;
            screen.orientation.unlock();
          }, 2 * 1000);
        }
      }
      this.$vuetube.navigationBar.show();
      this.$vuetube.statusBar.show();
      this.isFullscreen = false;
    },
    enterFullscreen(force) {
      if (force) {
        this.fullscreenLock = true;
        if (this.isVerticalVideo) {
          // Vertical fullscreen mode (vertical video only)
          screen.orientation.lock("portrait");
          this.verticalFullscreen = true;
        } else {
          // Standard fullscreen mode
          this.midRotation = true;
          screen.orientation.lock("landscape");
        }
      }
      this.$vuetube.navigationBar.hide();
      this.$vuetube.statusBar.hide();
      this.isFullscreen = true;
    },
    getPlayer() {
      return this.$refs.player;
    },
  },
};
</script>

<style>
.dim {
  filter: brightness(33%);
}
.invisible {
  opacity: 0;
}
</style>
