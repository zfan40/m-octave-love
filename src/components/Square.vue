<script>
import Card from "./common/BigCard";
import { swiper, swiperSlide } from "vue-awesome-swiper";
import * as Util from "../_common/js/util";
import * as Magic from "../_common/js/magic";
import { mbMixin } from "../_common/js/mixin.js";
import * as Api from "../_common/js/api";
import * as Cookies from "js-cookie";
import * as WxShare from "../_common/js/wx_share";
/**
 * 首页 component
 * 业务逻辑， 左滑动下一个， 右滑动点赞下一个， 双击点赞，点击中间播放， 点击下面按钮 （编曲页面 | 购买页面）
 *
 */
export default {
  name: "Square",
  mixins: [mbMixin],
  components: {
    Card,
    swiper,
    swiperSlide
  },
  data() {
    return {
      cards: [], //to be commented
      userId: 0,
      bigCardListOption: [],
      activeIndex: 0,
      autoplayFlag: false
    };
  },
  computed: {
    squareWorksObj() {
      return this.$store.state.squareWorksObj;
    },
    // operatingWorkId() {
    //   return this.$store.state.operatingWorkId;
    // },
    playingWorkId() {
      return this.$store.state.playingWorkId;
    },
    bigSwiper() {
      return this.$refs.bigCardList.swiper;
    }
  },
  methods: {
    redirectToMaker() {
      this.$router.push({
        path: "/new-music-box-maker",
        query: {}
      });
    },
    loadWorks() {
      this.$store
        .dispatch("FETCH_SQUARE_WORKS", {
          page: this.squareWorksObj.current
            ? this.squareWorksObj.current + 1
            : 1,
          size: 10,
          orderStrategy: 2
        })
        .then(() => {
          if (
            +this.squareWorksObj.content.length < +this.squareWorksObj.total
          ) {
            //otherwise no more content
            this.busy = false;
          }
        });
    }
  },
  beforeRouteLeave(to, from, next) {
    Magic.clearTone();
    next();
  },
  created() {
    const self = this;
    this.bigCardListOption = {
      loop: false,
      // spaceBetween: 30,
      centeredSlides: true,
      slidesPerView: 1,
      // slidesPerView: "auto",
      watchSlidesProgress: true,
      on: {
        transitionEnd(e) {
          const { size, current } = self.squareWorksObj;
          // console.log(this.realIndex);
          self.targetProduct = self.squareWorksObj.content[this.realIndex];
          self.activeIndex = this.realIndex;
          //TODO: first one need click, rest should be auto played.
          if (self.autoplayFlag)
            self.playWork(self.squareWorksObj.content[this.realIndex]);
          if (self.activeIndex >= (size * current - 1) / 2) {
            self.loadWorks();
          }
        },
        progress(progress) {
          // this.activeIndex = progress;
          // console.log()
        }
      }
    };
    //check cookie to get serviceToken first
    // if stoken not exist, go auth
    var docElem = document.documentElement;
    window.rem = docElem.getBoundingClientRect().width / 10;
    docElem.style.fontSize = window.rem + "px";

    const inWechat = /micromessenger/.test(navigator.userAgent.toLowerCase());
    this.userId = 239;
    this.$toast("向右滑动浏览");
    if (!inWechat) {
      self.loadWorks();
      return;
    }
    const fullPath = `${location.origin}${
      location.pathname
    }#/new-music-box-viewer?id=${self.$store.state.route.query.id}`;
    WxShare.prepareShareConfig().then(() => {
      WxShare.prepareShareContent({
        title: "哎八音推荐广场",
        desc: "｡:.ﾟヽ(*´∀`)ﾉﾟ.:｡",
        // fullPath:location.href.split('#')[0],
        fullPath,
        imgUrl: "http://cdn.cnbj1.fds.api.mi-img.com/mbox/aby.jpg"
      });
    });
    // alert(Cookies.get('serviceToken'))
    if (Util.getUrlParam("code") || Cookies.get("serviceToken")) {
      Api.getUserInfo(Util.getUrlParam("code"))
        .then(res => {
          if (res.data.errcode >= 20000) {
            // 网页内cookie失效，需要重新验证
            Cookies.remove("serviceToken");
            location.replace(
              // will publish to node project m-musixise, under '/music-box' path
              `https://open.weixin.qq.com/connect/oauth2/authorize?appid=wx353a60a8b049d366&redirect_uri=${encodeURIComponent(
                fullPath
              )}&response_type=code&scope=snsapi_userinfo&state=STATE#wechat_redirect`
            );
          }
          // self.userId =
          //   this.$store.state.route.query.id || res.data.data.userId;
          // self.isMe = self.userId == res.data.data.userId;
          self.loadWorks();
          console.log("get user info success", res.data.data);
        })
        .catch(err => {
          Cookies.remove("serviceToken");
          location.replace(
            `https://open.weixin.qq.com/connect/oauth2/authorize?appid=wx353a60a8b049d366&redirect_uri=${encodeURIComponent(
              fullPath
            )}&response_type=code&scope=snsapi_userinfo&state=STATE#wechat_redirect`
          );
        });
    } else {
      //又没有微信给的auth code又没有token存在cookie，只得验证
      location.replace(
        `https://open.weixin.qq.com/connect/oauth2/authorize?appid=wx353a60a8b049d366&redirect_uri=${encodeURIComponent(
          fullPath
        )}&response_type=code&scope=snsapi_userinfo&state=STATE#wechat_redirect`
      );
    }
  }
};
</script>
<template>
  <div class="container">
    <div id="swiper-container">
      <swiper :options="bigCardListOption" ref="bigCardList">
        <swiper-slide v-for="item in squareWorksObj.content" :key="item.id">
          <card
            :workInfo="item"
            :onPlayWork="()=>playWork(item)"
            :playingStatus="item.id==playingWorkId"
            :onDownloadWork="downloadWork"
            :onPurchaseWork="()=>purchaseWork(item)"
            :onToggleLike="()=>toggleLike(item,'squareWorksObj')"
          ></card>
        </swiper-slide>
      </swiper>
    </div>
    <div class="button_group">
      <button @click="redirectToMaker">我来试试</button>
      <!-- FUTURE WORK: not all work can be built, disabled here on condition -->
      <button @click="purchaseWork(squareWorksObj.content[activeIndex])">制成礼物</button>
    </div>
  </div>
</template>
<style lang="scss" scoped>
@import "../_common/style/_functions.scss";
@import "../_common/style/_variables.scss";
@import "../_common/style/_mixins.scss";
@import "../_common/style/_reboot.scss";
.container {
  position: absolute;
  height: 100%;
  width: 100%;
  top: 0;
  left: 0;
  display: flex;
  flex-direction: column;
  justify-content: space-around;
  align-items: center;
  background: rgb(26, 28, 30);
}
#swiper-container {
  position: relative;
  width: 100%;
  height: getRem(920);
}
.button_group {
  width: 100%;
  height: getRem(100);
  // position: absolute;
  bottom: getRem(60);
  margin: 0 auto;
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
  z-index: 100;
  button {
    outline: 0;
    border: 0;
    display: block;
    width: getRem(325);
    color: #fff;
    padding: getRem(20);
    font-size: 16px;
    &:first-of-type {
      border-radius: getRem(50) 0 0 getRem(50);
      background: rgb(113, 113, 230);
    }
    &:last-of-type {
      border-radius: 0 getRem(50) getRem(50) 0;
      background: rgb(69, 100, 215);
    }
  }
}
</style>

