<template>
  <el-row class="player">
    <el-row class="player-show" v-if="Object.keys(video).length > 0">
      <el-row class="player-title">
        <el-row class="player-title-box" type="flex" justify="space-between">
          <span>
            <span>{{video.name ? video.name + ' -- ' : '' }}</span>
            <span>{{ num }}</span>
          </span>
          <span>
            <el-button size="mini" @click="topEvent('top')" icon="el-icon-place" title="置顶" circle></el-button>
            <el-button size="mini" @click="openDetail" icon="el-icon-document" title="查看详情" circle></el-button>
            <el-button size="mini" v-show="!star" @click="starEvent" icon="el-icon-star-off" title="添加收藏" circle></el-button>
            <el-button size="mini" v-show="star" @click="starEvent" icon="el-icon-star-on" title="取消收藏" circle></el-button>
            <el-button size="mini" @click="shareEvent" icon="el-icon-share" title="分享" circle></el-button>
            <el-popover placement="bottom" width="150" trigger="click">
              <el-row id="qrcode"></el-row>
              <el-button v-show="xg !== null" size="mini" @click="mobileEvent" icon="el-icon-mobile-phone" title="手机观看" circle slot="reference" style="margin-left: 10px;"></el-button>
            </el-popover>
          </span>
        </el-row>
      </el-row>
      <el-row class="player-box">
        <div id="xg"></div>
      </el-row>
      <el-row class="player-films table-box">
        <el-row class="player-films-box">
          <el-button :type="j === video.index ? 'primary' : ''" size="mini" v-for="(i, j) in urls" :key="j" @click="playBtnClick(i, j)" plain>{{i | ftLink}}</el-button>
        </el-row>
      </el-row>
    </el-row>
    <el-row class="player-none"  v-if="Object.keys(video).length <= 0">
      <el-row class="tips">
        浏览或者搜索资源, 点击播放, 即可观看~
      </el-row>
      <el-row class="btns">
        <el-button size="small" @click="goView('Film')" icon="el-icon-film">浏览</el-button>
        <el-button size="small" @click="goView('Search')" icon="el-icon-search">搜索</el-button>
      </el-row>
    </el-row>
  </el-row>
</template>
<script lang="ts">
import Vue from 'vue'
import { mapMutations } from 'vuex'
import zy from '@/lib/util.zy'
import 'xgplayer'
// @ts-ignore
import Hls from 'xgplayer-hls.js'
import video from '@/plugins/dexie/video'
import { qrcanvas } from 'qrcanvas'
const { ipcRenderer: ipc, clipboard } = require('electron')
export default Vue.extend({
  data () {
    return {
      xg: null,
      config: {
        id: 'xg',
        url: '',
        cssFullscreen: true,
        fluid: false,
        autoplay: true,
        videoInit: true,
        keyShortcut: 'on',
        defaultPlaybackRate: 1,
        playbackRate: [0.5, 0.75, 1, 1.5, 2]
      },
      urls: [],
      num: '',
      star: false
    }
  },
  computed: {
    d () {
      return this.$store.getters.getDetail
    },
    video: {
      get () {
        return this.$store.getters.getVideo
      },
      set (val) {
        this.SET_VIDEO(val)
      }
    },
    Main: {
      get () {
        return this.$store.getters.getMain
      },
      set (val) {
        this.SET_MAIN(val)
      }
    }
  },
  watch: {
    video: {
      handler () {
        this.getUrls()
      },
      deep: true
    }
  },
  filters: {
    ftLink (e: string) {
      let name = e.split('$')[0]
      return name
    }
  },
  methods: {
    ...mapMutations(['SET_DETAIL', 'SET_VIDEO', 'SET_MAIN']),
    topEvent (e:string) {
      ipc.send(e)
    },
    goView (e: string) {
      this.Main = e
    },
    openDetail () {
      let d = {
        show: true,
        video: this.video
      }
      this.SET_DETAIL(d)
    },
    getUrls () {
      if (this.xg) {
        // @ts-ignore
        this.xg.destroy(true)
        this.xg = null
      }
      this.checkStar()
      zy.detail(this.video.detail).then((res: any) => {
        this.urls = res.urls
        if (this.xg === null) {
          let info: any = this.urls[this.video.index]
          let url = info.split('$')[1]
          this.num = info.split('$')[0]
          this.config.url = url
          this.$nextTick(() => {
            this.xg = new Hls(this.config)
            // @ts-ignore
            this.xg.on('ended', () => {
              if (this.urls.length > 1 && (this.urls.length - 1 > this.video.index)) {
                this.$message.success('自动播放下一集')
                this.video.index++
                let v: any = this.urls[this.video.index]
                let url = v.split('$')[1]
                this.num = v.split('$')[0]
                // @ts-ignore
                this.xg.src = url
                video.find({ detail: this.video.detail }).then(res => {
                  if (res) {
                    video.update(res.id, this.video)
                  }
                })
              }
            })
          })
        }
      })
    },
    checkStar () {
      video.find({ detail: this.video.detail }).then(res => {
        if (res) {
          this.star = true
        } else {
          this.star = false
        }
      })
    },
    starEvent () {
      video.find({ detail: this.video.detail }).then(res => {
        if (res) {
          video.remove(res.id).then(res => {
            if (!res) {
              this.$message.success('删除成功')
              this.star = false
            } else {
              this.$message.warning('删除失败, 请重试~')
            }
          })
        } else {
          video.add(this.video).then(res => {
            this.star = true
            this.$message.success('收藏成功')
          })
        }
      })
    },
    mobileEvent () {
      let info = this.urls[this.video.index]
      // @ts-ignore
      let time = this.xg.currentTime
      const canvas = qrcanvas({
        size: 120,
        data: `http://zy.hly120506.top/player/index.html?info=${info}&time=${time}`
      })
      const dom = document.getElementById('qrcode')
      if (dom) {
        dom.innerHTML = ''
        dom.appendChild(canvas)
      }
    },
    shareEvent () {
      let info: string = this.urls[this.video.index]
      let title = this.video.name.replace(/^\s*|\s*$/g, '')
      let url = info.split('$')[1]
      let data = `http://zy.hly120506.top/player/index.html?info=${url}`
      let txt = `资源名称: ${title}\n播放地址:${data}`
      clipboard.writeText(txt)
      this.$message.success('资源已复制到剪贴板中，快去分享吧~')
    },
    playBtnClick (i: string, j: number) {
      if (this.video.index !== j) {
        let url = i.split('$')[1]
        this.num = i.split('$')[0]
        this.video.index = j
        // @ts-ignore
        this.xg.src = url
        video.find({ detail: this.video.detail }).then(res => {
          if (res) {
            this.video.index = j
            video.update(res.id, this.video)
          }
        })
      }
    }
  }
})
</script>
<style lang="scss" scoped>
.player{
  height: 100%;
  width: 100%;
  position: relative;
  .player-show{
    height: 100%;
    width: 100%;
    position: relative;
  }
  .player-none{
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
    .tips{
      font-size: 14px;
      margin-bottom: 10px;
    }
  }
  .player-title{
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 50px;
    line-height: 50px;
    .player-title-box{
      width: 600px;
      margin: 0 auto;
    }
  }
  .player-box{
    position: absolute;
    top: 50px;
    left: 0;
    width: 100%;
    height: 350px;
    #xg{
      margin: 0 auto;
    }
  }
  .player-films{
    position: absolute;
    top: 400px;
    left: 0;
    width: 100%;
    height: calc(100% - 400px);
    overflow: auto;
    button{
      margin: 0 10px 10px 0;
    }
    &::-webkit-scrollbar{
      width: 6px;
    }
    .player-films-box{
      width: 600px;
      margin: 0 auto;
    }
  }
}
</style>
