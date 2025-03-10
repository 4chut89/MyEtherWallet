<template>
  <div>
    <div v-if="loading" class="d-flex align-center justify-center">
      <v-progress-circular indeterminate color="greenPrimary" />
    </div>
    <div v-if="!loading" class="d-flex align-center pa-6 greyLight rounded">
      <v-icon size="80" color="greenPrimary" class="mr-3">
        mdi-check-circle-outline
      </v-icon>
      <div>
        <div class="mew-heading-2 mb-2 greenPrimary--text">
          {{ $t('ens.is-available') }}
        </div>
        <div class="mew-heading-2">{{ name }}</div>
      </div>
    </div>
    <div class="pa-1 mt-3 mb-7 text-center">
      {{ $t('ens.request.about-domain') }}
    </div>
    <mew-select
      :has-filter="true"
      :label="$t('ens.request.select-duration')"
      :items="items"
      @input="setDuration"
    />

    <div class="font-weight-bold text-center">
      {{ $t('ens.request.estimated-price') }}: {{ rentPriceETH }}
      {{ $t('common.currency.eth') }} ({{ rentPriceUSD }})
    </div>
    <div class="d-flex justify-center mt-6">
      <mew-button
        :title="$t('common.next')"
        btn-size="xlarge"
        @click.native="onClick"
      />
    </div>
  </div>
</template>

<script>
import { formatFloatingPointValue } from '@/core/helpers/numberFormatHelper';
import { mapGetters } from 'vuex';

export default {
  components: {},
  props: {
    getRentPrice: {
      default: function () {
        return {};
      },
      type: Function
    },
    loading: {
      default: false,
      type: Boolean
    },
    name: {
      default: '',
      type: String
    }
  },
  data() {
    return {
      duration: 0,
      rentPriceETH: '',
      rentPriceUSD: ''
    };
  },
  computed: {
    items() {
      const items = [];
      for (let i = 0; i < 20; i++) {
        items.push({
          name: i + 1 + ' ' + `year${i < 1 ? '' : 's'}`,
          value: (i + 1).toString()
        });
      }
      return items;
    }
  },
  mounted() {
    this.rentPrice();
  },
  methods: {
    ...mapGetters('global', ['getFiatValue']),
    rentPrice() {
      if (this.duration > 0) {
        return this.getRentPrice(this.duration).then(resp => {
          if (resp) {
            this.rentPriceETH = formatFloatingPointValue(resp.eth).value;
            this.rentPriceUSD = this.getFiatValue()(resp.usd);
          }
        });
      }
    },
    onClick() {
      this.$emit('onRequest', this.duration);
    },
    setDuration(item) {
      this.duration = parseInt(item.value);
      this.rentPrice();
    }
  }
};
</script>
