<template>
  <div
    class="TransactionFormVote"
  >
    <Collapse
      :is-open="!isPassphraseStep"
    >
      <ListDivided>
        <ListDividedItem :label="$t('INPUT_ADDRESS.LABEL')">
          <WalletAddress
            :address="delegate.address"
            @click="emitCancel"
          />
        </ListDividedItem>
        <ListDividedItem
          :label="$t('WALLET_DELEGATES.RANK')"
          :value="delegate.rank"
        />
        <ListDividedItem
          :label="$t('WALLET_DELEGATES.APPROVAL')"
          :value="formatter_percentage(delegate.production.approval)"
        />
        <ListDividedItem
          :label="$t('WALLET_DELEGATES.FORGED')"
          :value="forged"
        />
        <ListDividedItem
          :label="$t('WALLET_DELEGATES.BLOCKS')"
          :value="blocksProduced"
        />
        <ListDividedItem
          v-if="delegate.votes"
          :label="$t('WALLET_DELEGATES.VOTES')"
          :value="formatter_votes(delegate.votes)"
        />
        <ListDividedItem
          :label="$t('WALLET_DELEGATES.VOTERS')"
          :value="voters"
        />
      </ListDivided>

      <button
        v-show="showVoteUnvoteButton"
        type="button"
        class="blue-button mt-5"
        @click="toggleStep"
      >
        {{ isVoter ? $t('WALLET_DELEGATES.UNVOTE') : $t('WALLET_DELEGATES.VOTE') }}
      </button>
    </Collapse>

    <Collapse
      :is-open="isPassphraseStep"
    >
      <div class="mt-12">
        <InputFee
          v-if="session_network.apiVersion === 2"
          ref="fee"
          :currency="session_network.token"
          :transaction-type="$options.transactionType"
          :show-insufficient-funds="true"
          @input="onFee"
        />
      </div>

      <div
        v-if="currentWallet.isLedger"
        class="mt-10"
      >
        {{ $t('TRANSACTION.LEDGER_SIGN_NOTICE') }}
      </div>
      <InputPassword
        v-else-if="currentWallet.passphrase"
        ref="password"
        v-model="$v.form.walletPassword.$model"
        :label="$t('TRANSACTION.PASSWORD')"
        :is-required="true"
      />
      <PassphraseInput
        v-else
        ref="passphrase"
        v-model="$v.form.passphrase.$model"
        :address="currentWallet.address"
        :pub-key-hash="session_network.version"
        class="mt-5"
      />

      <PassphraseInput
        v-if="currentWallet.secondPublicKey"
        ref="secondPassphrase"
        v-model="$v.form.secondPassphrase.$model"
        :label="$t('TRANSACTION.SECOND_PASSPHRASE')"
        :pub-key-hash="session_network.version"
        class="mt-5"
      />

      <button
        :disabled="$v.form.$invalid"
        type="button"
        class="blue-button mt-5"
        @click="onSubmit"
      >
        {{ $t('COMMON.NEXT') }}
      </button>
    </Collapse>

    <ModalLoader
      ref="modalLoader"
      :message="$t('ENCRYPTION.DECRYPTING')"
      :visible="showEncryptLoader"
    />
    <ModalLoader
      :message="$t('TRANSACTION.LEDGER_SIGN_WAIT')"
      :visible="showLedgerLoader"
    />
  </div>
</template>

<script>
import { required } from 'vuelidate/lib/validators'
import { TRANSACTION_TYPES, V1 } from '@config'
import { Collapse } from '@/components/Collapse'
import { InputFee, InputPassword } from '@/components/Input'
import { ListDivided, ListDividedItem } from '@/components/ListDivided'
import { ModalLoader } from '@/components/Modal'
import { PassphraseInput } from '@/components/Passphrase'
import WalletAddress from '@/components/Wallet/WalletAddress'
import TransactionService from '@/services/transaction'

export default {
  name: 'TransactionFormVote',

  transactionType: TRANSACTION_TYPES.VOTE,

  components: {
    Collapse,
    InputFee,
    InputPassword,
    ListDivided,
    ListDividedItem,
    ModalLoader,
    PassphraseInput,
    WalletAddress
  },

  props: {
    delegate: {
      type: Object,
      required: true
    },
    isVoter: {
      type: Boolean,
      required: false,
      default: false
    },
    hasVoted: {
      type: Boolean,
      required: false,
      default: false
    }
  },

  data: () => ({
    isPassphraseStep: false,
    form: {
      fee: 0,
      passphrase: '',
      walletPassword: ''
    },
    forged: 0,
    voters: 0,
    showEncryptLoader: false,
    showLedgerLoader: false,
    bip38Worker: null
  }),

  computed: {
    currentWallet () {
      return this.wallet_fromRoute
    },

    blocksProduced () {
      const { produced, missed } = this.delegate.blocks

      if (missed > 0) {
        return `${produced} (${missed} ${this.$t('WALLET_DELEGATES.MISSED')})`
      }
      return produced || '0'
    },

    showVoteUnvoteButton () {
      if (this.currentWallet.isContact || (this.hasVoted && !this.isVoter)) {
        return false
      }

      return !this.hasVoted || (this.hasVoted && this.isVoter)
    }
  },

  watch: {
    isPassphraseStep () {
      // Ignore Ledger wallets
      if (this.currentWallet.isLedger) {
        return
      }

      // The passphrase is stored: focus on the custom password input
      if (this.currentWallet.passphrase) {
        this.$refs.password.focus()
      } else {
        this.$refs.passphrase.focus()
      }
    }
  },

  beforeDestroy () {
    this.bip38Worker.send('quit')
  },

  mounted () {
    this.fetchForged()
    this.fetchVoters()
    if (this.bip38Worker) {
      this.bip38Worker.send('quit')
    }
    this.bip38Worker = this.$bgWorker.bip38()
    this.bip38Worker.on('message', message => {
      if (message.decodedWif === null) {
        this.$error(this.$t('ENCRYPTION.FAILED_DECRYPT'))
        this.showEncryptLoader = false
      } else if (message.decodedWif) {
        this.form.passphrase = null
        this.form.wif = message.decodedWif
        this.showEncryptLoader = false
        this.submit()
      }
    })

    // Set default fees with v1 compatibility
    if (this.session_network.apiVersion === 1) {
      this.form.fee = V1.fees[this.$options.transactionType] / 1e8
    } else {
      this.form.fee = this.$refs.fee.fee
    }
  },

  methods: {
    toggleStep () {
      this.isPassphraseStep = !this.isPassphraseStep
    },

    async fetchForged () {
      const forged = await this.$client.fetchDelegateForged(this.delegate)
      this.forged = this.currency_format(this.currency_subToUnit(forged), { currencyFrom: 'network' })
    },

    async fetchVoters () {
      this.voters = await this.$client.fetchDelegateVoters(this.delegate)
    },

    onFee (fee) {
      this.$set(this.form, 'fee', fee)
    },

    onSubmit () {
      if (this.form.walletPassword && this.form.walletPassword.length) {
        this.showEncryptLoader = true
        this.bip38Worker.send({
          bip38key: this.currentWallet.passphrase,
          password: this.form.walletPassword,
          wif: this.session_network.wif
        })
      } else {
        this.submit()
      }
    },

    async submit () {
      // Ensure that fee has value, even when the user has not interacted
      if (!this.form.fee) {
        this.form.fee = this.$refs.fee.fee
      }

      const { publicKey } = this.delegate
      const prefix = this.isVoter ? '-' : '+'

      const votes = [
        `${prefix}${publicKey}`
      ]
      const transactionData = {
        passphrase: this.form.passphrase,
        votes,
        fee: parseInt(this.currency_unitToSub(this.form.fee)),
        wif: this.form.wif
      }

      if (this.currentWallet.secondPublicKey) {
        transactionData.secondPassphrase = this.form.secondPassphrase
      }

      let success = true
      let transaction
      if (!this.currentWallet.isLedger) {
        transaction = await this.$client.buildVote(transactionData, this.$refs.fee && this.$refs.fee.isAdvancedFee)
      } else {
        success = false
        this.showLedgerLoader = true
        try {
          const transactionObject = await this.$client.buildVote(transactionData, this.$refs.fee && this.$refs.fee.isAdvancedFee, true)
          transaction = await TransactionService.ledgerSign(this.currentWallet, transactionObject, this)
          success = true
        } catch (error) {
          console.log('Error : ' + error)
          this.$error(`${this.$t('TRANSACTION.LEDGER_SIGN_FAILED')}: ${error.message}`)
        }
        this.showLedgerLoader = false
      }

      if (success) {
        this.emitNext(transaction)
        this.reset()
      }
    },

    reset () {
      this.isPassphraseStep = false
      if (!this.currentWallet.passphrase && !this.currentWallet.isLedger) {
        this.$set(this.form, 'passphrase', '')
        this.$refs.passphrase.reset()
      } else if (!this.currentWallet.isLedger) {
        this.$set(this.form, 'walletPassword', '')
        this.$refs.password.reset()
      }
      this.$v.$reset()
    },

    emitCancel () {
      this.$emit('cancel')
    },

    emitNext (transaction) {
      this.$emit('next', { transaction })
    }
  },

  validations: {
    form: {
      fee: {
        required,
        isValid () {
          if (this.$refs.fee) {
            return !this.$refs.fee.$v.$invalid
          }
          return this.session_network.apiVersion === 1 // Return true if it's v1, since it has a static fee
        }
      },
      passphrase: {
        isValid (value) {
          if (this.currentWallet.isLedger || this.currentWallet.passphrase) {
            return true
          }

          if (this.$refs.passphrase) {
            return !this.$refs.passphrase.$v.$invalid
          }
          return false
        }
      },
      walletPassword: {
        isValid (value) {
          if (this.currentWallet.isLedger || !this.currentWallet.passphrase) {
            return true
          }

          if (!this.form.walletPassword || !this.form.walletPassword.length) {
            return false
          }

          if (this.$refs.password) {
            return !this.$refs.password.$v.$invalid
          }

          return false
        }
      },
      secondPassphrase: {
        isValid (value) {
          if (!this.currentWallet.secondPublicKey) {
            return true
          }

          if (this.$refs.secondPassphrase) {
            return !this.$refs.secondPassphrase.$v.$invalid
          }
          return false
        }
      }
    }
  }
}
</script>

<style scoped>
.TransactionFormVote {
  min-width: 25em;
}

.TransactionFormVote /deep/ .Collapse__handler {
  display: none
}
</style>
