<template>
  <div>
    <mew-address-select
      ref="addressSelect"
      :resolved-addr="resolvedAddr"
      :copy-tooltip="$t('common.copy')"
      :save-tooltip="$t('common.save')"
      :enable-save-address="enableSave"
      :label="addrLabel"
      :items="addressBookWithMyAddress"
      :placeholder="$t('sendTx.enter-addr')"
      :success-toast="$t('sendTx.success.title')"
      :is-valid-address="isValidAddress"
      :show-copy="isValidAddress"
      :error-messages="errorMessages"
      @saveAddress="toggleOverlay"
      @input="setAddress"
    />
    <!-- add and edit the address book -->
    <mew-overlay
      :footer="{
        text: 'Need help?',
        linkTitle: 'Contact support',
        link: 'mailto:support@myetherwallet.com'
      }"
      :title="$t('interface.address-book.add-addr')"
      :show-overlay="addMode"
      :close="toggleOverlay"
      content-size="xlarge"
    >
      <address-book-add-edit
        :to-address="inputAddr"
        mode="add"
        @back="toggleOverlay"
      />
    </mew-overlay>
  </div>
</template>

<script>
import { isAddress } from '@/core/helpers/addressUtils';
import { mapGetters, mapState } from 'vuex';
import NameResolver from '@/modules/name-resolver/index';
import AddressBookAddEdit from './components/AddressBookAddEdit';
import { isObject, throttle } from 'lodash';
import { toChecksumAddress } from '@/core/helpers/addressUtils';
import WAValidator from 'multicoin-address-validator';

const USER_INPUT_TYPES = {
  typed: 'TYPED',
  selected: 'SELECTED',
  resolved: 'RESOLVED'
};

export default {
  components: {
    AddressBookAddEdit
  },
  props: {
    isValidAddressFunc: {
      type: Function,
      default: isAddress
    },
    isHomePage: {
      type: Boolean,
      default: false
    },
    label: {
      type: String,
      default: ''
    },
    currency: {
      type: String,
      default: 'ETH'
    },
    preselectCurrWalletAdr: {
      type: Boolean,
      default: false
    },
    enableSaveAddress: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      addMode: false,
      resolvedAddr: '',
      inputAddr: '',
      nameResolver: null,
      isValidAddress: false,
      loadedAddressValidation: false
    };
  },

  computed: {
    ...mapState('addressBook', ['addressBookStore']),
    ...mapGetters('global', ['network']),
    ...mapState('wallet', ['web3', 'address']),
    errorMessages() {
      if (!this.isValidAddress && this.loadedAddressValidation) {
        return this.$t('interface.address-book.validations.invalid-address');
      }
      if (!this.inputAddr && this.loadedAddressValidation) {
        return this.$t('interface.address-book.validations.addr-required');
      }
      return '';
    },
    addressBookWithMyAddress() {
      return this.isHomePage
        ? [
            {
              address: '0xDECAF9CD2367cdbb726E904cD6397eDFcAe6068D',
              currency: 'ETH',
              nickname: 'MEW Donations',
              resolverAddr: '0xDECAF9CD2367cdbb726E904cD6397eDFcAe6068D'
            }
          ]
        : this.address
        ? [
            {
              address: toChecksumAddress(this.address),
              nickname: 'My Address',
              resolverAddr: ''
            }
          ].concat(this.addressBookStore)
        : [
            {
              address: toChecksumAddress(this.address),
              nickname: 'My Address',
              resolverAddr: ''
            }
          ];
    },
    enableSave() {
      return this.isHomePage
        ? false
        : this.isValidAddress && this.enableSaveAddress;
    },
    addrLabel() {
      return this.label === '' ? this.$t('sendTx.to-addr') : this.label;
    }
  },
  watch: {
    web3() {
      if (this.network.type.ens && this.web3.currentProvider) {
        this.nameResolver = new NameResolver(this.network, this.web3);
      } else {
        this.nameResolver = null;
      }
    }
  },
  mounted() {
    if (this.network.type.ens && this.web3.currentProvider)
      this.nameResolver = new NameResolver(this.network, this.web3);
    if (this.isHomePage) {
      this.setDonationAddress();
    }
    if (this.preselectCurrWalletAdr) {
      this.$refs.addressSelect.selectAddress(this.addressBookWithMyAddress[0]);
      this.setAddress(
        toChecksumAddress(this.address),
        USER_INPUT_TYPES.selected
      );
    }
  },
  methods: {
    /**
     * Checks if address is valid
     * and sets the address value
     */
    async setAddress(value, inputType) {
      if (typeof value === 'string') {
        if (
          this.currency.toLowerCase() ===
          this.network.type.currencyName.toLowerCase()
        ) {
          /**
           * Checks if user typed or selected an address from dropdown
           */
          const typeVal =
            inputType === USER_INPUT_TYPES.typed
              ? value
              : this.addressBookWithMyAddress.find(item => {
                  return value.toLowerCase() === item.address.toLowerCase();
                });
          this.inputAddr = value;
          this.resolvedAddr = '';
          /**
           * Checks if the address is valid
           */
          const isAddValid = this.isValidAddressFunc(this.inputAddr);
          if (isAddValid instanceof Promise) {
            const validation = await isAddValid;
            this.isValidAddress = validation;
          } else {
            this.isValidAddress = isAddValid;
          }
          this.loadedAddressValidation = !this.isValidAddress ? false : true;

          /**
           * @emits setAddress
           */
          this.$emit('setAddress', value, this.isValidAddress, {
            type: inputType,
            value: isObject(typeVal) ? typeVal.nickname : typeVal
          });
          if (!this.isValidAddress) {
            this.resolveName();
          }
        } else {
          const currencyExists = WAValidator.findCurrency(
            this.currency.toLowerCase()
          );
          if (currencyExists) {
            const validate = WAValidator.validate(
              value,
              this.currency.toLowerCase()
            );
            if (validate) {
              this.inputAddr = value;
              this.isValidAddress = true;
            } else {
              this.isValidAddress = false;
            }
            this.loadedAddressValidation = true;
            /**
             * @emits setAddress
             */
            this.$emit('setAddress', value, this.isValidAddress, {
              type: inputType,
              value: value
            });
          } else {
            this.isValidAddress = false;
            this.loadedAddressValidation = true;
            this.$emit('setAddress', value, this.isValidAddress, {
              type: inputType,
              value: value
            });
          }
        }
      }
    },
    // is used from the parent context
    // eslint-disable-next-line
    clear() {
      this.addMode = false;
      this.resolvedAddr = '';
      this.inputAddr = '';
      this.nameResolver = null;
      this.isValidAddress = false;
      this.loadedAddressValidation = false;
      this.$refs.addressSelect.clear();
      this.$emit('setAddress', this.resolvedAddr, this.isValidAddress, {
        type: USER_INPUT_TYPES.typed,
        value: this.inputAddr
      });

      // Calls setups from mounted
      if (this.network.type.ens)
        this.nameResolver = new NameResolver(this.network, this.web3);
      if (this.isHomePage) {
        this.setDonationAddress();
      }
    },
    /**
     * Sets selected address to be MEW donation address
     * only happens on home page
     */
    setDonationAddress() {
      this.$refs.addressSelect.selectAddress(this.addressBookWithMyAddress[0]);
    },
    toggleOverlay() {
      this.addMode = !this.addMode;
    },
    /**
     * Resolves name and @returns address
     */
    resolveName: throttle(async function () {
      if (this.nameResolver) {
        try {
          await this.nameResolver.resolveName(this.inputAddr).then(addr => {
            this.resolvedAddr = addr;
            this.isValidAddress = true;
            this.loadedAddressValidation = true;
            this.$emit('setAddress', this.resolvedAddr, this.isValidAddress, {
              type: USER_INPUT_TYPES.resolved,
              value: this.inputAddr
            });
          });
        } catch (e) {
          this.loadedAddressValidation = true;
        }
      }
    }, 500)
  }
};
</script>
