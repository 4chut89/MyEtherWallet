<template>
  <div class="wallet-promo-snackbar">
    <v-snackbar
      :value="show"
      vertical
      :width="width"
      :min-width="width"
      :timeout="-1"
      right
      :class="[
        notSmallOrXs ? 'pr-8' : 'pr-6',
        'mr-16 snackbar-container',
        'snackbar-border'
      ]"
      transition="scale-transition"
      content-class="mew-survey-content"
    >
      <div class="d-flex align-center mb-2">
        <img
          class="eth--icon mr-4"
          src="@/dapps/stakewise/assets/icon-stakewise-purple.svg"
          width="32px"
          height="32px"
        />
        <div class="mew-heading-3 textDark--text">
          You can now earn rewards with any amount of ETH
        </div>
      </div>
      <div
        class="mew-component--module-side-apr backgroundGrey pa-5 border-radius--10px"
      >
        <div class="d-flex align-center justify-space-between mb-1">
          <div
            class="textLight--text text-uppercase mew-label font-weight-medium"
          >
            Current APR
          </div>
          <div class="textLight--text greenPrimary--text">
            {{ validatorApr }}%
          </div>
        </div>
        <div class="d-flex align-center justify-space-between">
          <div
            class="textLight--text text-uppercase mew-label font-weight-medium"
          >
            ETH in pool
          </div>
          <div class="textDark--text text-right">
            {{ formattedPoolSupply }} ETH
          </div>
        </div>
      </div>
      <div class="my-2">
        <mew-button
          has-full-width
          title="Start Staking"
          btn-style="background"
          btn-size="large"
          @click.native="openStaking"
        />
      </div>
      <div>
        <mew-button
          has-full-width
          title="Close"
          btn-style="transparent"
          btn-size="large"
          @click.native="close"
        />
      </div>
    </v-snackbar>
  </div>
</template>

<script>
import { STAKEWISE_ROUTES } from '@/dapps/stakewise/configsRoutes';
import { mapActions, mapGetters, mapState } from 'vuex';
import moment from 'moment';
import { formatFloatingPointValue } from '@/core/helpers/numberFormatHelper';
export default {
  data() {
    return {
      timeoutHolder: null,
      show: false
    };
  },
  computed: {
    ...mapState('stakewise', [
      'closedInitialPromo',
      'showForSeven',
      'showForFourteen',
      'validatorApr'
    ]),
    ...mapGetters('stakewise', ['getPoolSupply']),
    ...mapGetters('global', ['network']),
    ...mapState('wallet', ['address', 'web3']),
    ...mapGetters('global', ['isEthNetwork', 'network']),
    formattedPoolSupply() {
      return formatFloatingPointValue(this.getPoolSupply).value;
    },
    shouldShow() {
      if (this.diff === 7 && this.showForSeven) {
        return true;
      }
      if (this.diff === 14 && this.showForFourteen) {
        return true;
      }

      return !this.showInitialPromo;
    },
    notSmallOrXs() {
      return (
        this.$vuetify.breakpoint.lg ||
        this.$vuetify.breakpoint.xl ||
        this.$vuetify.breakpoint.md
      );
    },
    width() {
      if (this.notSmallOrXs) {
        return '320px';
      } else if (this.$vuetify.breakpoint.sm) {
        return '250px';
      }
      return '225px';
    },
    diff() {
      if (this.closedInitialPromo !== '') {
        const diff = moment(new Date()).diff(
          moment(this.closedInitialPromo),
          'days'
        );
        return diff;
      }
      return 0;
    }
  },
  watch: {
    closedInitialPromo(newVal) {
      if (newVal) {
        this.checkAndOpen();
      }
    }
  },
  mounted() {
    this.checkAndOpen();
  },
  methods: {
    ...mapActions('stakewise', ['hideForSeven', 'hideForFourteen']),
    checkAndOpen() {
      if (this.closedInitialPromo) {
        if (this.diff === 7 || this.diff === 14) {
          this.delayOpenSnackBar();
        }
      }
    },
    setCloseState() {
      if (this.diff === 7) {
        this.hideForSeven();
      }
      if (this.diff === 14) {
        this.hideForFourteen();
      }
    },
    close() {
      this.setCloseState();
      this.show = false;
    },
    openStaking() {
      this.$router.push({ name: STAKEWISE_ROUTES.CORE.NAME });
      this.close();
    },
    delayOpenSnackBar() {
      this.timeoutHolder = setTimeout(() => {
        this.show = this.shouldShow;
      }, 3000);
    }
  }
};
</script>

<style lang="scss">
.wallet-promo-snackbar {
  .eth--icon {
    border: 1px solid var(--v-greyMedium-base);
    border-radius: 8px;
  }
  .v-snack__action {
    margin: 0 !important;
  }
}
</style>
