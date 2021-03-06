<template>
  <v-layout class="container_layout" fill-height>
    <v-flex class="LeftSide" column>
      <!-- 实时动态开始 -->
      <Activity :activity-data="activityData" />
      <!-- 实时动态结束 -->
      <!-- k线图开始 -->
      <Chart />
      <client-only>
        <v-flex class="orders-area">
          <v-tabs v-model="currentTab" slider-color="cybex" dark height="40">
            <v-tab :ripple="false">
              {{ $t('exchange.order-table.tab-title.open-order') }}
              <!-- 数字与汉字无法行内对齐 添加特殊样式 -->
              <span class="ml-1" :class="{ 'zh-fixed-num': locale == 'zh' }"
                >({{ openOrderRowLen }})</span
              >
            </v-tab>
            <v-tab :ripple="false">{{
              $t('exchange.order-table.tab-title.history-order')
            }}</v-tab>
            <v-tab :ripple="false">{{
              $t('exchange.order-table.tab-title.history-trade')
            }}</v-tab>
            <v-tab-item>
              <ExchangeOpenOrder
                :white-flag="'white'"
                :enable-interval="currentTab == 0"
                :order-filter="orderFilter"
                @update-row-length="(v) => (openOrderRowLen = v)"
              />
            </v-tab-item>
            <v-tab-item>
              <ExchangeOrderHistory
                :white-flag="'white'"
                :enable-interval="currentTab == 1"
                :order-filter="orderFilter"
              />
            </v-tab-item>
            <v-tab-item>
              <ExchangeTradeHistory
                :white-flag="'white'"
                :enable-interval="currentTab == 2"
                :order-filter="orderFilter"
                :mode="'exchange'"
              />
            </v-tab-item>
          </v-tabs>
        </v-flex>
      </client-only>
    </v-flex>
    <!-- right side end-->
    <v-flex class="RightSide">
      <v-flex class="exchange-book-trade-container" d-flex>
        <client-only>
          <!-- 交易实时价格开始 -->
          <OrderBook @set-form-price="setFormPrice" />
          <!-- 交易实时价格结束 -->
          <!-- 交易记录列表开始-->
          <MarketTrades :trades="trades" @set-form-price="setFormPrice" />
          <!-- 交易记录列表结束-->
        </client-only>
      </v-flex>
      <!-- 交易表单开始 -->
      <v-flex class="ExchangeSellingFormContainer" d-flex>
        <v-flex class="FlexHalf-3">
          <client-only>
            <ExchangeForm
              is-buy
              :balance-digits="balanceDigits"
              :form-data="formData"
              @create-trade-success="createTradeSuccess"
            />
          </client-only>
        </v-flex>
        <v-flex class="FlexHalf-4">
          <client-only>
            <ExchangeForm
              :balance-digits="balanceDigits"
              :form-data="formData"
              @create-trade-success="createTradeSuccess"
            />
          </client-only>
        </v-flex>
      </v-flex>
      <!-- 交易表单结束 -->
    </v-flex>
  </v-layout>
</template>

<script>
import { mapGetters } from 'vuex'
import CybexDotClient from '~/lib/CybexDotClient.js'
import config from '~/lib/config/config.js'
import utils from '~/components/mixins/utils'
export default {
  components: {
    Activity: () => import('~/components/exchange/ExchangeActivity.vue'),
    Chart: () => import('~/components/exchange/ExchangeChart.vue'),
    OrderBook: () => import('~/components/exchange/ExchangeOrderBook.vue'),
    MarketTrades: () =>
      import('~/components/exchange/ExchangeMarketTrades.vue'),
    ExchangeForm: () => import('~/components/exchange/ExchangeForm.vue'),
    ExchangeOpenOrder: () =>
      import('~/components/exchange/ExchangeHistoryOpenOrder.vue'),
    ExchangeOrderHistory: () =>
      import('~/components/exchange/ExchangeHistoryOrder.vue'),
    ExchangeTradeHistory: () =>
      import('~/components/exchange/ExchangeHistoryTrade.vue')
    // VersionDialog: () => import('~/components/VersionDialog.vue')
  },
  mixins: [utils],

  data() {
    return {
      intervalActivity: null,
      activityData: {},
      currentTab: 0,
      createTrade: false,
      openOrderRowLen: 0,
      trades: [],

      formData: {
        buyPrice: null,
        sellPrice: null,
        buyTotal: null,
        sellAmount: null
      }
    }
  },
  computed: {
    ...mapGetters({
      locale: 'i18n/locale',
      tradesRefreshRate: 'exchange/tradesRefreshRate'
    }),
    balanceDigits() {
      return { base: this.basePrecision, quote: this.quotePrecision }
    },
    orderFilter() {
      const pair = this.hideOtherPair
        ? { base_id: this.base_id, quote_id: this.quote_id }
        : { base_id: null, quote_id: null }
      return Object.assign({}, pair, { refresh: this.createTrade })
    },
    digitsPrice() {
      return this.pair.book.last_price || 2
    }
  },

  async mounted() {
    await this.fetchData()
  },
  beforeDestroy() {
    clearInterval(this.intervalActivity)
  },
  methods: {
    setFormPrice(info) {
      const newData = {
        buyPrice: parseFloat(info.price).toFixed(this.digitsPrice),
        sellPrice: parseFloat(info.price).toFixed(this.digitsPrice),
        buyTotal: null,
        sellAmount: null,
        autoSet: false
      }
      // 用户点击Orderbook中的卖单 设置买单Total,  通过price计算amount 向下取值
      // 反之点击Orderbook中的买单 设置卖单Amount, 通过price计算Total 向上取值
      if (info.side === 'sell') {
        newData.buyTotal = parseFloat(info.total)
        // 清空sell
        newData.sellAmount = 'clean'
      } else if (info.side === 'buy') {
        newData.sellAmount = parseFloat(info.amount)
        newData.buyTotal = 'clean'
      }
      this.formData = Object.assign({}, this.formData, newData)
    },
    createTradeSuccess() {
      // 重新刷新 open Order
      this.createTrade = !this.createTrade
    },
    fetchData() {
      this.currentTab = 0
      this.formData = {
        buyPrice: null,
        sellPrice: null,
        buyTotal: null,
        sellAmount: null
      }
      this.intervalActivity = null

      this.$eventHandle(this.loadTicker)
        .then()
        .catch((e) => {})
    },

    async loadTicker() {
      const func = async () => {
        // 24小时数据
        const ticker = await CybexDotClient.getTicker(
          CybexDotClient.info.tradePairHash
        )
        this.$store.commit('exchange/SET_CURRENT_RTE_PRICE', {
          price:
            (ticker.latest_matched_price * 10 ** this.priceMatchedPrecision) /
            config.pricePrecision,
          legalPrice: null
        })
        this.activityData = {
          latest:
            (ticker.latest_matched_price * 10 ** this.priceMatchedPrecision) /
            config.pricePrecision,
          low:
            (ticker.one_day_lowest_price * 10 ** this.priceMatchedPrecision) /
            config.pricePrecision,
          high:
            (ticker.one_day_highest_price * 10 ** this.priceMatchedPrecision) /
            config.pricePrecision,
          quote_volume: ticker.one_day_trade_volume / 10 ** this.quotePrecision
        }
      }
      await func()

      // form price只需要初始化第一次 其余由用户自己交互
      const price = this.activityData.latest
      const newData = {
        buyPrice: parseFloat(price).toFixed(this.digitsPrice),
        sellPrice: parseFloat(price).toFixed(this.digitsPrice),
        autoSet: true
      }
      this.formData = Object.assign({}, this.formData, newData)

      if (!this.intervalActivity) {
        this.intervalActivity = setInterval(async function() {
          await func()
        }, this.tradesRefreshRate)
      }
    }
  }
}
</script>
