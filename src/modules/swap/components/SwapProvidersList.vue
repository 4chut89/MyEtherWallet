<template>
  <div class="modules--swap--components--swap-providers-list my-5">
    <!--
    =====================================================================================
      Provider Rate Row
    =====================================================================================
    -->
    <v-item-group
      v-if="step >= 1 && toTokenSymbol && !hasProviderError"
      :value="0"
    >
      <v-row no-gutters>
        <div v-if="providersList.length > 0" class="mew-heading-3 mb-5 pl-4">
          Select rate
        </div>
        <v-col
          v-for="(quote, idx) in providersList"
          :key="`quote-${idx}`"
          cols="12"
          class="mb-1"
        >
          <v-item v-slot="{ active, toggle }" :ref="`card${idx}`">
            <v-container
              :class="[
                active ? 'rate-active' : '',
                'd-flex align-center rate py-0 px-1'
              ]"
              @click="
                toggle();
                setProvider(idx, true);
              "
            >
              <v-row no-gutters class="align-center justify-start">
                <!--
                =====================================================================================
                  Token Image, Rate & Best Rate Chip
                  xs = 10 / total = 10
                  md = 7 / total = 7
                =====================================================================================
                -->
                <v-col cols="10" sm="7">
                  <v-container class="pa-2">
                    <v-row
                      class="align-center justify-start pl-5 pr-1 py-3 py-sm-4"
                    >
                      <!--
                    =====================================================================================
                      Token Icon
                    =====================================================================================
                    -->
                      <mew-token-container size="small" :img="toTokenIcon" />
                      <!--
                      =====================================================================================
                        Rate
                      =====================================================================================
                      -->
                      <div
                        class="d-block d-sm-flex mx-2 mx-sm-4 align-center justify-start"
                      >
                        <div
                          v-if="bestRate !== null && bestRate === quote.rate"
                          class="greenPrimary--text font-weight-medium mew-label order-sm-12 pl-sm-2"
                        >
                          Best Rate
                        </div>
                        <div
                          class="d-flex order-sm-1 justify-start align-center"
                        >
                          <div class="mb-0 mew-heading-3 font-weight-medium">
                            {{ quote.amount }} {{ toTokenSymbol }}
                          </div>
                          <mew-tooltip
                            v-if="quote.amount && quote.amount !== ''"
                            class="pl-1"
                            :text="quote.tooltip"
                          />
                        </div>
                      </div>
                    </v-row>
                  </v-container>
                </v-col>
                <!--
                =====================================================================================
                  Provider Image & Checkbox
                  xs = 2 / total = 12
                  sm = 5 / total = 12
                =====================================================================================
                -->
                <v-col cols="2" sm="5">
                  <v-row class="align-center justify-end pr-3">
                    <mew-checkbox :value="active" />
                  </v-row>
                </v-col>
              </v-row>
            </v-container>
          </v-item>
        </v-col>
      </v-row>
    </v-item-group>

    <!--
    =====================================================================================
      Show More Providers Button
    =====================================================================================
    -->
    <div
      v-if="step >= 1 && providersCut > 0 && toTokenSymbol && !hasProviderError"
      class="cursor--pointer user-select--none greenPrimary--text mt-7 ml-4"
      @click="showMore = !showMore"
    >
      {{ moreProvidersText }}
      <v-icon v-show="!showMore" small color="greenPrimary"
        >mdi-arrow-down</v-icon
      >
      <v-icon v-show="showMore" small color="greenPrimary">mdi-arrow-up</v-icon>
    </div>
    <!--
    =====================================================================================
      Providers Message
    =====================================================================================
    -->
    <app-user-msg-block
      v-if="step >= 1 && hasProviderError && !isLoading"
      :message="providersError"
      :is-alert="false"
      container-padding="pa-4 py-6"
    />
  </div>
</template>
<script>
import AppUserMsgBlock from '@/core/components/AppUserMsgBlock';
import { formatFloatingPointValue } from '@/core/helpers/numberFormatHelper';
import { isArray } from 'lodash';
const MAX_PROVIDERS = 3;
export default {
  name: 'SwapProvidersList',
  components: {
    AppUserMsgBlock
  },
  props: {
    step: {
      type: Number,
      default: 0
    },
    availableQuotes: {
      type: Array,
      default: () => []
    },
    setProvider: {
      type: Function,
      default: () => {}
    },
    toTokenSymbol: {
      type: String,
      default: ''
    },
    toTokenIcon: {
      type: String,
      default: ''
    },
    isLoading: {
      type: Boolean,
      default: false
    },
    providersError: {
      type: Object,
      default: () => {
        return { subtitle: '' };
      }
    },
    selectedProviderId: {
      type: Number,
      default: () => undefined
    }
  },
  data() {
    return {
      showMore: false
    };
  },
  computed: {
    /**
     * @Returns Boolean
     * checks whether there's errors
     * with the providers
     */
    hasProviderError() {
      return (
        this.availableQuotes.length === 0 || this.providersError.subtitle !== ''
      );
    },
    /**
     * Property returns best rate in the quotes
     * Or null id quotes not defined.
     * Used in best rate chip
     */
    bestRate() {
      return this.providersList.length > 0 && this.providersList[0].rate
        ? this.providersList[0].rate
        : null;
    },
    /**
     * Property returns quotes to be displayed on the ui
     * If more then 3 quotes found: the list will be sliced by max_providers
     * List Fitlers out null and undefined items
     * Used in Providers Rate Row
     */
    providersList() {
      if (!this.isLoading && this.step > 0) {
        const list =
          !this.showMore && this.providersCut > 0
            ? this.availableQuotes
                .slice(0, MAX_PROVIDERS)
                .filter(item => !!item)
            : this.availableQuotes.filter(item => !!item);
        const returnedList = list.map(quote => {
          const formatted = formatFloatingPointValue(quote.rate * 100);
          const formattedAmt = formatFloatingPointValue(quote.amount);
          return {
            rate: formatted.value,
            amount: formattedAmt.value,
            tooltip: `${formattedAmt.tooltipText} ${this.toTokenSymbol}`
          };
        });
        if (returnedList) return returnedList;
      }
      return [];
    },
    /**
     * Property returns number of providers that was sliced on the ui
     */
    providersCut() {
      return this.availableQuotes.filter(item => !!item).length - MAX_PROVIDERS;
    },
    /**
     * Property returns a string for show more providers button
     * If showMore - returns "Show Less",
     * If show less - returns "{providers cut} More Providers"
     */
    moreProvidersText() {
      if (this.providersCut > 0) {
        const single = 'More Rate';
        const multiple = 'More Rates';
        if (!this.showMore) {
          return this.providersCut === 1
            ? `${this.providersCut} ${single}`
            : `${this.providersCut} ${multiple}`;
        }
        return 'Show Less';
      }
      return '';
    }
  },
  watch: {
    providersList: {
      handler: function (newVal, oldVal) {
        const hasOldVal = !oldVal || (isArray(oldVal) && oldVal.length === 0);
        if (newVal.length > 0 && hasOldVal && !this.hasProviderError) {
          const bestRate = newVal.findIndex(item => {
            return item.rate === this.bestRate;
          });

          this.$nextTick(() => {
            if (bestRate !== -1) {
              const card = this.$refs[`card${bestRate}`][0];
              if (!card?.isActive) {
                card.toggle();
              }
              this.setProvider(bestRate, this.step === 1);
            }
          });
        }
      },
      immediate: true
    },
    selectedProviderId: {
      handler: function (id) {
        setTimeout(() => {
          if (id !== undefined) {
            const card = this.$refs[`card${id}`][0];
            if (!card?.isActive) {
              card.toggle();
            }
          } else {
            setTimeout(() => {
              const refs = Object.keys(this.$refs);
              const cards = refs.filter(c => c.includes('card'));
              cards.forEach(c => {
                const card = this.$refs[c][0];
                if (card?.isActive) {
                  card.toggle();
                }
              });
            }, 500);
          }
        }, 100);
      }
    }
  }
};
</script>

<style lang="scss" scoped>
.rate-active {
  border: 1px solid var(--v-greenPrimary-base) !important;
  background-color: var(--v-greyLight-base) !important;
}
.rate {
  min-height: 60px;
  border-radius: 8px;
  border: 1px solid var(--v-greyLight-base);
}
</style>
