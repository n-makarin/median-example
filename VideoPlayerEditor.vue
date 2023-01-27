<template>
  <div class="video-player-editor">
    <VideoPlayer
      v-bind="{ ...$props, ...$attrs }"
      :disablePlaying="!!expiredInfo || disablePlaying"
      v-on="$listeners"
      @timeupdate="onTimeUpdate"
      @ready="playerReady"
      @play="play"
      @pause="pause"
      @loadedmetadata="onLoadedMetadata"
      @fullscreenchange="onFullscreenChange"
      @click.native="playerOnClick"
    />
    <div v-show="isFullscreen" ref="editor" class="editor">
      <TimelineRange
        v-if="step === Step.Editing"
        class="timeline-range"
        :progress="progress"
        :totalDuration="videoDuration"
        :maxRangeDuration="maxRangeDuration"
        :twoPoints="twoPointsRange"
        @start="(val) => (rangeStart = val)"
        @end="(val) => (rangeEnd = val)"
        @click.native="onboardingCreateKinomTooltipVisible = false"
        @touchmove="onboardingCreateKinomTooltipVisible = false"
      >
        <template slot="start-point">
          <div id="start-point" />

          <v-popover
            trigger="manual"
            :open="onboardingCreateKinomTooltipVisible"
            :offset="16"
            :arrowPadding="8"
            placement="bottom-start"
            container="#start-point"
            :autoHide="false"
          >
            <template slot="popover">
              <span class="tooltip__title">Как обрезать фильм?</span>
              <span class="tooltip__content">
                Перетащите ползунок, где будет начинаться кином. Длина кинома 30 секунд
              </span>
            </template>
          </v-popover>
        </template>
      </TimelineRange>

      <div v-show="videoHasStarted && step !== Step.Preview" class="buttons">
        <Button
          v-show="step === Step.Watching || step === Step.Finished"
          id="create-kinom"
          class="button secondary"
          @click="createKinomButtonClick"
        >
          Создать кином
          <div id="tooltip-error-container" />
        </Button>

        <template v-if="step === Step.Editing">
          <Button class="button secondary" @click="step = Step.Watching">Отмена</Button>

          <Button class="button primary btn-continue" @click="proceedOnClick">Продолжить</Button>
        </template>

        <v-popover
          trigger="manual"
          :open="onboardingFullscreenTooltipVisible"
          :offset="-10"
          placement="top-start"
          container="#create-kinom"
          :autoHide="false"
        >
          <template slot="popover">
            <span class="tooltip__title h6">Нравится момент?</span>
            <span class="tooltip__content body-2">Сохраните и поделитесь киномом с друзьми</span>
          </template>
        </v-popover>

        <v-popover
          trigger="manual"
          :open="!!errorText"
          placement="bottom-start"
          container="#tooltip-error-container"
          :autoHide="false"
        >
          <template slot="popover">
            <span class="tooltip__content">{{ errorText }}</span>
          </template>
        </v-popover>
      </div>

      <Modal v-if="step === Step.Preview" :showActions="false" class="preview">
        <img src="icons/close.svg" alt="close" class="popup__close" @click="step = Step.Editing" />

        <div v-if="title" class="preview__title">{{ title }} — новый кином</div>

        <VideoPlayer
          class="preview__player"
          v-bind="{ ...$props, ...$attrs }"
          :src="src"
          :useCover="false"
          :startOffset="rangeStart"
          :showControls="false"
          playButtonIsIcon
          @ready="previewOnReady"
          @timeupdate="previewOnTimeUpdate"
          @touchstart.native="previewOnTouchstart"
          @play="previewOnPlay"
        >
          <div class="preview__duration">
            {{ getTime((rangeStart ? rangeDuration : maxRangeDuration) - previewViewdTime) }}
          </div>
        </VideoPlayer>

        <div class="preview__buttons">
          <Button class="button primary" @click="createKinom">Сохранить</Button>

          <Button class="button secondary" @click="step = Step.Editing">Отменить</Button>
        </div>
      </Modal>

      <div v-if="step === Step.Finished" class="kinom-created popup">
        <img src="icons/close.svg" alt="close" class="popup__close" @click="step = Step.Watching" />

        <div class="title h5">Вы создали кином!</div>

        <div class="description body-2">
          Через несколько минут
          <br />
          он появится в «Моей коллекции»
        </div>

        <Button class="button primary subtitle-2" @click="goToMyCollection">Перейти в Мою коллекцию</Button>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
  import dayjs from '@dayjs';
  import { computed, defineComponent, PropType, ref, useContext, useRouter, watch } from '@nuxtjs/composition-api';
  import videojs from 'video.js';

  import { Button, Modal, TimelineRange, VideoPlayer } from '~/components';
  import { HTTPStatusCodes } from '~/const/errors';
  import { ErrorText, IPublishResponse } from '~/const/kinom-crop';
  import { Routes } from '~/const/routes';
  import { SecondsToMs } from '~/const/time';
  import { cookies } from '~/plugins/utils/cookies';
  import { cn as playPauseCn } from '~/plugins/video-player/modules/play-pause';

  enum Step {
    Watching = 'watching',
    Editing = 'editing',
    Preview = 'preview',
    Finished = 'finished',
  }

  const ONBOARDING_FULLSCREEN_MAX_NUMBER = 2;
  const ONBOARDING_CRATE_KINOM_MAX_NUMBER = 3;

  export default defineComponent({
    components: {
      VideoPlayer,
      TimelineRange,
      Button,
      Modal,
    },

    props: {
      title: {
        type: String,
        default: '',
      },
      src: {
        type: String,
        default: '',
      },
      expiredInfo: {
        type: String,
        default: '',
      },
      maxRangeDuration: {
        type: Number,
        default: 30, // seconds
      },
      twoPointsRange: {
        type: Boolean,
        default: false,
      },
      showTooltips: {
        type: Boolean,
        default: false,
      },
      publishResponse: {
        type: Array as PropType<IPublishResponse[]>,
        default: () => [],
      },
      canCreateKinom: {
        type: Boolean,
        required: true,
      },
      disablePlaying: {
        type: Boolean,
        default: false,
      },
    },

    setup(props, { emit }) {
      const progress = ref(0);
      const player = ref<videojs.Player | null>(null);
      const videoDuration = ref(0);
      const videoHasStarted = ref(false);
      const step = ref<Step>(Step.Watching);
      const rangeStart = ref(0);
      const rangeEnd = ref(0);
      const editor = ref();
      const isFullscreen = ref(false);
      const previewViewdTime = ref(0);
      const previewTimer = ref(0);
      const router = useRouter();

      const onboardingFullscreenTooltipVisible = ref(false);
      const onboardingCreateKinomTooltipVisible = ref(false);

      const errorText = ref('');
      const errorTimeout = ref(0);
      const stepWatchingTimeout = ref(0);

      const { $cookies } = useContext();

      const onboardingFullscreenTooltipShowed = ref(false);
      const previewEnded = ref(false);

      const previewPlayer = ref<videojs.Player | null>(null);
      const previewPauseRequested = ref(false);

      const needShowOnboardingFullscreen = () => {
        const impressionsNumber = Number($cookies.get(cookies.videoOnboardingFullscreenCounter.name) | 0);
        return impressionsNumber < ONBOARDING_FULLSCREEN_MAX_NUMBER;
      };

      const needShowOnboardingCreateKinom = (prevValue: Step) => {
        if (prevValue === Step.Preview) return false;
        const impressionsNumber = Number($cookies.get(cookies.videoOnboardingCreateKinomCounter.name) | 0);
        return impressionsNumber < ONBOARDING_CRATE_KINOM_MAX_NUMBER;
      };

      const rangeDuration = computed(() => {
        return rangeEnd.value - rangeStart.value;
      });

      const onTimeUpdate = (val: number) => {
        progress.value = val;
      };

      const replaceEditorToPlayerContainer = () => {
        player.value?.el().appendChild(editor.value);
      };

      const playerReady = (playerInstance: videojs.Player) => {
        player.value = playerInstance;
        replaceEditorToPlayerContainer();
      };

      const onLoadedMetadata = () => {
        if (!player.value) return;
        videoDuration.value = player.value.duration();
      };

      const play = () => {
        videoHasStarted.value = true;

        if (step.value === Step.Finished) {
          step.value = Step.Watching;
        }
      };

      let previewTime = 0;

      const setPreviewTimer = (time: number) => {
        if (previewTime === time) {
          return;
        }
        previewTime = time;
        previewTimer.value++;
        previewViewdTime.value = previewTimer.value - 1;
      };

      const showPreviewwPlayIcon = (playerInstance: videojs.Player) => {
        const playerEl = playerInstance.el();
        const playButton = playerEl.querySelector(`.${playPauseCn.playButton}`);
        playButton?.classList.remove(playPauseCn.paused);
        playButton?.classList.add(playPauseCn.visible);
      };

      const previewOnTimeUpdate = (val: number, playerInstance: videojs.Player) => {
        if (val >= rangeEnd.value) {
          playerInstance.pause();
          previewEnded.value = true;
          showPreviewwPlayIcon(playerInstance);
        }

        setPreviewTimer(val);
      };

      const createKinom = () => {
        emit('createKinom', rangeStart.value, rangeEnd.value);
      };

      const toggleControls = () => {
        if (isFullscreen.value && step.value === Step.Editing) {
          player.value?.controls(false);
        } else {
          player.value?.controls(true);
        }
      };

      const clearModalsTimeouts = () => {
        clearTimeout(errorTimeout.value);
        clearTimeout(stepWatchingTimeout.value);
      };

      const increaseOnboardingFullscreenCounter = () => {
        const impressionsNumber = Number($cookies.get(cookies.videoOnboardingFullscreenCounter.name) | 0);
        $cookies.set(cookies.videoOnboardingFullscreenCounter.name, impressionsNumber + 1);
      };

      const increaseOnboardingCreateKinomCounter = () => {
        const impressionsNumber = Number($cookies.get(cookies.videoOnboardingCreateKinomCounter.name) | 0);
        $cookies.set(cookies.videoOnboardingCreateKinomCounter.name, impressionsNumber + 1);
      };

      const showOnboardingFullscreenTooltip = () => {
        onboardingFullscreenTooltipVisible.value = true;

        increaseOnboardingFullscreenCounter();
        onboardingFullscreenTooltipShowed.value = true;

        setTimeout(() => {
          onboardingFullscreenTooltipVisible.value = false;
        }, SecondsToMs.Ten);
      };

      const hideTooltips = () => {
        onboardingFullscreenTooltipVisible.value = false;
        onboardingFullscreenTooltipShowed.value = false;
      };

      const onFullscreenChange = (playerInstance: videojs.Player) => {
        isFullscreen.value = playerInstance.isFullscreen();

        if (
          isFullscreen.value &&
          props.showTooltips &&
          needShowOnboardingFullscreen() &&
          !onboardingFullscreenTooltipShowed.value
        ) {
          showOnboardingFullscreenTooltip();
        } else {
          hideTooltips();
        }

        step.value = Step.Watching;
        emit('resetPublishResponse');
        toggleControls();
        clearModalsTimeouts();
      };

      const getTime = (seconds: number) => {
        return dayjs.utc(seconds * 1000).format('mm:ss');
      };

      const goToMyCollection = () => {
        router.push(`/${Routes.MyCollections}/`);
      };

      const showOnboardingCrateKinomTooltip = () => {
        setTimeout(() => {
          onboardingCreateKinomTooltipVisible.value = true;

          setTimeout(() => {
            onboardingCreateKinomTooltipVisible.value = false;
          }, SecondsToMs.Ten);
        }, 200);
      };

      const setRange = () => {
        rangeStart.value = progress.value;

        const end = rangeStart.value + props.maxRangeDuration;
        rangeEnd.value = end <= videoDuration.value ? end : Math.floor(videoDuration.value);
      };

      watch(
        () => step.value,
        (_value, prevValue: Step) => {
          if (step.value === Step.Editing) {
            player.value?.controls(false);

            previewTimer.value = 0;
            previewViewdTime.value = 0;
            if (props.showTooltips && needShowOnboardingCreateKinom(prevValue)) {
              onboardingFullscreenTooltipVisible.value = false;
              showOnboardingCrateKinomTooltip();
              increaseOnboardingCreateKinomCounter();
            }
          } else {
            player.value?.controls(true);
          }

          if (step.value === Step.Preview) {
            player.value?.pause();

            setRange();

            if (props.showTooltips) {
              onboardingCreateKinomTooltipVisible.value = false;
            }
          }
        },
      );

      watch(
        () => rangeStart.value,
        () => {
          player.value?.currentTime(rangeStart.value);
        },
      );

      const onPublishSuccess = () => {
        step.value = Step.Finished;

        stepWatchingTimeout.value = window.setTimeout(() => {
          step.value = Step.Watching;
        }, SecondsToMs.Ten);
      };

      const onPublishFalure = () => {
        const failedResponse = props.publishResponse.find((response) => !response.success);
        if (!failedResponse) {
          return;
        }

        step.value = Step.Watching;
        errorText.value = failedResponse.status === HTTPStatusCodes.Forbidden ? ErrorText.Limit : ErrorText.Publish;

        errorTimeout.value = window.setTimeout(() => {
          emit('resetPublishResponse');
        }, SecondsToMs.Ten);
      };

      const onKinomsLimit = () => {
        errorText.value = ErrorText.Limit;

        errorTimeout.value = window.setTimeout(() => {
          errorText.value = '';
        }, SecondsToMs.Ten);
      };

      watch(
        () => props.publishResponse,
        () => {
          if (!props.publishResponse.length) {
            errorText.value = '';
            return;
          }

          if (props.publishResponse.find((response) => response.success)) {
            onPublishSuccess();
          } else {
            onPublishFalure();
          }
        },
      );

      const createKinomButtonClick = () => {
        onboardingFullscreenTooltipVisible.value = false;

        if (!props.canCreateKinom) {
          onKinomsLimit();
          return;
        }
        if (!player.value?.paused()) player.value?.pause();

        step.value = Step.Editing;
        emit('resetPublishResponse');
        clearModalsTimeouts();
      };

      const proceedOnClick = () => {
        step.value = Step.Preview;
      };

      const hideOnboarding = (event: Event) => {
        onboardingFullscreenTooltipVisible.value = false;
        onboardingCreateKinomTooltipVisible.value = false;

        const eventTarget = event.target as Element;
        if (eventTarget?.id === 'create-kinom') return;
        errorText.value = '';
        clearTimeout(errorTimeout.value);
      };

      const playerOnClick = (event: Event) => {
        hideOnboarding(event);
      };

      const pause = (player: videojs.Player) => {
        if (step.value !== Step.Watching) return;
        const isHalfVideoViewed = player.currentTime() >= player.duration() / 2;
        const impressionsNumber = Number($cookies.get(cookies.videoOnboardingFullscreenCounter.name) | 0);
        const isImpressionsLessThanLimit = impressionsNumber < ONBOARDING_FULLSCREEN_MAX_NUMBER;

        if (isFullscreen.value && props.showTooltips && isImpressionsLessThanLimit && isHalfVideoViewed) {
          showOnboardingFullscreenTooltip();
        }
      };

      const previewOnReady = (playerInstance: videojs.Player) => {
        previewPlayer.value = playerInstance;
      };

      const previewOnPlay = (playerInstance: videojs.Player) => {
        if (previewPauseRequested.value) {
          playerInstance.pause();
        }

        if (!previewEnded.value) return;
        playerInstance.currentTime(rangeStart.value);
        previewTimer.value = 0;
        previewEnded.value = false;
      };

      const previewOnTouchstart = () => {
        if (previewPlayer.value?.paused()) {
          previewPlayer.value?.play();
          previewPauseRequested.value = false;
        } else {
          previewPlayer.value?.pause();
          previewPauseRequested.value = true;
        }
      };

      return {
        progress,
        videoDuration,
        videoHasStarted,
        step,
        Step,
        rangeStart,
        rangeEnd,
        editor,
        isFullscreen,
        rangeDuration,
        previewViewdTime,
        SecondsToMs,
        onboardingFullscreenTooltipVisible,
        onboardingCreateKinomTooltipVisible,
        errorText,
        onTimeUpdate,
        playerReady,
        onLoadedMetadata,
        play,
        previewOnTimeUpdate,
        createKinom,
        onFullscreenChange,
        getTime,
        goToMyCollection,
        createKinomButtonClick,
        proceedOnClick,
        playerOnClick,
        pause,
        previewOnTouchstart,
        previewOnReady,
        previewOnPlay,
      };
    },
  });
</script>

<style lang="scss" scoped>
  @use '~@/assets/fonts' as fonts;
  @use '~@/assets/vars' as vars;

  .video-player-editor {
    position: relative;
  }

  .editor {
    opacity: 1;
    transition: opacity 0.2s;
  }

  .vjs-user-inactive:not(.vjs-paused) {
    .editor {
      opacity: 0;
    }
  }

  .buttons {
    position: absolute;
    top: 30px;
    right: 30px;

    @media screen and (max-width: 767px) {
      left: 20px;
      top: 55px;
      right: 0;
      width: 100%;
      flex-grow: 1;
    }

    .button {
      padding: 13px 30px;
      @include fonts.body-1;

      @media screen and (max-width: 767px) {
        padding: 9px 25px;
      }

      &.btn-continue {
        margin-left: 20px;

        @media screen and (max-width: 767px) {
          margin-left: 12px;
        }
      }

      &:last-child {
        margin-left: 20px;
      }

      &.btn.secondary {
        background: rgba($color-white, 0.2);
        backdrop-filter: blur(20px);

        &:hover {
          background-color: rgba($color-white, 0.2);
        }
      }
    }
  }

  .timeline-range {
    ::v-deep {
      position: absolute;
      bottom: 60px;
      left: 0;
      height: 30px;
      width: calc(100% - 60px);
      margin: 0 30px;

      @media screen and (max-width: 767px) {
        bottom: 30px;
      }
    }
  }

  .popup {
    position: absolute;
    padding: 40px 40px 30px;
    background: $color-white;
    border-radius: 24px;
    z-index: vars.$z-index-player-popup;

    @media screen and (max-width: 767px) {
      padding: 30px 30px 17px;
    }

    &__close {
      position: absolute;
      top: 20px;
      right: 20px;
      width: 16px;
      cursor: pointer;
      opacity: 0.7;

      &:hover {
        opacity: 1;
      }
    }
  }

  .preview {
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 100%;
    height: 100%;
    background: transparent;
    z-index: 3;

    ::v-deep {
      .modal__content {
        padding: 0;
        border-radius: 24px;
      }

      .modal__body-inner {
        padding: 40px 40px 30px 40px;

        @media screen and (max-width: 900px) {
          padding: 30px 16px 16px 16px;
        }
      }
    }

    &__title {
      margin-bottom: 20px;
      @include fonts.h5;
      color: $color-dark-green;
      font-family: Aeroport;

      @media screen and (max-width: 767px) {
        margin-bottom: 10px;
        @include fonts.body-2;
      }
    }

    &__player {
      margin-bottom: 30px;

      @media screen and (max-width: 767px) {
        margin-bottom: 16px;
      }

      ::v-deep .video-js {
        width: 500px;
        height: 280px;
        border-radius: 30px;
        overflow: hidden;

        @media screen and (max-width: 900px) {
          width: 370px;
          max-height: 200px;
        }

        @media screen and (max-width: 767px) {
          width: 303px;
          max-height: 170px;
        }

        @media screen and (max-width: 340px) {
          width: 280px;
        }

        .vjs-tech {
          border-radius: 30px;
        }
      }
    }

    &__buttons {
      display: flex;
      justify-content: flex-end;

      .button {
        @include fonts.body-1;
        padding: 13px 30px;
      }

      .btn.native-button.secondary {
        background: $color-white;
        color: $color-dark-primary;

        &:hover {
          background-color: $color-white;
        }
      }
    }

    ::v-deep {
      .vjs-big-play-button {
        pointer-events: inherit;
      }
    }

    &__duration {
      position: absolute;
      bottom: 20px;
      right: 20px;
      color: $color-white;
      @include fonts.body-2;
    }
  }

  .kinom-created {
    right: 30px;
    bottom: 117px;
    max-width: 333px;
    height: 190px;
    width: 100%;
    color: $color-dark-primary;
    padding: 20px 30px 30px;
    text-align: center;
    font-family: Aeroport;

    @media screen and (max-width: 912px) {
      height: fit-content;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      padding: 20px;
    }

    @media screen and (max-width: 350px) {
      max-width: 295px;
      padding: 20px 10px;

      .button {
        @include fonts.body-1;
      }
    }

    .title {
      margin-bottom: 6px;
      font-weight: bold;
    }

    .description {
      margin-bottom: 20px;
    }

    .button {
      padding: 13px 30px;
      margin-bottom: 0;
      font-weight: normal;
      white-space: nowrap;
    }

    .popup__close {
      right: 30px;
    }
  }

  ::v-deep {
    .tooltip-inner {
      position: absolute;
      padding: 14px 20px 20px;
      background: $color-white;
      border-radius: 12px;
      opacity: 1;
      color: $color-dark-primary;
      flex-direction: column;

      .create-kinom {
        flex-shrink: 0;
        width: 207px;
      }
    }

    .tooltip {
      pointer-events: none;
      font-family: Aeroport;

      &__title {
        display: block;
        margin-bottom: 6px;
        @include fonts.h6;
        white-space: nowrap;
        text-align: left;
        font-weight: 700;
      }

      &__content {
        @include fonts.body-2;
        display: block;
        text-align: left;
      }
    }

    .tooltip-arrow {
      display: block;
      transform: rotate(45deg);
      background: $color-white;
      width: 13px;
      height: 13px;
    }

    .tooltip {
      background: transparent;
      border: none;
    }

    #start-point {
      .tooltip .wrapper {
        transform: translate(-6px, -3px);
      }

      .tooltip.vue-tooltip-theme,
      .tooltip-inner {
        width: 279px;
        max-width: 279px;
        height: 124px;
      }

      .tooltip-arrow {
        position: absolute;
        bottom: -4px;
        transform: rotate(45deg) translate(15px, -12px);
      }
    }

    #create-kinom {
      .tooltip .wrapper {
        transform: translate(-22px, 12px);

        @media screen and (max-width: 767px) {
          transform: translate(15px, 14px);
        }
      }

      .tooltip.vue-tooltip-theme,
      .tooltip-inner {
        width: 207px;
        max-width: 207px;
        height: fit-content;
      }

      .tooltip-arrow {
        position: absolute;
        right: 0px;
        top: 0;
        transform: rotate(45deg) translate(110px, -119px);

        @media screen and (max-width: 767px) {
          transform: rotate(45deg) translate(14px, -22px);
        }
      }
    }

    #tooltip-error-container {
      .tooltip.vue-tooltip-theme,
      .tooltip-inner {
        width: 267px;
        max-width: 100%;
        min-height: 80px;
        height: min-content;
        padding: 20px;
        text-align: left;
      }

      .tooltip .wrapper {
        transform: translate(-26px, -13px);

        @media screen and (max-width: 767px) {
          transform: translate(0px, -13px);
        }
      }

      .tooltip-inner {
        background: $color-highlight;
        color: $color-white;
        font-family: Aeroport;
        border-radius: 10px;
      }

      .tooltip-arrow {
        display: none;
      }
    }
  }
</style>
