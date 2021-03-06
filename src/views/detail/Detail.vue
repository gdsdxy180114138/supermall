<template>
  <div id="detail">
    <detail-nav-bar class="detail-nav" @titleClick="titleClick" ref="nav" />
    <scroll
      class="content"
      ref="scroll"
      :probe-type="3"
      @scroll="contentScroll"
    >
      <detail-swiper :top-images="topImages" />
      <detail-base-info :goods="goods" />
      <detail-shop-info :shop="shop" />
      <detail-goods-info :detail-info="detailInfo" @imageLoad="imageLoad" />
      <detail-param-info :param-info="paramInfo" ref="params" />
      <detail-comment-info :comment-info="commentInfo" ref="comment" />
      <goods-list :goods="recommends" ref="recommend" />
    </scroll>
    <detail-bottom-bar @addCart="addToCart" />
    <back-top @click.native="backClick" v-show="isShowBackTop" />
  </div>
</template>

<script>
import DetailNavBar from "./childComps/DetailNavBar";
import DetailSwiper from "./childComps/DetailSwiper";
import DetailBaseInfo from "./childComps/DetailBaseInfo";
import DetailShopInfo from "./childComps/DetailShopInfo";
import DetailGoodsInfo from "./childComps/DetailGoodsInfo";
import DetailParamInfo from "./childComps/DetailParamInfo";
import DetailCommentInfo from "./childComps/DetailCommentInfo";
import DetailBottomBar from "./childComps/DetailBottomBar";

import Scroll from "components/common/scroll/Scroll";
import GoodsList from "components/content/goods/GoodsList";
import { debounce } from "common/utils";
import { itemListenerMixin, backTopMixin } from "common/mixin";
import Toast from "components/common/toast/Toast";

import { mapActions } from "vuex";

import {
  getDetail,
  Goods,
  Shop,
  GoodsParam,
  getRecommend,
} from "network/detail";

export default {
  name: "Detail",
  mixins: [itemListenerMixin, backTopMixin],
  data() {
    return {
      iid: null,
      topImages: [],
      goods: {},
      shop: {},
      detailInfo: {},
      paramInfo: {},
      commentInfo: {},
      recommends: [],
      //itemImgListener: null,
      themeTopYs: [],
      getThemeTopY: null,
      currentIndex: 0,
    };
  },
  components: {
    DetailNavBar,
    DetailSwiper,
    DetailBaseInfo,
    DetailShopInfo,
    DetailGoodsInfo,
    DetailParamInfo,
    DetailCommentInfo,
    DetailBottomBar,
    GoodsList,
    Toast,
    Scroll,
  },
  methods: {
    ...mapActions(["addCart"]),
    imageLoad() {
      this.$refs.scroll.refresh();
      //获取offsetTop的值
      this.getThemeTopY();
    },
    titleClick(index) {
      this.$refs.scroll.scrollTo(0, -this.themeTopYs[index], 200);
    },
    contentScroll(position) {
      //1.获取y值
      const positionY = -position.y;
      //2.positionY和主题中值进行对比
      let length = this.themeTopYs.length;
      for (let i = 0; i < length; i++) {
        if (
          this.currentIndex !== i &&
          ((i < length - 1 &&
            positionY >= this.themeTopYs[i] &&
            positionY < this.themeTopYs[i + 1]) ||
            (i === length - 1 && positionY >= this.themeTopYs[i]))
        ) {
          this.currentIndex = i;
          this.$refs.nav.currentIndex = this.currentIndex;
        }
      }

      //3.判断BackTop是否显示
      this.isShowBackTop = Math.abs(position.y) > 1000;
    },
    addToCart() {
      //1.获取购物车需要展示的信息
      const product = {};
      product.image = this.topImages[0];
      product.title = this.goods.title;
      product.desc = this.goods.desc;
      product.price = this.goods.realPrice;
      product.iid = this.iid;

      //2.将商品添加到购物车(1.Promise,2.mapActions)
      //正常调用方式
      // this.$store.dispatch("addCart", product).then((res) => {
      //   console.log(res);
      // });
      //使用mapActions调用addCart
      this.addCart(product).then((res) => {
        // this.isShow = true;
        // this.message = res;
        // setTimeout(() => {
        //   this.isShow = false;
        //   this.message = "";
        // }, 1500);

        //使用自定义封装的插件
        this.$toast.show(res, 1500);
      });
    },
  },
  created() {
    //1.保存传入的iid
    this.iid = this.$route.params.iid;

    //2.根据iid请求详情数据
    getDetail(this.iid).then((res) => {
      //1.获取顶部的图片轮播数据
      //console.log(res);
      const data = res.result;
      this.topImages = res.result.itemInfo.topImages;
      //2.获取商品信息
      this.goods = new Goods(
        data.itemInfo,
        data.columns,
        data.shopInfo.services
      );
      //3.创建店铺信息的对象
      this.shop = new Shop(data.shopInfo);
      //4.保存商品的详情数据
      this.detailInfo = data.detailInfo;
      //5.获取参数的信息
      this.paramInfo = new GoodsParam(
        data.itemParams.info,
        data.itemParams.rule
      );
      //6.取出评论的信息
      if (data.rate.cRate !== 0) {
        this.commentInfo = data.rate.list[0];
      }
      //7.请求推荐数据
      getRecommend().then((res) => {
        //console.log(res);
        this.recommends = res.data.list;
      });

      //8.给getThemeTopY赋值(对给getThemeTopY的赋值操作进行防抖)
      this.getThemeTopY = debounce(() => {
        this.themeTopYs = [];
        this.themeTopYs.push(0);
        this.themeTopYs.push(this.$refs.params.$el.offsetTop);
        this.themeTopYs.push(this.$refs.comment.$el.offsetTop);
        this.themeTopYs.push(this.$refs.recommend.$el.offsetTop);
      }, 100);

      // //第一次获取，值不对，原因：created里还没有渲染DOM
      // this.themeTopYs = [];
      // this.themeTopYs.push(0);
      // this.themeTopYs.push(this.$refs.params.$el.offsetTop);
      // this.themeTopYs.push(this.$refs.comment.$el.offsetTop);
      // this.themeTopYs.push(this.$refs.recommend.$el.offsetTop);

      // //第二次获取：值不对，原因图片没有计算在内
      // //根据最新的数据，对应的DOM是已经被渲染出来
      // //但是图片依然是没有加载完(目前获取到offsetTop不包含其中的图片)
      // //offsetTop值不对的时候，都是因为图片的问题
      // this.$nextTick(() => {
      //   this.themeTopYs = [];
      //   this.themeTopYs.push(0);
      //   this.themeTopYs.push(this.$refs.params.$el.offsetTop);
      //   this.themeTopYs.push(this.$refs.comment.$el.offsetTop);
      //   this.themeTopYs.push(this.$refs.recommend.$el.offsetTop);
      // });
    });
  },
  mounted() {
    // let refresh = debounce(this.$refs.scroll.refresh, 500);
    // this.itemImgListener = () => {
    //   refresh();
    // };
    // this.$bus.$on("itemImgLoad", this.itemImgListener);
    //在mounted获取不到的原因，数据还没获取到
    // this.themeTopYs = [];
    // this.themeTopYs.push(0);
    // this.themeTopYs.push(this.$refs.params.$el.offsetTop);
    // this.themeTopYs.push(this.$refs.comment.$el.offsetTop);
    // this.themeTopYs.push(this.$refs.recommend.$el.offsetTop);
  },
  destroy() {
    this.$bus.$off("itemImgLoad", this.itemImgListener);
  },
};
</script>

<style scoped>
#detail {
  position: relative;
  z-index: 9;
  background-color: #fff;
  height: 100vh;
}

.detail-nav {
  position: relative;
  z-index: 9;
  background-color: #fff;
}

.content {
  /* height: calc(100% - 44px - 49px); */
  position: absolute;
  top: 44px;
  bottom: 60px;
}
</style>
