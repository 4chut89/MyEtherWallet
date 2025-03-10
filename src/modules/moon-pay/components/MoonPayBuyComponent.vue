<template>
  <div class="py-8 px-8">
    <!-- ============================================================== -->
    <!-- Currency Select -->
    <!-- ============================================================== -->
    <div class="mb-2">
      <div class="mew-heading-3 textDark--text mb-5">Select currency</div>
      <mew-select
        label="Currency"
        :items="currencyItems"
        :value="selectedCurrency"
        :disabled="loading"
        is-custom
        @input="setCurrency"
      />
    </div>

    <div class="mb-11">
      <div class="mew-heading-3 textDark--text mb-5">
        How much do you want to spend?
      </div>
      <div class="d-flex align-center">
        <mew-input
          v-model="amount"
          type="number"
          :error-messages="amountErrorMessages"
          class="mr-2"
        />
        <mew-select
          v-model="selectedFiat"
          :items="fiatCurrencyItems"
          is-custom
          class="selectedFiat"
        />
      </div>
      <div class="mb-2">You will get</div>
      <div v-if="!loading" class="mb-1">
        <div class="d-flex mb-1 align-center justify-space-between">
          <div class="d-flex align-center mew-heading-3 textDark--text">
            {{ cryptoToFiat }}
            <span class="mew-heading-3 pl-1">{{ selectedCryptoName }}</span>
            <div class="mr-1 textDark--text">&nbsp;≈ {{ plusFeeF }}</div>
            <mew-tooltip style="height: 23px">
              <template #contentSlot>
                <div>
                  {{ includesFeeText }}
                  <br />
                  <br />
                  {{ networkFeeText }}
                  <br />
                  <br />
                  {{ dailyLimit }}
                  <br />
                  {{ monthlyLimit }}
                </div>
              </template>
            </mew-tooltip>
          </div>
        </div>
        <div class="d-flex align-center"></div>
      </div>

      <div v-else class="mb-1">
        <v-skeleton-loader max-width="200px" type="heading" />
      </div>
      <div v-if="!inWallet" class="mt-5">
        <div class="mew-heading-3 textDark--text mb-5">
          Where should we send your crypto?
        </div>
        <mew-input
          v-model="toAddress"
          label="Enter Crypto Address"
          :rules="[isValidToAddress]"
          :error-messages="addressErrorMessages"
        />
      </div>
    </div>
    <div class="mb-1">
      <mew-button
        btn-size="xlarge"
        has-full-width
        :disabled="disableBuy"
        :title="buyBtnTitle"
        :is-valid-address-func="isValidToAddress"
        @click.native="buy"
      />
    </div>
  </div>
</template>

<script>
import MultiCoinValidator from 'multicoin-address-validator';
import { ERROR, Toast } from '@/modules/toast/handler/handlerToast';
import { isEmpty, cloneDeep, isEqual } from 'lodash';
import BigNumber from 'bignumber.js';
import { mapGetters, mapActions, mapState } from 'vuex';
import { fromWei, toBN } from 'web3-utils';
import Web3 from 'web3';
import nodeList from '@/utils/networks';
import {
  formatFloatingPointValue,
  formatFiatValue
} from '@/core/helpers/numberFormatHelper';
import { getCurrency } from '@/modules/settings/components/currencyList';
import { buyContracts } from './tokenList';
export default {
  name: 'ModuleBuyEth',
  props: {
    orderHandler: {
      type: Object,
      default: () => {}
    },
    defaultCurrency: {
      type: Object,
      default: () => {}
    },
    inWallet: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      selectedCurrency: this.defaultCurrency,
      loading: true,
      selectedFiat: {
        name: 'USD',
        value: 'USD',
        // eslint-disable-next-line
        img: require(`@/assets/images/currencies/USD.svg`)
      },
      fetchedData: {},
      currencyRates: [],
      amount: '300',
      toAddress: '',
      validToAddress: false,
      gasPrice: '0',
      web3Connections: {},
      simplexQuote: {},
      showMoonpay: true
    };
  },
  computed: {
    ...mapGetters('global', ['network', 'getFiatValue']),
    ...mapState('wallet', ['address']),
    ...mapState('external', ['currencyRate', 'coinGeckoTokens']),
    ...mapGetters('external', ['contractToToken']),
    includesFeeText() {
      return `Includes ${this.percentFee} fee (${
        formatFiatValue(this.minFee, this.currencyConfig).value
      } min)`;
    },
    networkFeeText() {
      return `${
        this.selectedCurrency.symbol
      } network fee (for transfers to your wallet) ~${
        formatFiatValue(this.networkFeeToFiat, this.currencyConfig).value
      }`;
    },
    dailyLimit() {
      const value = BigNumber(this.fiatMultiplier).times(12000);
      return `Daily limit: ${
        formatFiatValue(value.toString(), this.currencyConfig).value
      }`;
    },
    monthlyLimit() {
      const value = BigNumber(this.fiatMultiplier).times(50000);
      return `Monthly limit: ${
        formatFiatValue(value.toString(), this.currencyConfig).value
      }`;
    },
    currencyConfig() {
      const fiat = this.selectedFiat.value;
      const rate = this.currencyRate[fiat];
      const currency = fiat;
      return { rate, currency };
    },
    fiatMultiplier() {
      if (this.hasData) {
        const selectedCurrencyPrice = this.fetchedData[0].conversion_rates.find(
          item => item.fiat_currency === this.selectedFiatName
        );
        return selectedCurrencyPrice
          ? BigNumber(selectedCurrencyPrice.exchange_rate)
          : toBN(1);
      }
      return toBN(1);
    },
    selectedFiatName() {
      return this.selectedFiat.name;
    },
    actualAddress() {
      return this.inWallet ? this.address : this.toAddress;
    },
    actualValidAddress() {
      return this.inWallet ? true : this.validToAddress;
    },
    networkFee() {
      return fromWei(BigNumber(this.gasPrice).times(21000).toString());
    },
    priceOb() {
      return !isEmpty(this.fetchedData)
        ? this.fetchedData[0].prices.find(
            item => item.fiat_currency === this.selectedFiatName
          )
        : { crypto_currency: 'ETH', fiat_currency: 'USD', price: '3379.08322' };
    },
    networkFeeToFiat() {
      return BigNumber(this.networkFee).times(this.priceOb.price).toString();
    },
    minFee() {
      return BigNumber(4.43).times(this.fiatMultiplier).toString();
    },
    plusFee() {
      const fee = this.isEUR
        ? BigNumber(BigNumber(0.7).div(100)).times(this.amount)
        : BigNumber(BigNumber(3.25).div(100)).times(this.amount);
      const withFee = fee.gt(this.minFee)
        ? BigNumber(this.amount).minus(fee)
        : BigNumber(this.amount).minus(fee).minus(this.minFee);
      return withFee.minus(this.networkFeeToFiat).toString();
    },
    plusFeeF() {
      return formatFiatValue(this.plusFee, this.currencyConfig).value;
    },
    percentFee() {
      return this.isEUR ? '0.7%' : '3.25%';
    },
    selectedCryptoName() {
      return this.selectedCurrency.symbol;
    },
    isEUR() {
      return this.selectedFiatName === 'EUR' || this.selectedFiatName === 'GBP';
    },
    disableBuy() {
      return (
        (!this.inWallet && !this.actualValidAddress) ||
        this.loading ||
        this.amountErrorMessages !== ''
      );
    },
    buyBtnTitle() {
      return 'BUY NOW';
    },
    amountErrorMessages() {
      const moonpayMax = this.max.moonpay;
      const simplexMax = this.max.simplex;
      if (BigNumber(this.amount).isNaN() || BigNumber(this.amount).eq(0)) {
        return 'Amount required';
      }
      if (BigNumber(this.amount).lt(0)) {
        return `Amount can't be negative`;
      }
      if (this.min.gt(this.amount)) {
        return `Amount can't be below provider's minimum: ${this.min.toFixed()} ${
          this.selectedFiatName
        }`;
      }
      if (
        moonpayMax.lt(BigNumber(this.amount)) &&
        simplexMax.lt(BigNumber(this.amount))
      ) {
        return `Amount can't be above provider's maximum: ${simplexMax.toFixed()} ${
          this.selectedFiatName
        }`;
      }
      return '';
    },
    addressErrorMessages() {
      if (!this.actualValidAddress && !isEmpty(this.toAddress)) {
        return 'Invalid Address';
      }
      return '';
    },
    currencyItems() {
      const tokenList = new Array();
      for (const contract of buyContracts) {
        const token = this.contractToToken(contract);
        if (token) tokenList.push(token);
      }
      const imgs = tokenList.map(item => {
        return item.img;
      });
      const tokensListWPrice =
        this.currencyRates.length > 0
          ? tokenList.map(token => {
              const priceRate = this.currencyRates.find(rate => {
                return rate.crypto_currency === token.symbol;
              });
              const actualPrice = priceRate?.quotes.find(quote => {
                return quote.fiat_currency === this.selectedFiatName;
              });
              token.price = formatFiatValue(
                actualPrice ? actualPrice.price : '0',
                this.currencyConfig
              ).value;
              token.value = token.name;
              token.name = token.symbol;
              return token;
            })
          : tokenList;
      const returnedArray = [
        {
          text: 'Select Token',
          imgs: imgs.splice(0, 3),
          total: `${tokenList.length}`,
          divider: true,
          selectLabel: true
        },
        ...tokensListWPrice
      ];
      return returnedArray;
    },
    hasData() {
      return !isEmpty(this.fetchedData);
    },
    cryptoToFiat() {
      return this.showMoonpay
        ? this.moonpayCryptoAmount
        : this.simplexCryptoAmount;
    },
    moonpayCryptoAmount() {
      return formatFloatingPointValue(
        BigNumber(this.plusFee).div(this.priceOb.price).toString()
      ).value;
    },
    simplexCryptoAmount() {
      return formatFloatingPointValue(this.simplexQuote.crypto_amount).value;
    },
    fiatCurrencyItems() {
      const arrItems = this.hasData
        ? this.fetchedData[0].fiat_currencies.filter(item => item !== 'RUB')
        : ['USD'];
      return getCurrency(arrItems);
    },
    max() {
      if (this.hasData) {
        const moonpayMax = this.fetchedData[0]?.limits.find(
          item => item.fiat_currency === this.selectedFiatName
        );
        const simplexMax = this.fetchedData[1]?.limits.find(
          item => item.fiat_currency === this.selectedFiatName
        );
        return {
          moonpay: moonpayMax
            ? BigNumber(moonpayMax.limit.max)
            : BigNumber(12000),
          simplex: simplexMax
            ? BigNumber(simplexMax.limit.max)
            : BigNumber(12000)
        };
      }
      return {
        moonpay: BigNumber(12000),
        simplex: BigNumber(12000)
      };
    },
    min() {
      if (this.hasData) {
        const foundLimit = this.fetchedData[0].limits.find(
          item => item.fiat_currency === this.selectedFiatName
        );
        return foundLimit ? BigNumber(foundLimit.limit.min) : BigNumber(30);
      }
      return BigNumber(30);
    },
    hideSimplex() {
      return (
        this.selectedCryptoName === 'USDC' || this.selectedCryptoName === 'USDT'
      );
    }
  },
  watch: {
    selectedCurrency: {
      handler: function (newVal, oldVal) {
        if (!isEqual(newVal, oldVal)) {
          this.fetchCurrencyData();
        }
        this.$emit('selectedCurrency', newVal);
      },
      deep: true
    },
    selectedFiat: {
      handler: function (newVal, oldVal) {
        if (!isEqual(newVal, oldVal)) {
          this.amount = newVal.name != 'JPY' ? '300' : '30000';
          this.$emit('selectedFiat', newVal);
        }
      },
      deep: true
    },
    network: {
      handler: function () {
        this.selectedCurrency = this.defaultCurrency;
      },
      deep: true
    },
    orderHandler: {
      handler: function () {
        this.fetchCurrencyData();
      },
      deep: true
    },
    amount: {
      handler: function (newVal) {
        const simplexMax = this.max.simplex;
        this.checkMoonPayMax();
        if (
          simplexMax.lt(newVal) ||
          isEmpty(newVal) ||
          this.min.gt(newVal) ||
          isNaN(newVal)
        ) {
          this.loading = true;
        } else {
          this.loading = false;
          this.getSimplexQuote();
        }
      }
    },
    validToAddress: {
      handler: function (newVal) {
        if (!newVal) return;
        this.$emit('toAddress', this.toAddress);
        this.getSimplexQuote();
      }
    },
    toAddress(newVal) {
      this.validToAddress = this.isValidToAddress(newVal);
    },
    coinGeckoTokens: {
      handler: function () {
        this.fetchCurrencyData();
      }
    }
  },
  mounted() {
    this.fetchCurrencyData();
  },
  methods: {
    ...mapActions('global', ['setNetwork']),
    async fetchGasPrice() {
      const supportedNodes = {
        ETH: 'ETH',
        BNB: 'BSC',
        MATIC: 'MATIC'
      };
      const nodeType = !supportedNodes[this.selectedCurrency.symbol]
        ? 'ETH'
        : supportedNodes[this.selectedCurrency.symbol];
      const node = nodeList[nodeType];
      if (!this.web3Connections[nodeType]) {
        const web3 = new Web3(node[0].url);
        this.web3Connections[nodeType] = web3;
      }
      this.gasPrice = await this.web3Connections[nodeType].eth.getGasPrice();
    },
    isLT(num, num2) {
      return BigNumber(num).lt(num2);
    },
    isValidToAddress(address) {
      return MultiCoinValidator.validate(address, this.selectedCurrency.symbol);
    },
    checkMoonPayMax() {
      const moonpayMax = this.max.moonpay;
      const hideMoonpay = this.isLT(moonpayMax, this.amount);
      this.$emit('hideMoonpay', hideMoonpay);
    },
    setCurrency(e) {
      this.selectedCurrency = e;
    },
    fetchCurrencyData() {
      this.loading = true;
      this.fetchData = {};
      this.fetchGasPrice();
      this.orderHandler
        .getSupportedFiatToBuy(this.selectedCurrency.symbol)
        .then(res => {
          this.orderHandler.getFiatRatesForBuy().then(res => {
            this.currencyRates = cloneDeep(res);
            this.loading = false;
          });
          this.fetchedData = Object.assign({}, res);
        })
        .catch(e => {
          Toast(e, {}, ERROR);
        });
      this.getSimplexQuote();
    },
    getSimplexQuote() {
      if (
        this.hideSimplex ||
        !this.actualValidAddress ||
        isEmpty(this.amount) ||
        this.min.gt(this.amount) ||
        isNaN(this.amount)
      )
        return;
      this.loading = true;
      this.simplexQuote = {};
      this.orderHandler
        .getSimplexQuote(
          this.selectedCryptoName,
          this.selectedFiatName,
          this.amount,
          this.actualAddress
        )
        .then(res => {
          this.simplexQuote = Object.assign({}, res);
          this.loading = false;
          this.$emit('simplexQuote', this.simplexQuote);
          this.compareQuotes();
        })
        .catch(e => {
          Toast(e, {}, ERROR);
        });
    },
    compareQuotes() {
      const moonpayMax = this.max.moonpay;
      // Moonpay has better rate and is not above max
      this.showMoonpay = this.isLT(moonpayMax, this.amount) // max < amount
        ? false
        : this.isLT(this.simplexQuote.crypto_amount, this.moonpayCryptoAmount);
    },
    buy() {
      const buyObj = {
        cryptoToFiat: this.moonpayCryptoAmount,
        selectedCryptoName: this.selectedCryptoName,
        plusFeeF: this.plusFeeF,
        includesFeeText: this.includesFeeText,
        networkFeeText: this.networkFeeText,
        dailyLimit: this.dailyLimit,
        monthlyLimit: this.monthlyLimit,
        fiatAmount: this.amount
      };
      this.checkMoonPayMax();
      this.$emit('success', [
        this.simplexQuote,
        this.toAddress,
        buyObj,
        1,
        this.selectedCurrency,
        this.selectedFiat
      ]);
    }
  }
};
</script>

<style lang="scss" scoped>
// Force set button border color(greyMedium) for not selected buttons
.not-selected {
  border: 1px solid var(--v-greyMedium-base);
}
.icon-holder {
  border: 2px solid var(--v-greyMedium-base);
  border-radius: 100px;
  width: 20px;
  height: 20px;
}
.section-block {
  height: 145px;
  border-radius: 12px;
  left: 0px;
  top: 0px;
  box-sizing: border-box;
  border: 2px solid var(--v-greyMedium-base);
  flex: none;
  order: 0;
  align-self: stretch;
  flex-grow: 0;
  margin: 8px 0px;
  position: relative;
}
.section-block:hover {
  cursor: pointer;
  border: 2px solid #1eb19b;
  background-color: #e5eaee;
}
.selected {
  border: 2px solid #1eb19b;
}
.provider-logo {
  position: absolute;
  top: 18px;
  right: 20px;
}
.selectedFiat {
  max-width: 120px;
}
</style>
