<template>
  <div id="live-list" class="platform-content app-scrollbar">
    <v-window :model-value="platformTab">
      <v-window-item v-for="(item, tabIndex) in livePlatformItems" :key="tabIndex" :value="tabIndex">

        <v-list style="padding-top: 0;padding-bottom: 0;background: #fff0;">
          <div v-for="(room, index) in liveList[tabIndex]" :key="index" style="">
            <!--            <v-divider v-if="index !== 0"></v-divider>-->

            <v-list-item
                @click.left.prevent="openVideo(tabIndex, room)"
                @click.right.prevent="switchExtend(tabIndex, room)"
                height="60px"
                :key="room.room_id"
                :prepend-avatar="room.anchor_avatar">
              <v-list-item-title style="font-size: 15px">
                <template v-if="room.is_live === 1 && room.is_loop === 0">
                  <v-icon size="20" style="margin-top: -4px" color="green">mdi-television-classic</v-icon>
                </template>
                <template
                    v-if="(room.is_live === 0 && room.is_loop === 1) || (room.is_live === 1 && room.is_loop === 1)">
                  <v-icon size="24" style="margin-top: -2px" color="#eaa400">mdi-video-outline</v-icon>
                </template>
                <template v-if="room.is_live === 0 && room.is_loop === 0">
                  <v-icon size="20" style="margin-top: -5px" color="red">mdi-television-classic-off</v-icon>
                </template>
                {{ room.room_name }}
              </v-list-item-title>
              <v-list-item-subtitle>
                {{ room.anchor_name }}
              </v-list-item-subtitle>
              <template v-slot:append style="width: 100px">
                <div style="margin-right: 10px">
                  <v-row style="float: right;margin-top: -10px">

                    <template v-if="room.showExtend === 1">
                      <v-btn variant="outlined" size="x-small" color="blue" width="55"
                             @click.stop="getStreamSourceUrl(tabIndex, room)">
                        直播源
                      </v-btn>
                    </template>
                    <template v-else>
                      <span v-if="room.is_live === 1 || room.is_loop === 1"
                            style="font-size: 12px">
                        <v-icon color="red" size="15"
                                style="margin-top: -3px;margin-right: -3px">mdi-fire</v-icon>
                          {{ room.watch_num_label }}
                      </span>
                    </template>


                  </v-row>
                  <v-row style="margin-top: 12px;float: right">

                    <template v-if="room.showExtend === 1">
                      <v-btn variant="outlined" size="x-small" color="red" width="55"
                             @click.stop="delFollow(tabIndex, room)">
                        取消关注
                      </v-btn>
                    </template>
                    <template v-else>
                      <span v-if="room.is_live === 1 || room.is_loop === 1" style="font-size: 12px">
                        <v-icon color="blue" size="14" style="margin-top: -3px;margin-right: -3px">mdi-clock-time-eight-outline</v-icon>
                          {{ room.live_time_label }}
                      </span>
                    </template>


                  </v-row>
                </div>
              </template>
            </v-list-item>
            <v-divider></v-divider>

          </div>
        </v-list>


      </v-window-item>
    </v-window>
  </div>

</template>

<script>
import {defineComponent, toRaw} from "vue";
import mainConfig from "../../../electron/conf/main";
import {ipcRenderer} from "electron";

export default defineComponent({
  name: "LiveList",
  props: ['platformTab'],
  data() {
    return {
      liveList: [],
      livePlatformItems: mainConfig.getLivePlatformItems(),
    }
  },
  methods: {
    openVideo(platformTab, roomInfo) {
      if (roomInfo.is_live === 1 || roomInfo.is_loop === 1) {
        ipcRenderer.invoke('open-video', [platformTab, roomInfo.room_id]).then(function () {
          ipcRenderer.send('change-video-info', [platformTab, roomInfo.room_id, roomInfo.room_name])
        })
      } else {
        ipcRenderer.send('alert-msg', ['warning', '未开播'])
      }
    },
    switchExtend(tabIndex, room) {
      if (room.showExtend === 1) {
        room.showExtend = -1
      } else {
        room.showExtend = 1
      }
    },
    getStreamSourceUrl(platformTab, roomInfo) {
      console.log("获取直播源：" + roomInfo.room_id)
      if (roomInfo.is_live === 1 && roomInfo.is_loop === 0) {
        ipcRenderer.invoke('get-live-url-info', [platformTab, roomInfo.room_id]).then((liveUrl) => {
          // 直播源地址复制到粘贴板
          ipcRenderer.invoke('live-url-to-clipboard', [liveUrl])
          ipcRenderer.send('alert-msg', ['info', '直播源地址已经复制到粘贴板'])
        })
      } else {
        ipcRenderer.send('alert-msg', ['error', '未开播，无法获取直播源地址'])
      }
    },
    delFollow(platformTab, roomInfo) {
      console.log('取消关注', platformTab, roomInfo)
      ipcRenderer.invoke('del-follow', [this.platformTab, roomInfo.room_id]).then((data) => {
        if (data.code === 200) {
          let oldLiveList = this.liveList[this.platformTab]
          let newLiveList = []
          for (let i = 0; i < oldLiveList.length; i++) {
            if (oldLiveList[i].room_id !== roomInfo.room_id) {
              newLiveList.push(oldLiveList[i])
            }
          }
          this.liveList[this.platformTab] = newLiveList
        }
      })
    },
    reflushLiveList() {
      const self = this
      let douyuDatas = []
      let bilibiliDatas = []
      let huyaDatas = []
      ipcRenderer.invoke('get-all-follow', []).then(function (follows) {
        ipcRenderer.invoke('get-room-info', [0, follows.douyu]).then(function (data) {
          for (let i = 0; i < data.length; i++) {
            douyuDatas.push(data[i])
          }
          self.liveList[0] = douyuDatas
        })
        ipcRenderer.invoke('get-room-info', [1, follows.bilibili]).then(function (data) {
          for (let i = 0; i < data.length; i++) {
            bilibiliDatas.push(data[i])
          }
          self.liveList[1] = bilibiliDatas
        })
        ipcRenderer.invoke('get-room-info', [2, follows.huya]).then(function (data) {
          for (let i = 0; i < data.length; i++) {
            huyaDatas.push(data[i])
          }
          self.liveList[2] = huyaDatas
          ipcRenderer.send('reflush-live-list-over', [])
        })
      })
    }
  },
  mounted() {
    const self = this
    for (let i = 0; i < 3; i++) {
      this.liveList.push([])
    }
    ipcRenderer.send('reflush-live-list', [])
    ipcRenderer.on('reflush-live-list-reply', function (event, args) {
      self.reflushLiveList()
    })
    setInterval(function () {
      ipcRenderer.send('reflush-live-list', [])
    }, 1000 * 60)
  }
})
</script>

<style scoped>
#live-list {
  background-color: #FFF;
}
</style>
