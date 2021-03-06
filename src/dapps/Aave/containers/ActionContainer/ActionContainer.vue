<template>
  <div>
    <div class="balance-wrapper">
      <div class="balance-container">
        <p class="title">
          {{
            activeDepositTab
              ? $t('dappsAave.aave-depost-bal')
              : $t('dappsAave.you-borrowed')
          }}
        </p>
        <p class="token-balance">
          {{
            activeDepositTab
              ? convertToFixed(userToken.principalATokenBalance)
              : convertToFixed(userToken.currentBorrows)
          }}
          <span class="token-name"> {{ token.symbol }} </span>
        </p>
        <p class="usd-amt">
          ${{
            activeDepositTab
              ? convertToFixed(userToken.currentUnderlyingBalanceUSD)
              : convertToFixed(userToken.currentBorrowsUSD)
          }}
        </p>
      </div>
      <div class="balance-container">
        <p class="title">
          {{
            activeDepositTab
              ? $t('dappsAave.aave-wallet-bal')
              : $t('dappsAave.total-collateral')
          }}
        </p>
        <p class="token-balance">
          {{
            activeDepositTab
              ? convertToFixed(token.tokenBalance)
              : convertToFixed(userSummary.totalCollateralETH, 6)
          }}
          <span class="token-name">{{
            activeDepositTab ? token.symbol : $t('common.currency.eth')
          }}</span>
        </p>
        <p class="usd-amt">
          ${{
            activeDepositTab
              ? getUSDBalance(token)
              : convertToFixed(userSummary.totalCollateralUSD)
          }}
        </p>
      </div>
    </div>
    <div class="action-container mt-5">
      <i18n class="action-question" tag="p" path="dappsAave.how-much-action">
        <span slot="action" class="action-title">{{ actionTitle }}</span>
      </i18n>
      <i18n class="action-info" tag="p" path="dappsAave.action-info">
        <span slot="action" class="action-title">{{ actionTitle }}</span>
      </i18n>
      <input v-model="amount" type="number" placeholder="0" />
      <div class="percentage-container">
        <div
          :class="percentBtns.twentyFivePercentEnabled ? 'active' : ''"
          @click="setPercentAmount('twentyFivePercentEnabled', 0.25)"
        >
          25%
        </div>
        <div
          :class="percentBtns.fiftyPercentEnabled ? 'active' : ''"
          @click="setPercentAmount('fiftyPercentEnabled', 0.5)"
        >
          50%
        </div>
        <div
          :class="percentBtns.seventyFivePercentEnabled ? 'active' : ''"
          @click="setPercentAmount('seventyFivePercentEnabled', 0.75)"
        >
          75%
        </div>
        <div
          :class="percentBtns.maxEnabled ? 'active' : ''"
          @click="setPercentAmount('maxEnabled', 1)"
        >
          {{ $t('dappsAave.max') }}
        </div>
      </div>
      <p v-if="errorMsg" class="error">{{ errorMsg }}</p>
      <button
        :class="[
          'take-action-btn',
          errorMsg || !amount ? 'disabled' : 'enabled'
        ]"
        @click="takeAction()"
      >
        {{ actionTitle }}
      </button>
      <button class="cancel-btn" @click="goToHome()">
        {{ $t('common.cancel') }}
      </button>
    </div>
    <rate-modal
      ref="rateModal"
      :action-title="actionTitle"
      :amount="amount"
      :token="token"
      :user-summary="userSummary"
      @emitTakeAction="emitTakeAction"
    />
    <confirmation-modal
      ref="confirmationModal"
      :action-title="actionTitle"
      :user-summary="userSummary"
      :active-deposit-tab="activeDepositTab"
      :amount="amount"
      :token="token"
      @emitTakeAction="emitTakeAction"
    />
  </div>
</template>

<script>
import BigNumber from 'bignumber.js';
import { Toast } from '@/helpers';
import { mapState, mapActions } from 'vuex';
import RateModal from '@/dapps/Aave/components/RateModal';
import ConfirmationModal from '@/dapps/Aave/components/ConfirmationModal';

export default {
  components: {
    'rate-modal': RateModal,
    'confirmation-modal': ConfirmationModal
  },
  props: {
    userSummary: {
      type: Object,
      default: () => {
        return {};
      }
    },
    activeBorrowTab: {
      type: Boolean,
      default: false
    },
    activeDepositTab: {
      type: Boolean,
      default: true
    },
    reserves: {
      type: Array,
      default: function () {
        return [];
      }
    }
  },
  data() {
    return {
      errorMsg: '',
      amount: null,
      ethPrice: 0,
      disableBtn: false,
      userToken: {},
      token: { user: {}, price: {} },
      actionType: null,
      percentBtns: {
        twentyFivePercentEnabled: false,
        fiftyPercentEnabled: false,
        seventyFivePercentEnabled: false,
        maxEnabled: false
      },
      actionTitles: {
        deposit: 'Deposit',
        withdraw: 'Withdraw',
        repay: 'Repay',
        borrow: 'Borrow'
      }
    };
  },
  computed: {
    ...mapState('main', ['online']),
    actionTitle() {
      if (!this.actionType) {
        return this.activeDepositTab
          ? this.$tc('dappsAave.deposit', 1)
          : this.$t('dappsAave.borrow');
      }
      return this.actionType === 'Withdraw'
        ? this.$t('dappsAave.withdraw')
        : this.$t('dappsAave.repay');
    },
    amountToCheck() {
      if (this.actionTitle === this.actionTitles.deposit) {
        return this.token.tokenBalance;
      } else if (this.actionTitle === this.actionTitles.borrow) {
        return new BigNumber(this.userSummary.availableBorrowsETH).div(
          this.token.price.priceInEth
        );
      } else if (this.actionTitle === this.actionTitles.repay) {
        return this.userToken.currentBorrows;
      } else if (this.actionTitle === this.actionTitles.withdraw) {
        return this.userToken.principalATokenBalance;
      }
      return false;
    }
  },
  watch: {
    amount() {
      if (this.checkAmount(this.amountToCheck)) {
        this.errorMsg = this.$t('dappsAave.cannot-exceed-bal', {
          amount: this.convertToFixed(this.amountToCheck)
        });
      } else if (
        this.amount === this.userToken.currentBorrows &&
        this.amount > this.token.tokenBalance
      ) {
        this.errorMsg = this.$t('dappsAave.cannot-exceed-tkn-bal', {
          amount: this.convertToFixed(this.token.tokenBalance)
        });
      } else if (this.amount <= 0) {
        this.errorMsg = this.$t('dappsAave.greater-than');
      } else {
        this.errorMsg = '';
      }
    }
  },
  mounted() {
    this.token = this.$route.params.token || {};
    this.actionType = this.$route.params.actionType || null;
    this.callSetToken(this.token);

    if (this.online) {
      this.getEthPrice();
    }

    this.userToken = this.token.user ? this.token.user : {};
  },
  methods: {
    ...mapActions('aave', ['setToken']),
    checkAmount(total) {
      if (new BigNumber(this.amount).gt(new BigNumber(total))) {
        return true;
      }
    },
    callSetToken(token) {
      this.setToken(token);
    },
    findReserve(id, reserves) {
      return reserves.find(reserve => {
        return reserve.underlyingAsset
          ? reserve.underlyingAsset === id
          : reserve.reserve.underlyingAsset === id;
      });
    },
    convertToFixed(val, num) {
      if (!val || val == 0) {
        return 0;
      }

      if (!num) {
        num = 2;
      }

      return new BigNumber(val).toFixed(num).toString();
    },
    takeAction() {
      this.actionTitle === this.actionTitles.borrow
        ? this.$refs.rateModal.$refs.rateModal.show()
        : this.$refs.confirmationModal.$refs.confirmationModal.show();
    },
    emitTakeAction(param) {
      this.goToHome();
      this.$emit('emitTakeAction', param);
    },
    setPercentAmount(selectedBtn, percentage) {
      this.amount = new BigNumber(this.amountToCheck)
        .times(percentage)
        .toFixed();
      Object.keys(this.percentBtns).forEach(btn => {
        if (selectedBtn === btn) {
          if (this.percentBtns[btn] === true) {
            this.percentBtns[btn] = false;
            this.amount = null;
          } else {
            this.percentBtns[btn] = true;
          }
        } else {
          this.percentBtns[btn] = false;
        }
      });
    },
    goToHome() {
      this.$router.push('/interface/dapps/aave');
    },
    getUSDBalance(token) {
      let usdBalance = 0;

      if (token && token.price) {
        const ethBalance = new BigNumber(
          new BigNumber(token.tokenBalance).times(
            new BigNumber(token.price.priceInEth)
          )
        );

        if (ethBalance && this.ethPrice) {
          usdBalance = new BigNumber(
            new BigNumber(ethBalance).times(new BigNumber(this.ethPrice))
          )
            .toFixed(2)
            .toString();
        }
      }
      return usdBalance;
    },
    async getEthPrice() {
      const price = await fetch(
        'https://cryptorates.mewapi.io/ticker?filter=ETH'
      )
        .then(res => {
          return res.json();
        })
        .catch(e => {
          Toast.responseHandler(e, Toast.ERROR);
        });
      this.ethPrice =
        typeof price === 'object' ? price.data.ETH.quotes.USD.price : 0;
    }
  }
};
</script>

<style lang="scss" scoped>
@import 'ActionContainer.scss';
</style>
