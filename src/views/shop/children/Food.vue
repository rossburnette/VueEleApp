<template>
  <section class="food">
    <!-- better-scrool 容器  && 左侧菜单栏 -->
    <section class="categories_wrapper" ref="categories_wrapper">
      <ul class="food_categories">
        <li
          v-for="(item,index) in foodList"
          :key="item.id"
          class="categories_item"
          :class="{activeClass:currentIndex == index}"
          @click="scrollToElement(index)"
        >{{item.name}}</li>
      </ul>
    </section>
    <!-- better-scrool 容器  && 右侧食品栏-->
    <section class="food_wrapper" ref="food_wrapper">
      <section>
        <div v-for="(foodItem) in foodList" :key="foodItem.id" class="list_item">
          <!-- 分类标题 -->
          <h3 class="title">
            {{foodItem.name}}
            <!-- 标题描述 -->
            <span class="title_info">{{foodItem.description}}</span>
          </h3>
          <!-- 分类下的商品 -->
          <ul>
            <li v-for="(item) in foodItem.foods" :key="item.item_idd" class="food_info">
              <img :src="`//elm.cangdu.org/img/${item.image_path}`" alt class="food_img" />
              <section>
                <!-- 商品名称 -->
                <h5 class="food_name">{{item.name}}</h5>
                <!-- 商品描述 -->
                <p class="food_des">{{item.description}}</p>
                <!-- 商品提示 -->
                <p class="food_tips">{{item.tips}}</p>
                <section class="buy_container">
                  <!-- 商品价格 -->
                  <span class="price">
                    <span class="icon" v-if="item.specfoods[0].price">￥</span>
                    {{item.specfoods[0].price}}
                  </span>
                  <!-- 添加购物车按钮 -->
                  <!-- 选择规格 -->
                  <span class="add_btn" v-if="item.specfoods.length > 1" @click="selected(item.specfoods)">选规格</span>
                  <!-- 添加购物车 -->
                  <CartControl v-else :foodItem="item.specfoods[0]"></CartControl>
                </section>
              </section>
            </li>
          </ul>
        </div>
      </section>
    </section>
    <!-- 选规格 -->
    <SelectedFood :foods="selectedFood" v-if="isShow"/>

    <!-- 底部购物车栏 -->
    <footer>
      <!-- 购物车详情 -->
      <CartDetail :foodList="cartFoods" v-if="isShowCart && cartFoods.length>0" />
      <!-- 购物车图标 -->
      <span
        class="cart"
        :style="{backgroundColor:cartFoods.length>0?'#0089dc':'#535356'}"
        @click="isShowCart = true;"
      >
        <span class="count" v-if="foodCount">{{foodCount}}</span>
        <i class="fa fa-shopping-cart" aria-hidden="true"></i>
      </span>
      <section class="cart_container">
        <section class="total">
          <p class="price">￥{{totalPrice}}元</p>
          <p class="fee">配送费5元</p>
        </section>
        <router-link
          class="payBtn"
          :style="{backgroundColor:cartFoods.length>0?'#4cd946':'#535356'}"
          tag="span"
          to="/confirmOrder"
        >去结算</router-link>
      </section>
    </footer>
    <!-- 遮罩层 -->
    <div class="layer_mask" v-if="isShowCart&&cartFoods.length > 0" @click="isShowCart = false"></div>
  </section>
</template>

<script>
import BScroll from "@better-scroll/core";
import { getFoods } from "../../../server/getData";
import CartControl from "../../../components/common/CartControl";
import CartDetail from "../../../components/common/CartDetail";
import SelectedFood from "../../../components/common/SelectedFood";
export default {
  name: "Food",
  data() {
    return {
      cart: [],
      foodList: [], //食品列表
      id: this.$route.query.id, // 路由查询参数
      listHeight: [], //右边每个li所处高度
      foodScroll: {}, //右侧的BScroll对象
      menuScroll: {}, //左侧的BScroll对象
      scrollY: 0, // Y轴滚动距离
      isShowCart: false,
      selectedFood: [],
      isShow:false
    };
  },
  computed: {
    totalPrice() {
      return this.$store.getters.totalPrice;
    },
    foodCount() {
      return this.$store.getters.foodCount;
    },
    // 计算出添加进购物车的商品
    cartFoods() {
      // 购物车列表
      let cartList = [];
      this.foodList.forEach(foods => {
        foods.foods.forEach(item => {
          item.specfoods.forEach(foodItem => {
            if (foodItem.count > 0) {
              cartList.push(foodItem);
            }
          });
        });
      });
      return cartList;
    },
    // 滚动过程中实时计算所在区间索引
    currentIndex() {
      for (let i = 0; i < this.listHeight.length; i++) {
        let height1 = this.listHeight[i];
        let height2 = this.listHeight[i + 1];
        // 触发滚动事件过程中进行判断，如果滚动过程中处于某个区间，则currentIndex为当前区间的索引
        if (this.scrollY >= height1 && this.scrollY < height2) {
          return i;
        }
      }
      return 0;
    }
  },
  watch: {
    cartFoods(newValue, oldValue) {
      this.$store.commit("UPDATE_CARTLIST", this.cartFoods);
      // 如果购物车中已经没有东西，那么购物车详情置为false，解决点击时弹出购物车详情bug
      if (newValue.length < 1) {
        this.isShowCart = false;
      }
    }
  },
  methods: {
    selected(foods) {
      this.selectedFood = foods;
      this.isShow = true;
    },
    // 关闭弹出框
    closeAlert() {
      if (this.isShow) {
        this.isShow = false;
      }
    },
    // 点击菜单滚动到某个元素
    scrollToElement(index) {
      let elementArray = this.$refs.food_wrapper.getElementsByClassName(
        "list_item"
      );
      this.foodScroll.scrollToElement(elementArray[index], 200);
    },
    // 初始化better-scroll
    initScroll() {
      this.menuScroll = new BScroll(this.$refs.categories_wrapper, {
        // ...... 详见配置项
        click: true
      });
      this.foodScroll = new BScroll(this.$refs.food_wrapper, {
        click: true,
        // 派发滚动事件
        probeType: 3
      });
    },
    calculateHeight() {
      // 获取所有li
      let itemArray = this.$refs.food_wrapper.getElementsByClassName(
        "list_item"
      );
      let height = 0;
      let heightArray = [];
      heightArray.push(height);
      for (let i = 0; i < itemArray.length; i++) {
        height += itemArray[i].clientHeight;
        heightArray.push(height);
      }
      this.listHeight = heightArray;
    }
  },
  created() {
    getFoods(this.id).then(res => {
      // 获取数据
      this.foodList = res.data;
      // 获取数据完成后关闭骨架图
      this.$parent.successLoadData = true;
      this.$nextTick(() => {
        // 初始化better-scroll
        this.initScroll();
        // 记录每项高度
        this.calculateHeight();
        // 监听滚动事件
        this.foodScroll.on("scroll", pos => {
          // 记录Y轴滚动距离
          this.scrollY = Math.abs(Math.round(pos.y));
        });
      });
    });
  },
  components: {
    CartControl,
    CartDetail,
    SelectedFood
  }
};
</script>

<style scoped>
.food {
  display: flex;
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.food .categories_wrapper {
  width: 76px;
  height: 100%;
}
.food .food_wrapper {
  flex: 1;
  width: calc(100% - 76px);
}
.food .food_categories {
  padding-bottom: 100px;
  width: 100%;
  overflow: hidden;
}
.food .food_categories .categories_item {
  box-sizing: border-box;
  width: 76px;
  padding: 14px 6px;
  font-size: 12px;
  color: #666;
  background-color: #f5f5f5;
  border-bottom: 1px solid #ddd;
  border-left: 2px solid #fff;
  /* 单行省略显示 */
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.food .food_wrapper .list_item {
  background-color: #fff;
}

.food .food_wrapper .list_item:last-child {
  padding-bottom: 100vh;
}
.food .food_wrapper .list_item .title {
  box-sizing: border-box;
  padding: 12px 8px;
  font-size: 14px;
  color: #666;
  font-weight: bold;
  background-color: #f5f5f5;
}
.food .food_wrapper .list_item .title .title_info {
  font-size: 10px;
  color: #999;
  margin-left: 8px;
}
.food .food_wrapper .list_item .food_info {
  display: flex;
  padding: 12px 8px;
  border-bottom: 1px solid #eee;
  box-sizing: border-box;
}
.food .food_wrapper .list_item .food_info > section {
  flex: 1;
}
.food .food_wrapper .list_item .food_info .food_img {
  width: 40px;
  height: 40px;
  margin-right: 8px;
  border-radius: 3px;
}
.food .food_wrapper .list_item .food_info .food_name {
  font-size: 14px;
  color: #333;
  font-weight: bold;
}
.food .food_wrapper .list_item .food_info .food_des {
  font-size: 10px;
  color: #999;
  margin: 5px 0;
}
.food .food_wrapper .list_item .food_info .food_tips {
  font-size: 10px;
  color: #333;
  margin: 5px 0;
}
.food .food_wrapper .list_item .food_info section section {
  display: flex;
  justify-content: space-between;
}
.food .food_wrapper .list_item .food_info section .buy_container .price {
  font-size: 14px;
  color: #f60;
  font-weight: bold;
}
.food .food_wrapper .list_item .food_info section .buy_container .price .icon {
  font-size: 10px;
}
.food .food_wrapper .list_item .food_info section .buy_container .add_btn {
  font-size: 11px;
  padding: 2px 4px;
  border-radius: 3px;
  border: 1px solid #3190e8;
  color: #fff;
  background-color: #0085ff;
}

/* 左侧菜单选中样式 */
.food .food_categories .categories_item.activeClass {
  border-left: 2px solid #0089dc;
  background-color: #fff;
  font-weight: bold;
}
/* 底部购物车栏 */
.food > footer {
  box-sizing: border-box;
  position: fixed;
  left: 0;
  bottom: 0;
  width: 100%;
  height: 40px;
  z-index: 13;
}
.food > footer .cart {
  position: absolute;
  top: -15px;
  left: 14px;
  width: 40px;
  height: 40px;
  text-align: center;
  line-height: 40px;
  border-radius: 50%;
  background-color: #0085ff;
  border: 4px solid #3d3d3f;
  font-size: 20px;
  color: #fff;
  z-index: 999;
}
.food > footer .cart .count {
  position: absolute;
  right: -1px;
  top: -3px;
  display: inline-block;
  width: 12px;
  height: 12px;
  color: #fff;
  font-size: 8px;
  line-height: 12px;
  text-align: center;
  border-radius: 50%;
  background: red;
}
.food .cart_container {
  width: 100%;
  display: flex;
  justify-content: space-between;
  background-color: #3d3d3f;
}
.food .cart_container .total {
  flex: 1;
  padding: 8px 0 5px 80px;
}

.food .cart_container .total .price {
  font-size: 16px;
  color: #fff;
  font-weight: bold;
  margin-bottom: 4px;
}

.food .cart_container .total .fee {
  font-size: 9.6px;
  color: #fff;
}
.food .cart_container .payBtn {
  flex: 0 0 100px;
  background-color: #4cd946;
  font-size: 14px;
  color: #fff;
  text-align: center;
  line-height: 40px;
  font-weight: bold;
}
/* 购物车遮罩层 */

.food .layer_mask {
  position: fixed;
  left: 0;
  top: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.4);
  z-index: 12;
}
</style>