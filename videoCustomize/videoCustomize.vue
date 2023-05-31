<template>
  <div class="videoCustomize"
       ref="videoContainer"
       @mouseover="showControl = true"
       @mouseout="showControl = false">
    <div class="top-section"></div>
    <div class="middle-section">
      <video ref="video"
             preload="auto"
             v-bind="$attrs"
             v-on="$listeners"
             @click="isPlay === true ? pause() : play()"
             @waiting="videoLoading = true"
             @playing="videoLoading = false"
             @timeupdate="timeupdate"
             @progress="progress"
             @loadedmetadata="timeupdate"></video>
      <div v-show="videoLoading"
           class="loading-content"
           v-loading="videoLoading"
           element-loading-background="transparent">
        <div></div>
      </div>
    </div>
    <div class="bottom-section"
         @mouseleave="progressMouseLeave()"
         @mouseup="progressMouseLeave()"
         @mousemove="progressMousemove">
      <div :class="[showControl === true ? 'show-control' : '']"
           class="progress_bar hide-control"
           ref="progressController"
           @click="progressClickHandler">
        <div class="cache"
             :style="{ width: cacheValue }"></div>
        <div class="done"
             :style="{ width: progressPercent }"></div>
        <i :style="{ left: progressPercent }"
           @mousedown="progressMousedown"></i>
      </div>
    </div>
    <div class="bottom-mask"></div>
    <img :class="[showControl === true ? 'show-control' : '']"
         :src="isPlay === true ? playImg : pauseImg"
         class="play hide-control"
         @click="isPlay === true ? pause() : play()" />
    <div :class="[showControl === true ? 'show-control' : '']"
         class="time hide-control">
      {{ currentTime }} / {{ durationTime }}
    </div>

    <div :class="[showControl === true ? 'show-control' : '']"
         class="sound hide-control"
         @mouseleave="soundTrigger = false"
         @mouseup="soundTrigger = false"
         @mousemove="soundMousemove">
      <div class="controller"
           ref="soundController"
           @click="soundClickHandler">
        <div :style="{ width: `${volume}px` }"></div>
        <i @mousedown="soundMousedown"
           :style="{ left: `${volume}px` }"></i>
      </div>
      <img @click="soundImgHandler"
           class="sound-img"
           :src="volume === 0 ? quitImg : soundImg" />
    </div>
    <img :class="[showControl === true ? 'show-control' : '']"
         class="full hide-control"
         :src="isFull === true ? shrinkImg : fullImg"
         @click="fullscreenToggle" />
  </div>
</template>

<script>
import { mediaLaunchFullscreen, mediaExitFullscreen } from './tools';

export default {
  name: 'videoCustomize',
  inheritAttrs: false,
  data () {
    return {
      isPlay: false, // 是否正在播放
      volume: 0, // 音量
      soundInfo: {}, // 音量区域的宽度、最左边和最右边
      soundTrigger: false, // 鼠标在音量键上按下
      isFull: false, // 是否全屏
      videoLoading: true, // 是否正在加载
      video: null,

      progressInfo: {}, // 进度条区域的宽度、最左边和最右边
      progressTrigger: false, // 鼠标在进度条上按下
      currentTime: '00:00', // 当前时间
      durationTime: '00:00', // 总共时间
      duration: 0, //总共时间的值
      progressValue: 0,
      cacheValue: '0%',
      showControl: false,
      // 引用的图标
      playImg: require('./play.png'),
      pauseImg: require('./pause.png'),
      soundImg: require('./sound.png'),
      quitImg: require('./quit.png'),
      fullImg: require('./full.png'),
      shrinkImg: require('./shrink.png'),
    };
  },
  computed: {
    progressPercent () {
      return `${(
        (this.progressValue * 100) /
        this.progressInfo.width
      ).toFixed(6)}%`;
    },
  },
  mounted () {
    this.video = this.$refs.video;
    this.$refs.video.volume = 0;
    this.getProgressInfo();
    this.getSoundInfo();
  },
  methods: {
    // 开始播放
    play () {
      this.isPlay = true;
      this.video.play().catch((err) => {
        console.log(err);
      });
    },
    // 暂停播放
    pause () {
      this.isPlay = false;
      this.video.pause();
    },
    getProgressInfo () {
      const { x, width } =
        this.$refs.progressController.getBoundingClientRect();
      this.progressInfo = { begin: x, width, end: x + width };
    },
    // 鼠标按下进度调节按钮时
    progressMousedown (e) {
      this.getProgressInfo();
      this.progressBeginX = e.x;
      this.progressTrigger = true;
      if (this.isPlay === true) {
        // 如果正在播放，设置为暂停
        this.pause();
        this.statusChange = true;
      } else {
        this.statusChange = false;
      }
    },
    // 鼠标调节进度调节按钮时
    progressMousemove ({ x }) {
      const { begin, end, width } = this.progressInfo;
      if (this.progressTrigger === false || x < begin || x > end) {
        return false;
      }
      const xDiff = x - this.progressBeginX;
      this.progressValue += xDiff;
      this.progressBeginX = x;
      this.processChange(width);
    },
    processChange (width) {
      if (this.progressValue < 0) {
        this.progressValue = 0;
      }
      if (this.progressValue > width) {
        this.progressValue = width;
      }
      this.$refs.video.currentTime =
        (this.progressValue / width) * this.duration;
    },
    // 点击音量进度条时触发
    progressClickHandler ({ x }) {
      const { begin, width } = this.progressInfo;
      this.progressValue = x - begin;
      this.processChange(width);
    },
    progressMouseLeave () {
      if (this.progressTrigger === true) {
        this.progressTrigger = false;
        if (this.statusChange === true) {
          // 如果鼠标按下时为播放，此时恢复为播放
          this.play();
        }
      }
    },
    // 获取缓存进度
    progress (e) {
      let { duration, buffered, currentTime } = e.target; // 视频总长度
      if (duration > 0) {
        for (var i = 0; i < buffered.length; i++) {
          // 寻找当前时间之后最近的点
          if (buffered.start(buffered.length - 1 - i) < currentTime) {
            this.cacheValue =
              (buffered.end(buffered.length - 1 - i) / duration) * 100 + '%';
            break;
          }
        }
      }
    },
    setCurrentTime (seconds) {
      this.$refs.video.currentTime = seconds || 0;
    },
    fullscreenToggle () {
      if (this.isFull === false) {
        mediaLaunchFullscreen(this.$refs.videoContainer);
      } else {
        mediaExitFullscreen();
      }
      this.isFull = !this.isFull;
    },
    // 获取当前播放进度
    timeupdate (e) {
      const { currentTime = 0, duration = 0 } = e.target;
      this.duration = duration;
      this.progressValue = Math.ceil(
        (currentTime / duration) * this.progressInfo.width
      );
      this.currentTime = this.timeOperate(currentTime);
      this.durationTime = this.timeOperate(duration);
      if (currentTime === duration) {
        // 播放结束
        this.pause();
        this.video.currentTime = 0;
      }
      // this.videoLoading = false;
    },
    // 对时间进行格式转换
    timeOperate (time = 0) {
      if (Object.is(time, NaN)) {
        time = 0;
      }
      const second = parseInt(time);
      const minute = parseInt(time / 60);
      const hour = parseInt(minute / 60);
      return `${hour === 0 ? '' : `${hour.toString().padStart(2, '0')}:`}${(
        minute % 60
      )
        .toString()
        .padStart(2, '0')}:${(second % 60).toString().padStart(2, '0')}`;
    },
    getSoundInfo () {
      const { x, width } = this.$refs.soundController.getBoundingClientRect();
      this.soundInfo = { begin: x, width, end: x + width };
    },
    // 鼠标按下音量调节按钮时
    soundMousedown (e) {
      this.getSoundInfo();
      this.soundBeginX = e.x;
      this.soundTrigger = true;
    },
    // 鼠标调节音量调节按钮时
    soundMousemove ({ x }) {
      const { begin, end, width } = this.soundInfo;
      if (this.soundTrigger === false || x < begin || x > end) {
        return false;
      }
      const xDiff = x - this.soundBeginX;
      this.volume += xDiff;
      this.soundBeginX = x;
      this.soundChange(width);
    },
    // 点击音量进度条时触发
    soundClickHandler ({ x }) {
      const { begin, width } = this.soundInfo;
      this.volume = x - begin;
      this.soundChange(width);
    },
    // 修改音量
    soundChange (width) {
      if (this.volume < 0) {
        this.volume = 0;
      }
      if (this.volume > width) {
        this.volume = width;
      }
      this.$refs.video.volume = this.volume / width;
    },
    // 点击音量图标时触发
    soundImgHandler () {
      this.volume = this.volume === 0 ? 20 : 0;
      this.$refs.video.volume = this.volume / this.soundInfo.width;
    },
  },
};
</script>

<style scoped lang="less">
.videoCustomize {
  position: relative;
  display: flex;
  flex-direction: column;
  width: 100%;
  height: 100%;
  border-radius: 8px 0px 0px 8px;
  user-select: none;
  /deep/.el-loading-spinner .path {
    stroke: white;
    stroke-width: 4;
  }
  .hide-control {
    opacity: 0;
    transition: opacity 0.3s;
    cursor: pointer;
  }
  .show-control {
    opacity: 1;
  }
  .middle-section {
    position: relative;
    flex: 1;
    video {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
      &::-webkit-media-controls {
        display: none !important;
      }
    }
    .loading-content {
      position: absolute;
      left: 50%;
      top: 50%;
      width: 100px;
      height: 100px;
      transform: translate(-50%, -50%);
    }
  }
  .top-section,
  .bottom-section {
    position: relative;
    height: 49px;
    background-color: black;
    z-index: 1;
    /* 进度条 */
    .progress_bar {
      position: absolute;
      bottom: 20px;
      left: 20px;
      width: calc(100% - 40px);
      height: 4px;
      background-color: rgba(255, 255, 255, 0.3);
      border-radius: 2px;
      > div {
        position: absolute;
        width: 0;
        height: 100%;
        border-radius: 2px;
        &.done {
          background-color: #be965a;
        }
        &.cache {
          background-color: #999;
        }
      }
      > i {
        position: absolute;
        top: 50%;
        left: 0;
        transform: translate(-50%, -50%);
        width: 12px;
        height: 12px;
        background-color: white;
        border-radius: 50%;
      }
    }
  }
  .bottom-mask {
    position: absolute;
    width: 100%;
    height: 124px;
    bottom: 0;
    left: 0;
    pointer-events: none;
    background-image: linear-gradient(
      180deg,
      rgba(0, 0, 0, 0) 0%,
      #1e1e1e 100%
    );
    border-radius: 0px 0px 0px 8px;
  }
  img {
    width: 24px;
    height: 24px;
  }
  .play {
    position: absolute;
    left: 20px;
    bottom: 44px;
    z-index: 1;
  }
  .time {
    position: absolute;
    left: 64px;
    bottom: 46px;
    color: white;
    white-space: nowrap;
    font-size: 14px;
    z-index: 1;
  }
  /* 音量 */
  .sound {
    position: absolute;
    width: 138px;
    height: 36px;
    right: 62px;
    bottom: 38px;
    background-color: rgba(51, 51, 51, 0.5);
    border-radius: 18px;
    z-index: 1;
    .controller {
      position: absolute;
      left: 22px;
      top: 50%;
      width: 64px;
      height: 4px;
      transform: translateY(-50%);
      background-color: rgba(255, 255, 255, 0.3);
      border-radius: 2px;
      > div {
        position: absolute;
        height: 100%;
        background-color: #fff;
        border-radius: 2px;
      }
      > i {
        position: absolute;
        left: 0;
        top: 50%;
        width: 12px;
        height: 12px;
        transform: translate(-50%, -50%);
        border-radius: 50%;
        background-color: white;
      }
    }
    img {
      position: absolute;
      right: 12px;
      top: 50%;
      transform: translateY(-50%);
    }
  }
  /* 全屏 */
  .full {
    position: absolute;
    bottom: 44px;
    right: 20px;
    cursor: pointer;
    z-index: 1;
  }
}
</style>
