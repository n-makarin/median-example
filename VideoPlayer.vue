<template>
  <div :class="['video-player', className]">
    <video ref="video" :class="classPlayer" class="video-js vjs-skin" />
    <Icons />

    <transition name="fade">
      <div>
        <Cover v-if="disablePlaying" :darkened="darkenedCover" :picture="poster" />
        <Cover
          v-else-if="useCover && playButtonText && isLoading && !momentsPlaylist"
          :buttonText="playButtonText"
          :darkened="darkenedCover"
          :picture="poster"
          :subPlayButtonText="subPlayButtonText"
          @buttonOnClick="onClickWatch"
        />
        <Cover v-else-if="useCover && showPoster" :layerPosition="0" :picture="poster" />
      </div>
    </transition>

    <div class="video-player__slot">
      <Cover
        v-if="isError"
        :darkened="isFullscreen"
        :picture="poster"
        :variant="isFullscreen ? 'error-btn' : 'error'"
        @hideError="onHideFullscreenError"
      />

      <slot />
    </div>
  </div>
</template>

<script lang="ts">
  import '~/plugins/video-player/middlewares/virtual-offset';
  import '~/plugins/video-player/techs/hls-tech';

  import { computed, defineComponent, onMounted, onUnmounted, ref, watch } from '@nuxtjs/composition-api';
  import videojs from 'video.js';

  import Cover from '~/components/VideoPlayer/Cover.vue';
  import additionalEvents from '~/components/VideoPlayer/events/additional';
  import mainEvents from '~/components/VideoPlayer/events/main';
  import Icons from '~/components/VideoPlayer/Icons.vue';
  import Props from '~/components/VideoPlayer/props';
  import { useSafePlayback } from '~/composition';
  import { useControlTitlesPlugin } from '~/composition/plugins/video-player/useControlTitlesPlugin';
  import { useDefaultSlotPlugin } from '~/composition/plugins/video-player/useDefaultSlotPlugin';
  import { useDescriptionPlugin } from '~/composition/plugins/video-player/useDescriptionPlugin';
  import { useEmitPlugin } from '~/composition/plugins/video-player/useEmitPlugin';
  import { useIconsPlugin } from '~/composition/plugins/video-player/useIconsPlugin';
  import { useLivePlugin } from '~/composition/plugins/video-player/useLivePlugin';
  import { useNextContentPlugin } from '~/composition/plugins/video-player/useNextContentPlugin';
  import { usePlayPausePlugin } from '~/composition/plugins/video-player/usePlayPausePlugin';
  import { useRewindPlugin } from '~/composition/plugins/video-player/useRewindPlugin';
  import { useSettingsPlugin } from '~/composition/plugins/video-player/useSettingsPlugin';
  import { useSpinner } from '~/composition/plugins/video-player/useSpinner';
  import { useMomentOffset } from '~/composition/useMomentOffset';
  import { cn, Events, PlayerWidth } from '~/const/video-player';
  import { useVideoJSError } from '~/plugins/video-player/hooks/error';
  import { Events as NextContentEvents } from '~/plugins/video-player/modules/next-content/const';
  import { Events as RewindEvents } from '~/plugins/video-player/modules/rewind/const';
  import { IPLuginEvent, IVideoJsPlayer, IVideoPlayerOptions } from '~/types/video-player';

  export default defineComponent({
    components: {
      Icons,
      Cover,
    },
    props: { ...Props },
    setup(props, { emit, root }) {
      const player = ref<IVideoJsPlayer | null>(null);
      const video = ref<Element>();
      const isResized = ref(false);
      const isEventsBind = ref(false);
      const playerWidth = ref<number>(0);
      const isLoading = ref(true);

      const bindMainEvents = mainEvents();
      const bindAdditionalEvents = additionalEvents();

      const { isError, initErrorHook, removeErrorHook } = useVideoJSError();
      const { livePlugin, registerLivePlugin } = useLivePlugin(player);
      const { emitPlugin, registerEmitPlugin } = useEmitPlugin(player);
      const { iconsPlugin, registerIconsPlugin } = useIconsPlugin(player);
      const { rewindPlugin, registerRewindPlugin } = useRewindPlugin(player);
      const { settingsPlugin, registerSettingsPlugin } = useSettingsPlugin(player);
      const { defaultSlotPlugin, registerDefaultSlot } = useDefaultSlotPlugin(player);
      const { playPausePlugin, registerPlayPausePlugin } = usePlayPausePlugin(player);
      const { nextContentPlugin, registerNextContentPlugin } = useNextContentPlugin(player);
      const { descriptionPlugin, registerDescriptionPlugin } = useDescriptionPlugin(player);
      const { controlTitlesPlugin, registerControlTitlesPlugin } = useControlTitlesPlugin(player);

      const { play, pause, completeAllPlayPromises } = useSafePlayback();
      const { setSpinner, showSpinner, hideSpinner } = useSpinner(player);

      const showPoster = computed(() => isLoading.value || !props.src);
      const isFullscreen = computed(() => !!player.value?.isFullscreen());

      const startPosition = computed(() => {
        const { getStartOffset, getQualityLevel } = useMomentOffset();
        const qualityLevel = getQualityLevel(player.value!);
        return getStartOffset(props.metadata, qualityLevel, props.startOffset);
      });

      const options = computed<IVideoPlayerOptions>(
        (): IVideoPlayerOptions => ({
          // Standard <video> Element Options
          sources: [{ type: props.type, src: props.src }],
          controls: props.showControls,
          autoplay: props.autoplay,
          width: props.width,
          height: props.height,
          loop: props.loop,
          muted: props.muted,
          preload: props.preload as any,
          poster: props.poster,
          errorDisplay: props.errorDisplay,
          html5: {
            vhs: {
              blacklistDuration: Infinity,
              enableLowInitialPlaylist: props.enableLowInitialPlaylist,
              maxPlaylistRetries: props.maxPlaylistRetries,
            },
            // VIRTUAL OFFSET Video.js middleware options
            offset: {
              start: startPosition.value,
              duration: props.duration,
              isMomentPlaylist: props.momentsPlaylist,
            },
            livePlaylist: props.livePlaylist,
          },

          // Video.js-specific Options
          aspectRatio: props.aspectRatio,
          fluid: props.fluid,
          inactivityTimeout: props.inactivityTimeout,
          language: props.language,
          languages: props.languages,
          liveui: props.liveui,
          notSupportedMessage: props.mediaSrcNotSupportedMsg,
          playbackRates: props.playbackRates,
          plugins: props.plugins,
          techCanOverridePoster: props.techCanOverridePoster,
          userActions: props.userActions,
          playsinline: props.playsInline,
        }),
      );

      const className = computed(() => [
        'video-player',
        {
          'video-player_ready': isResized,
          'video-player--disabled': props.disablePlaying,
          [cn.playerLg]: playerWidth.value >= PlayerWidth.Lg,
          [cn.playerMd]: playerWidth.value >= PlayerWidth.Md && playerWidth.value < PlayerWidth.Lg,
          [cn.playerSm]: playerWidth.value < PlayerWidth.Md,
          [cn.subHidden]: !props.subtitlesHidden,
        },
      ]);

      watch(
        () => props.width,
        (value?: number) => {
          if (!player.value || !value) return;
          player.value.width(value);
        },
        { immediate: true },
      );

      watch(
        () => props.height,
        (value?: number) => {
          if (!player.value || !value) return;
          player.value.height(value);
        },
        { immediate: true },
      );

      watch(
        () => props.playing,
        (value: boolean) => {
          if (!player.value) return;

          value ? play(player.value) : pause(player.value);
        },
      );

      watch(
        () => props.title,
        () => {
          descriptionPlugin.value?.updateOptions({ title: props.title });
        },
      );

      watch(
        () => props.subtitle,
        () => {
          descriptionPlugin.value?.updateOptions({ subtitle: props.subtitle });
        },
      );

      const onPlayNext = (_event: IPLuginEvent) => {
        emit(NextContentEvents.PlayNext, nextContentPlugin.value?.nextContent);
      };

      const onBackward = (_event: IPLuginEvent) => {
        emit(RewindEvents.Backward);
      };

      const onForward = (_event: IPLuginEvent) => {
        emit(RewindEvents.Forward);
      };

      const onPlayerResize = () => {
        playerWidth.value = player.value?.el().clientWidth || 0;
      };

      const onLoadStart = () => {
        isLoading.value = true;
        if (props.showPreloadSpinner) showSpinner();
      };

      const onClickWatch = () => {
        player.value?.focus();
        emit('clickWatch');
      };

      const onPlaying = () => {
        isLoading.value = false;
        isError.value = false;
        hideSpinner();
      };

      const onError = () => {
        isError.value = true;
      };

      const onClickPlayer = (event: MouseEvent) => {
        emit('onClickPlayer', event);
      };

      const onHideFullscreenError = () => {
        player.value?.exitFullscreen();
      };

      const onPlayerReady = () => {
        emit(Events.Ready, player.value);
        isResized.value = true;
        setSpinner(root.$el);

        if (window) {
          // @ts-ignore
          window.VIDEOJS_NO_DYNAMIC_STYLE = true;
        }
      };

      const bindPlayerEvents = () => {
        player.value?.on(Events.LoadStart, onLoadStart);
        player.value?.on(Events.PlayerResize, onPlayerResize);
        player.value?.on(Events.Playing, onPlaying);
        player.value?.on(Events.Click, onClickPlayer);
        player.value?.on(Events.Error, onError);

        if (!isEventsBind.value) {
          isEventsBind.value = true;
          bindMainEvents(player.value, isResized.value, emit);
          bindAdditionalEvents(player.value, emit);
        }
      };

      const bindPluginsEvents = () => {
        if (!emitPlugin.value) return;

        emitPlugin.value.on(NextContentEvents.PlayNext, onPlayNext);
        emitPlugin.value.on(RewindEvents.Backward, onBackward);
        emitPlugin.value.on(RewindEvents.Forward, onForward);
      };

      const registerPlugins = (): void => {
        registerEmitPlugin(() => bindPluginsEvents());
        registerNextContentPlugin(!props.useNextContent, props.contentType);
        registerPlayPausePlugin(props.playButtonIsIcon);
        registerDefaultSlot();
        registerSettingsPlugin();
        registerDescriptionPlugin(props.title, props.subtitle, props.descriptionPluginWindowed);
        registerControlTitlesPlugin();
        registerRewindPlugin();
        registerLivePlugin();
        // @NOTE: important to be last, because
        // plugins above can add custom controls
        registerIconsPlugin();
      };

      const setStartPosition = (startPosition: number) => {
        player.value?.currentTime(startPosition);
      };

      const initPlayer = (): void => {
        if (!video.value) return;

        if (!player.value) {
          player.value = videojs(video.value, options.value, onPlayerReady);
          props.startOffset && setStartPosition(props.startOffset);
          return;
        }

        player.value.options(options.value);
        player.value.poster(props.poster);
        player.value.src(props.src);
        player.value.ready(onPlayerReady);
        player.value.trigger(Events.LoadStart);
      };

      const init = (): void => {
        if (!props.src) return;

        initErrorHook();
        initPlayer();
        bindPlayerEvents();
        registerPlugins();
      };

      const dispose = () => {
        isEventsBind.value = false;

        removeErrorHook();
        livePlugin.value?.dispose();
        rewindPlugin.value?.dispose();
        emitPlugin.value?.dispose();
        nextContentPlugin.value?.dispose();
        playPausePlugin.value?.dispose();
        iconsPlugin.value?.dispose();
        settingsPlugin.value?.dispose();
        descriptionPlugin.value?.dispose();
        controlTitlesPlugin.value?.dispose();
        defaultSlotPlugin.value?.dispose();

        // @NOTE: dispose player
        player.value?.pause();
        player.value?.off();
        player.value?.dispose();
      };

      onMounted(() => init());

      onUnmounted(async () => {
        if (process.client) {
          await completeAllPlayPromises();
        }
        dispose();
      });

      watch(
        () => props.src,
        () => {
          if (!props.src) {
            pause(player.value!);
            return;
          }
          isError.value = false;
          init();
        },
      );

      return {
        options,
        player,
        video,
        isResized,
        className,
        isLoading,
        showPoster,
        isError,
        isFullscreen,
        onHideFullscreenError,
        onClickWatch,
      };
    },
  });
</script>

<style lang="scss">
  @import '~@/assets/video-player';
  @import '~@/assets/plugins/video-player/moments';
  @import '~@/assets/plugins/video-player/next-content';
  @import '~@/assets/plugins/video-player/play-pause';
  @import '~@/assets/plugins/video-player/icons';
  @import '~@/assets/plugins/video-player/settings';
  @import '~@/assets/plugins/video-player/description';
  @import '~@/assets/plugins/video-player/live';

  .video-player {
    position: relative;
    // @NOTE: show border-radios on iOS
    transform: translateZ(0);

    &--disabled {
      .video-js .vjs-big-play-button {
        display: none;
      }
    }
  }

  .fade-enter-active,
  .fade-leave-active {
    transition: opacity 0.15s;
  }

  .fade-enter,
  .fade-leave-to {
    opacity: 0;
  }

  .vjs__cover {
    overflow: hidden;
    border-radius: 20px;
  }
</style>
