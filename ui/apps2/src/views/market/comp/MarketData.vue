<template>
  <div>
    <div :class="`tipDiv marketReward ${handleGetClass(thisMarket.mid)}`" v-if="Number(token)">
      <div class="flexb" v-if="isList">
        <div class="flexa symbolInfo">
          <img class="imgCoin" :src="thisMarket.sym0Data.imgUrl" :onerror="errorCoinImg"/>
          <span>{{ thisMarket.symbol0 }}</span>
          <span class="and">+</span>
          <img class="imgCoin" :src="thisMarket.sym1Data.imgUrl" :onerror="errorCoinImg"/>
          <span>{{ thisMarket.symbol1 }}</span>
        </div>
        <span @click="handleJoin" class="green">{{ $t('market.manage') }}</span>
      </div>
      <div class="flex">
        <span>{{ $t('mine.accPools') }}: </span>
        <span class="">{{ nowMarket.getNum1 || `0.0000` }} {{thisMarket.symbol0}} / {{ nowMarket.getNum2 || `0.0000`}} {{thisMarket.symbol1}}</span>
      </div>
      <div class="flex">
        <span>{{ $t('market.capital') }}: </span>
        <span v-if="!marketData.length || !Number(marketData[0])" class="tip maxW">
          <span>{{ $t('market.anthorOne') }}</span>
          <!-- <span v-if="isList" class="green" @click="handleJoin">立即操作</span> -->
        </span>
        <span v-else>{{ `${marketData[0]} ${thisMarket.symbol0}` }} / {{ `${marketData[1]} ${thisMarket.symbol1}` }}</span>
      </div>
      <div class="flexa">
        <span>{{ $t('market.marketReward') }}: </span>
        <span :class="{'green': marketReward > 0, 'red': marketReward < 0}">{{ marketReward }} </span>
        <span @click="direction = !direction" class="flexa ml10">
          <span>{{ direction ? thisMarket.symbol1 : thisMarket.symbol0 }}</span><span
            class="small">（{{ $t('market.has', {coin: direction ? thisMarket.symbol0 : thisMarket.symbol1}) }}）</span>
          <img class="changeImg" v-if="direction" src="@/assets/img/dex/price_switch_icon_green_left.svg" alt="">
          <img class="changeImg" v-else src="@/assets/img/dex/price_switch_icon_green_right.svg" alt="">
        </span>
        <img class="qusTip" src="@/assets/img/dex/tips_icon_btn.svg" @click="showMarketTip = !showMarketTip">
      </div>
      <div class="flexa">
        <span>{{ $t('market.marketTime') }}: </span>
        <span>{{ $t('market.timer', {
            days: marketTime.days,
            hours: marketTime.hours,
            mins: marketTime.minutes,
            secs: marketTime.seconds
          }) }}</span>
        <span class="tip">（{{ $t('market.pl') }}: 
          <span :class="{'green': Number(percent) > 0, 'red': Number(percent < 0)}">
            {{ percent }}%
          </span>）
        </span>
        <!-- <span>{{ JSON.stringify(marketTime) }}</span> -->
      </div>
    </div>

    <el-dialog
      class="myDialog"
      :visible.sync="showMarketTip">
      <MarketTip v-if="showMarketTip"/>
    </el-dialog>
  </div>
</template>

<script>
import axios from "axios";
import { mapState } from 'vuex';
import { toFixed, accSub, accMul, accDiv, getClass, getMarketTime } from '@/utils/public';
import { sellToken } from '@/utils/logic';
import MarketTip from '../popup/MarketTip';
export default {
  components: {
    MarketTip
  },
  props: {
    token: {
      type: String,
      default: '0'
    },
    thisMarket: {
      type: Object,
      default: function tmt() {
        return {}
      }
    },
    capital: {
      type: Array,
      default: function cp() {
        return []
      }
    },
    isList: {
      type: Boolean,
      default: false,
    },
    startTime: {
      type: String,
      default: '0'
    }
  },
  data() {
    return {
      errorCoinImg: 'this.src="https://ndi.340wan.com/eos/eosio.token-eos.png"',
      nowMarket: {},
      marketData: [],
      direction: true,
      sTime: '0',
      timer: null,
      marketTime: {
        days: 0,
        hours: '00',
        minutes: '00',
        seconds: '00'
      },
      showMarketTip: false
    }
  },
  mounted() {
    this.handleGetTime()
  },
  beforeDestroy() {
    clearTimeout(this.timer)
  },
  watch: {
    startTime: {
      handler: function st() {
        this.sTime = this.startTime;
      },
      immediate: true
    },
    sTime() {
      this.handleGetTime()
    },
    token: {
      handler: function tk() {
        this.handleGetNowMarket();
      },
      immediate: true
    },
    thisMarket: {
      handler: function tm() {
        if (this.capital.length) {
          this.marketData = this.capital;
          return
        }
        this.handleGetMarketData()
        this.handleGetNowMarket();
      },
      immediate: true
    }
  },
  computed: {
    ...mapState({
      // 箭头函数可使代码更简练
      baseConfig: state => state.sys.baseConfig, // 基础配置 - 默认为{}
      scatter: state => state.app.scatter,
    }),
    marketReward() {
      if (!this.marketData.length || !Number(this.marketData[0])  || !this.nowMarket.getNum1) {
        return '0.0000';
      }
      const sym0 = accSub(parseFloat(this.nowMarket.getNum1), this.marketData[0]);
      const sym1 = accSub(parseFloat(this.nowMarket.getNum2), this.marketData[1]);
      const price = accDiv(parseFloat(this.nowMarket.getNum2), parseFloat(this.nowMarket.getNum1));
      if (this.direction) {
        const reward = sym1 + sym0 * price;
        return toFixed(reward, this.thisMarket.decimal1)
      }
      const reward = sym0 + sym1 / price;
      return toFixed(reward, this.thisMarket.decimal0)
    },
    percent() {
      if (!this.marketData.length || !this.nowMarket.getNum1) {
        return '0.0000';
      }
      const sym0 = accSub(parseFloat(this.nowMarket.getNum1), this.marketData[0]);
      const sym1 = accSub(parseFloat(this.nowMarket.getNum2), this.marketData[1]);
      const price = accDiv(parseFloat(this.nowMarket.getNum2), parseFloat(this.nowMarket.getNum1));
      const reward = sym0 + sym1 / price;
      let mD = accMul(this.marketData[0], 2)
      let p = accDiv(reward, mD)
      p = accMul(p, 100);
      return toFixed(p, 2)
    }
  },
  methods: {
    handleGetTime() {
      clearTimeout(this.timer)
      if (Number(this.sTime)) {
        this.marketTime = getMarketTime(this.sTime)
        this.timer = setTimeout(() => {
          this.handleGetTime()
        }, 1000);
        return
      }
      this.marketTime = {
        days: 0,
        hours: '00',
        minutes: '00',
        seconds: '00'
      }
    },
    handleGetNowMarket() {
      try {
        const inData = {
          poolSym0: this.thisMarket.reserve0.split(' ')[0],
          poolSym1: this.thisMarket.reserve1.split(' ')[0],
          poolToken: this.thisMarket.liquidity_token,
          sellToken: this.token
        }
        const nowMarket = sellToken(inData);
        nowMarket.getNum1 = toFixed(nowMarket.getNum1, 4)
        nowMarket.getNum2 = toFixed(nowMarket.getNum2, 4)
        this.nowMarket = nowMarket;
      } catch(error) {
        setTimeout(() => {
          this.handleGetNowMarket()
        }, 200);
      }
    },
    handleGetMarketData() {
      const params = {
        user: this.scatter.identity.accounts[0].name,
        mid: this.thisMarket.mid || this.$route.params.mid,
      }
      axios.get('https://dfsinfoapi.sgxiang.com/dapi/changelogdata', {params}).then((result) => {
        const res = result.data;
        this.marketData = [];
        this.sTime = '0'
        if (!result.data.logs.length) {
          return
        }
        const newArr = []
        for (const key in res) {
          if (key !== 'logs' && key !== 'tag_log_format_block_time' && key !== 'tag_log_utc_block_time') {
            if (key !== 'EOS') {
              newArr.push(toFixed(res[key], this.thisMarket.decimal1 || 4))
            } else {
              newArr.unshift(toFixed(res[key], this.thisMarket.decimal0 || 4))
            }
          }
        }
        this.sTime = res.tag_log_utc_block_time
        this.marketData = newArr;
      })
    },
    handleGetClass(mid) {
      return getClass(mid)
    },
    handleJoin() {
      this.$emit('listenToMarket', this.thisMarket)
    },
  },
}
</script>

<style lang="scss" scoped>
.green{
  color: #07D79B;
}
.red{
  color: #E9574F;
}
.ml10{
  margin-left: 10px;
}
.tipDiv{
  border: 1px solid #e3e3e3;
  margin-top: 40px;
  border-radius: 20px;
  padding: 20px 40px;
  font-size: 26px;
  overflow: hidden;
}
.marketReward{
  &>div{
    margin-top: 10px;
    &:first-child{
      margin-top: 0px;
    }
    &>span{
      line-height: 37px;
      &:first-child{
        margin-right: 10px;
      }
    }
    .small{
      font-size: 26px;
    }
    .changeImg{
      width: 30px;
    }
    .maxW{
      max-width: 420px;
    }
    .qusTip{
      padding: 0 0 0 10px;
      width: 30px;
      font-size: 24px;
    }
  }
  .symbolInfo{
    .imgCoin{
      width: 50px;
      height: 50px;
      margin-right: 10px;
      border-radius: 60px;
      overflow: hidden;
    }
    .and{
      margin: 0 20px;
    }
  }
}
.myDialog{
  /deep/ .el-dialog{
    position: relative;
    margin: auto;
    width: 590px;
    border-radius: 20px;
    .el-dialog__body,
    .el-dialog__header{
      padding: 0;
    }
  }
}
</style>
