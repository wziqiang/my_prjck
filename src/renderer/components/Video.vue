<template>
  <div class="video-container" @contextmenu="contextmenu">
    <transition name="router" mode="out-in">
      <FileInfo @click="isShowInfo=false" :isShow="isShowInfo" :video="videoInfo" />
    </transition>
    <div class="message">
      <p>{{speedMsg}}</p>
      <p>{{volumeMsg}}</p>
      <p>{{playModeMsg}}</p>
    </div>
    <div @click="changePlayingMode" v-show="currentVideo" id="dplayer" class="my-video"></div>
    <div v-show="!currentVideo" class="my-video"></div>
    <div
      :style="{'animation-play-state':animationPlayState}"
      @click="changePlayingMode"
      v-show="isMusic && currentVideo"
      class="music-bg"
    ></div>
    <div
      :style="{'color':theme.textColor,'border': `1px solid ${theme.textColor}`}"
      class="open-file"
      v-if="!currentVideo"
    >
      <div class="flexrowcenter" @click="openFile">
        <span class="fa fa-folder-open-o"></span>
        <span>{{$t('common.openFile')}}</span>
      </div>
      <span @click.stop="showMenu" class="fa fa-angle-down"></span>
      <transition name="router" mode="out-in">
        <ul :style="{'background-color':theme.bgColor}" v-if="isShowFileMenu" class="my-file">
          <li :class="theme.hover" @click="openFolder">
            <span class="fa fa-file-video-o"></span>
            {{$t('common.openFolder')}}
          </li>
          <li :class="theme.hover" @click="openUrl">
            <span class="fa fa-link"></span>
            {{$t('common.openUrl')}}
          </li>
        </ul>
      </transition>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapMutations, mapActions } from "vuex";
import OpenDialog from "../api/OpenDialog";
import connect from "../api/bus.js";
import Mousetrap from "mousetrap";
import path from "path";
import "DPlayer/dist/DPlayer.min.css";
import DPlayer from "DPlayer";
import { musicReg } from "../api/util";
import { remote } from "electron";
import fs from "fs";
import { playOrderList, volumePercentList } from "../config";
import { log } from "util";

const openDialog = new OpenDialog();

const { Menu } = remote;

export default {
  name: "my-video",
  data() {
    return {
      // ????????????????????????
      isShowFileMenu: false,
      // ????????????????????????
      volumeMsg: "",
      // ??????????????????????????????
      speedMsg: "",
      // ??????????????????????????????
      playModeMsg: "",
      // ?????????????????????
      isMusic: false,
      // ?????????????????????????????????
      animationPlayState: "running",
      // ????????????????????????
      isShowInfo: false,
      // ????????????
      videoInfo: null
    };
  },
  methods: {
    ...mapMutations([
      "setPlaying",
      "setCurrentVideoIndex",
      "setCurrentTime",
      "setTotalTime",
      "setSpeed",
      "setOldVideo",
      "setCurrentVideo",
      "setFullScreen",
      "setHistoricalRecords",
      "setAlwaysOnTop",
      "setPlayMode",
      "setInWidth"
    ]),
    ...mapActions(["changeVideoList"]),
    // ?????????????????????
    initListener() {
      // ???????????????????????????????????????????????????setCurrentTime????????????
      connect.$on("setCurrentTime", () => {
        this.dp.seek(this.currentTime);
      });
      connect.$on("videoInfo", data => {
        this.videoInfo = data;
        this.isShowInfo = true;
      });
      connect.$on("openUrl", () => {
        this.openUrl();
      });
    },
    // ??????????????????
    removeListener() {
      connect.$off("setCurrentTime");
      connect.$off("videoInfo");
      connect.$off("openUrl");
    },
    // ???????????????????????????????????????
    showMenu() {
      // ????????????????????????????????????????????????????????????????????????????????????????????????
      if (!this.isShowFileMenu) {
        document.body.click();
      }
      this.isShowFileMenu = !this.isShowFileMenu;
    },
    onClick() {
      // ????????????
      this.isShowFileMenu = false;
    },
    // ????????????
    openFile() {
      openDialog.openFile();
    },
    // ????????????????????????
    timeupdate() {
      this.setCurrentTime(this.dp.video.currentTime);
    },
    // ????????????????????????
    durationchange() {
      this.setTotalTime(this.dp.video.duration);
    },
    // ???????????????????????????/?????????????????????,?????????????????????????????????????????????????????????
    loadedmetadata() {
      // ??????????????????????????????
      this.setOldVideo(
        Object.assign({}, this.currentVideo, {
          totalTime: this.dp.video.duration
        })
      );
      // ??????????????????
      this.setPlaying(true);
      this.dp.play();
      // ??????????????????????????????
      this.dp.speed(this.currentVideo.speed);
      this.dp.seek(this.currentVideo.currentTime);
    },
    // ????????????????????????
    changePlayingMode() {
      // ???????????????????????????
      if (!this.currentVideo) {
        return;
      }
      this.setPlaying(!this.isPlaying);
    },
    // ??????????????????
    ended() {
      // ????????????
      this.setPlaying(false);
      // ????????????
      this.setCurrentTime(0);
      switch (this.playMode) {
        // ????????????
        case 1:
          this.setCurrentVideo(null);
          break;
        // ????????????
        case 2:
          this.$nextTick(() => {
            this.setPlaying(true);
          });
          break;
        // ??????????????????
        case 3:
          // ?????????????????????????????????
          if (this.videoList.length - 1 == this.currentVideoIndex) {
            // ??????????????????????????????
            this.setCurrentVideoIndex(0);
          } else {
            // ????????????????????????
            let index = this.currentVideoIndex + 1;
            this.setCurrentVideoIndex(index);
          }
          break;
        // ????????????
        case 4:
          // ?????????????????????????????????
          if (this.videoList.length - 1 == this.currentVideoIndex) {
            // ??????????????????????????????
            this.setCurrentVideo(null);
          } else {
            // ????????????????????????
            let index = this.currentVideoIndex + 1;
            this.setCurrentVideoIndex(index);
          }
          break;
        // ????????????
        case 5:
          // ????????????????????????????????????
          if (this.videoList.length <= 1) {
            this.$nextTick(() => {
              this.setPlaying(true);
            });
          } else {
            // ????????????????????????
            let index = Math.floor(Math.random() * this.videoList.length);
            // ?????????????????????????????????????????????
            while (index == this.currentVideoIndex) {
              // ??????????????????????????????????????????????????????
              index = Math.floor(Math.random() * this.videoList.length);
            }
            this.setCurrentVideoIndex(index);
          }
          break;
      }
    },
    // ?????????????????????????????????
    error() {
      console.log("error");
    },
    // ??????????????????
    initGlobalShortcut() {
      // ???????????????
      Mousetrap.bind("ctrl+o", () => {
        this.openFile();
        // ?????? false ?????????????????????????????????????????????
        return false;
      });
    },
    // ???????????????
    openFolder() {
      openDialog.openFolder();
    },
    // ??????????????????
    openUrl() {
      this.$prompt(this.$t("common.enterNetworkUrl"), {
        confirmButtonText: this.$t("common.sureBtn"),
        cancelButtonText: this.$t("common.cancelBtn"),
        inputValidator: this.inputValidator,
        inputErrorMessage: this.$t("common.errorNetworkUrl")
      })
        .then(({ value }) => {
          openDialog.openUrl(value);
        })
        .catch(e => {
          console.log(e);
        });
    },
    // ??????url
    inputValidator(e) {
      // ??????url
      let reg1 = /(http|ftp|https):\/\/[\w\-_]+(\.[\w\-_]+)+([\w\-\.,@?^=%&:/~\+#]*[\w\-\@?^=%&/~\+#])?/;
      let flag1 = reg1.test(e);
      return flag1;
    },
    // ?????????dplayer?????????
    initDplayer() {
      this.dp = new DPlayer({
        container: document.getElementById("dplayer"),
        hotkey: false
      });
      this.dp.on("timeupdate", this.timeupdate);
      this.dp.on("durationchange", this.durationchange);
      this.dp.on("loadedmetadata", this.loadedmetadata);
      this.dp.on("ended", this.ended);
      this.dp.on("error", this.error);
      this.dp.volume(this.volumePercent);
    },
    // ??????????????????
    handelDBClick() {
      if (!this.currentVideo) {
        return;
      }
      if (this.isFullScreen) {
        this.setFullScreen(false);
      } else {
        this.setFullScreen(true);
      }
    },
    speedTimers(msg) {
      if (this.speedTimer) {
        clearTimeout(this.speedTimer);
      }
      this.speedMsg = msg;
      this.speedTimer = setTimeout(() => {
        this.speedMsg = "";
        clearTimeout(this.speedTimer);
      }, 5000);
    },
    volumeTimers(msg) {
      if (this.volumeTimer) {
        clearTimeout(this.volumeTimer);
      }
      this.volumeMsg = msg;
      this.volumeTimer = setTimeout(() => {
        this.volumeMsg = "";
        clearTimeout(this.volumeTimer);
      }, 5000);
    },
    playModeTimers(msg) {
      if (this.playModeTimer) {
        clearTimeout(this.playModeTimer);
      }
      this.playModeMsg = msg;
      this.playModeTimer = setTimeout(() => {
        this.playModeMsg = "";
        clearTimeout(this.playModeTimer);
      }, 5000);
    },
    // ????????????
    contextmenu() {
      document.body.click();
      const isother = !volumePercentList.includes(this.volumePercent);
      const playOrderArr = playOrderList.map(item => {
        return {
          label: this.$t(item.label),
          type: "checkbox",
          checked: this.playMode == item.playMode,
          click: () => {
            this.setPlayMode(item.playMode);
          }
        };
      });
      const voiceArr = [];
      volumePercentList.forEach(item => {
        if (item !== 0) {
          voiceArr.push({
            label: `${item * 100}%`,
            type: "checkbox",
            checked: this.volumePercent == item,
            click: () => {
              let inWidth = item * 62;
              this.setInWidth(inWidth);
            }
          });
        }
      });
      let contextMenuTemplate = [
        {
          label: this.$t("common.open"),
          submenu: [
            {
              label: this.$t("common.openFile"),
              click: () => {
                openDialog.openFile();
              }
            },
            {
              label: this.$t("common.openFolder"),
              click: () => {
                openDialog.openFolder();
              }
            },
            {
              label: this.$t("common.openUrl"),
              click: () => {
                this.openUrl();
              }
            }
          ]
        },
        {
          type: "separator"
        },
        {
          label: this.$t("common.winRoof"),
          submenu: [
            {
              label: this.$t("common.never"),
              type: "checkbox",
              checked: !this.isAlwaysOnTop,
              click: () => {
                this.setAlwaysOnTop(false);
              }
            },
            {
              label: this.$t("common.alway"),
              type: "checkbox",
              checked: this.isAlwaysOnTop,
              click: () => {
                this.setAlwaysOnTop(true);
              }
            }
          ]
        },
        {
          type: "separator"
        },
        {
          label: this.$t("common.playList"),
          click: () => {
            connect.$emit("showPlayList");
          }
        },
        {
          label: this.$t("common.playOrder"),
          submenu: playOrderArr
        },
        {
          type: "separator"
        },
        {
          label: this.$t("common.voice"),
          submenu: [
            ...voiceArr,
            {
              label: isother
                ? `${this.$t("common.other")}(${Math.round(
                    this.volumePercent * 100
                  )}%)`
                : this.$t("common.other"),
              type: "checkbox",
              checked: isother
            },
            {
              label: this.$t("common.mute"),
              type: "checkbox",
              checked: this.volumePercent == 0,
              click: () => {
                let inWidth = 0;
                this.setInWidth(inWidth);
              }
            }
          ]
        },
        {
          type: "separator"
        },
        {
          label: this.$t("common.setting")
        }
      ];
      if (this.currentVideo) {
        let addMenu = [
          {
            label: this.isPlaying
              ? this.$t("common.suspend")
              : this.$t("common.play"),
            click: () => {
              this.setPlaying(!this.isPlaying);
            }
          },
          {
            type: "separator"
          }
        ];
        contextMenuTemplate.unshift(...addMenu);

        contextMenuTemplate.splice(4, 0, {
          label: this.isFullScreen
            ? this.$t("common.exitScreen")
            : this.$t("common.fullScreen"),
          click: () => {
            this.setFullScreen(!this.isFullScreen);
          }
        });

        contextMenuTemplate.push({
          label: this.$t("common.fileInfo"),
          click: () => {
            this.videoInfo = this.currentVideo;
            this.isShowInfo = true;
          }
        });
      }
      let m = Menu.buildFromTemplate(contextMenuTemplate);
      Menu.setApplicationMenu(m);
      m.popup({ window: remote.getCurrentWindow() });
    }
  },
  mounted() {
    window.addEventListener("click", this.onClick);
    this.initDplayer();
    this.$nextTick(() => {
      this.initListener();
    });
    this.initGlobalShortcut();
  },
  computed: {
    ...mapGetters([
      "currentVideo",
      "isPlaying",
      "videoList",
      "currentTime",
      "speed",
      "oldVideo",
      "inWidth",
      "volumePercent",
      "playMode",
      "currentVideoIndex",
      "isFullScreen",
      "isAlwaysOnTop",
      "theme",
      "isTrace"
    ])
  },
  watch: {
    isPlaying(newVal) {
      // ??????video???store??????isPlaying?????????
      this.$nextTick(() => {
        if (newVal) {
          this.isMusic && (this.animationPlayState = "running");
          this.playModeTimers(this.$t("common.play"));
          this.dp.play();
        } else {
          this.isMusic && (this.animationPlayState = "paused");
          this.playModeTimers(this.$t("common.suspend"));
          this.dp.pause();
        }
      });
    },
    currentVideo: {
      immediate: true,
      handler: function(newVal, oldVal) {
        this.$nextTick(() => {
          // ????????????????????????
          if (!this.isTrace) {
            // ?????????????????????????????????????????????????????????????????????????????????
            this.changeVideoList(
              Object.assign({}, this.oldVideo, {
                currentTime: this.currentTime,
                speed: this.speed
              })
            );
            // ????????????
            if (oldVal) {
              this.setHistoricalRecords(
                Object.assign({}, this.oldVideo, {
                  currentTime: this.currentTime,
                  speed: this.speed
                })
              );
            }
          }
          // ?????????????????????
          if (newVal) {
            // ?????????????????????,??????????????????
            if (!fs.existsSync(newVal.src) && newVal.mode == "local") {
              // ?????????????????????????????????
              this.setCurrentVideo(null);
              // ????????????
              this.setPlaying(false);
              // ????????????????????????
              let video = Object.assign({}, newVal, { msg: "????????????" });
              this.setOldVideo(Object.assign({}, video));
              // ??????????????????
              this.changeVideoList(video);
              return;
            }
            // ????????????????????????????????????
            this.setCurrentTime(newVal.currentTime);
            // ????????????????????????????????????
            this.setSpeed(newVal.speed);
            // ?????????????????????????????????????????????
            const index = this.videoList.findIndex(i => i.id == newVal.id);
            // ?????????????????????
            this.setCurrentVideoIndex(index);

            if (this.dp) {
              this.initDplayer();
            }
            // ???????????????
            let extname = path.extname(newVal.src);
            this.isMusic = musicReg.test(extname);
            // ????????????,?????????????????????????????????????????????
            const url = this.inputValidator(newVal.src)
              ? newVal.src
              : `http://localhost:6789/video?video=${encodeURIComponent(
                  newVal.src
                )}`;
            // console.log(url);

            // this.dp.switchVideo({
            //   video: {
            //     url: url,
            //     type: "auto"
            //   }
            // });
            this.dp.switchVideo({
              url
            });
            if (!this.isTrace) {
              // ????????????????????????
              this.setHistoricalRecords(newVal);
            }
          } else {
            this.dp.switchVideo({
              url: ""
            });
          }
        });
      }
    },
    speed: {
      immediate: true,
      handler: function(newVal) {
        // ?????????????????????????????????video?????????
        this.$nextTick(() => {
          if (newVal > 1) {
            this.speedTimers(
              `${this.$t("common.playback")}???ctrl+up??????${newVal}${this.$t(
                "common.times"
              )}???${this.$t("common.reset")}???`
            );
          } else if (newVal < 1) {
            this.speedTimers(
              `${this.$t(
                "common.deceleration"
              )}???ctrl+down??????${newVal}${this.$t("common.times")}???${this.$t(
                "common.reset"
              )}???`
            );
          } else if (newVal == 1) {
            this.speedTimers(
              `${this.$t("common.normalSpeed")}???1.0${this.$t(
                "common.times"
              )}???${this.$t("common.reset")}???`
            );
          }
          this.dp.speed(newVal);
        });
      }
    },
    volumePercent: {
      // ??????????????????????????????????????????????????????????????????
      immediate: true,
      handler: function(newVal) {
        this.$nextTick(() => {
          this.volumeTimers(
            `${this.$t("common.volume")}???${Math.round(newVal * 100)}%`
          );
          this.dp.volume(newVal);
        });
      }
    }
  },
  beforeDestroy() {
    window.removeEventListener("click", this.onClick);
    if (this.volumeTimer) {
      clearTimeout(this.volumeTimer);
    }
    this.dp.destroy();
    this.removeListener();
  }
};
</script>

<style lang="less" scoped>
.video-container {
  flex: 1;
  position: relative;
  .message {
    position: absolute;
    top: 10px;
    left: 10px;
    z-index: 10;
    color: #ffffff;
  }
}
.my-video {
  width: 100%;
  height: 100%;
  vertical-align: top;
}
.music-bg {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 400px;
  height: 400px;
  margin-top: -200px;
  margin-left: -200px;
  background: url("../assets/musicBg.jpg") no-repeat center center;
  background-size: cover;
  border-radius: 50%;
  animation: rotate 10s;
  animation-iteration-count: infinite;
  animation-timing-function: linear;
}
.open-file {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 150px;
  height: 40px;
  border-radius: 40px;
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-around;
  padding: 5px;
  .my-file {
    z-index: -1;
    position: absolute;
    top: 45px;
    left: 0;
    width: 100%;
    border-radius: 5px;
    &:before {
      content: "";
      height: 0;
      width: 0;
      position: absolute;
      top: -10px;
      left: 23px;
      border: 5px solid transparent;
      border-bottom-color: greenyellow;
    }
    > li {
      width: 100%;
      height: 40px;
      padding: 10px 15px;
      color: #878788;
      cursor: pointer;
      &:hover {
        border-radius: 5px;
        color: #5dee00;
      }
    }
  }
  > div {
    cursor: pointer;
    span:nth-child(1) {
      padding-right: 10px;
    }
    &:hover {
      color: #5dee00;
    }
  }
  > span {
    font-size: 20px;
    border-left: 1px solid #818181;
    padding-left: 10px;
    cursor: pointer;
    &:hover {
      color: #5dee00;
    }
  }
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>