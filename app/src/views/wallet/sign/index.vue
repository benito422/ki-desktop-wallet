<template>
<div id="sign-form" class="d-flex w-100 h-100 flex-column px-3" style="background-color:white">
  <form v-if="sign.file_valid" class="basic-form modal-body" style="padding-top:40px">
    <b-row>
      <b-col>
        <b-row style="margin-bottom:10px;">
          <b-col cols="10">
            <h6>{{ $t('webwallet_signing_file_label') }}</h6>
          </b-col>
          <b-col />
        </b-row>
        <b-row style="margin-bottom:15px;">
          <b-col cols="12">
            <div class="buttonInside">
              <input type="text" style="margin:0" :value="sign.file.name" disabled />
              <a class="inside" @click="removeFile()"><img src="static/img/icons/delete.png" style="width:25px; opacity:0.2" /></a>
            </div>
          </b-col>
        </b-row>
      </b-col>
      <b-col cols="6" v-if="sign.onbehalf != account.id">
        <b-row style="margin-bottom:10px;">
          <b-col cols="10">
            <h6>{{ $t('webwallet_sign_onbehalf') }} <span style="font-weight:800"> {{ sign.onbehalf_name }} </span></h6>
          </b-col>
          <b-col />
        </b-row>
        <b-row style="margin-bottom:25px;">
          <b-col cols="12">
            <input v-model="sign.onbehalf" type="text" :placeholder="$t('webwallet_for_multisig')" />
          </b-col>
        </b-row>
      </b-col>
    </b-row>


    <b-row style="margin-bottom:10px;">
      <b-col cols="10">
        <h6>{{ $t('webwallet_sign_summary') }}</h6>
      </b-col>
      <b-col />
    </b-row>
    <b-row style="margin-bottom:25px;">
      <b-col cols="12">
        <textarea v-model="sign.summary" class="warning" rows="3" disabled />
        </b-col>
    </b-row>

    <b-row style="margin-bottom:10px;">
      <b-col cols="10">
        <h6>{{ $t('enter_password') }}</h6>
      </b-col>
      <b-col />
    </b-row>
    <b-row style="margin-bottom:25px;">
      <b-col cols="10">
        <div class="buttonInside">
          <input
            v-model="wallet_pass_tmp"
            :type="password"
            :class="[wallet_pass_tmp ? '' : sign.alert]"
          />
          <a
            v-if="password == 'password'"
            class="inside"
            @click="password = 'text'"
            ><img
              src="static/img/icons/eye-on.png"
              style="width:25px; opacity:0.2"
          /></a>

          <a
            v-if="password == 'text'"
            class="inside"
            @click="password = 'password'"
            ><img
              src="static/img/icons/eye-off.png"
              style="width:25px; opacity:0.2"
          /></a>
        </div>
      </b-col>
      <b-col cols="2">
        <a v-if="sign.signature == ''" class="btn  btn-primary mt-1" @click="signTxFile">{{
          $t('signtx')
        }}</a>
        <a v-else class="btn btn-download mt-1" @click="downloadSig">{{
          $t('download')
        }}</a>
      </b-col>
    </b-row>
    <div v-if="sign.signature != ''">
      <b-row style="margin-bottom:10px;">
        <b-col cols="10">
          <h6>{{ $t('webwallet_sign_signature') }}</h6>
        </b-col>
        <b-col />
      </b-row>
      <b-row style="margin-bottom:25px;">
        <b-col cols="12">
          <textarea v-model="sign.signature" class="" rows="3" disabled />
        </b-col>
      </b-row>
    </div>



    </form>
    <form v-else>
      <!-- <div class="basic-form"> -->
        <div class="upload-form">
          <!-- <b-row align-v="center"> -->
            <b-col cols="4" />
            <b-col>
              <div
                v-cloak
                ref="myFile"
                class="upload-area"
                @drop.prevent="upload"
                @dragover.prevent
              >

                <p>
                  <img
                    src="static/img/icons/add.png"
                    style="width:100px; opacity:0.2;margin-bottom:20px"
                  />
                </p>
                <span style="opacity:0.3">{{
                  $t('webwallet_drag_drop')
                }}</span>
              </div>
            </b-col>
            <b-col cols="4" />
          <!-- </b-row> -->
        </div>
      <!-- </div> -->
    </form>
  </div>
</template>

<script>
import {
  mapActions,
  mapState
} from 'vuex';
import {
  services
} from '@services/index';
import {
  signTx,
  createBroadcastTx
} from '@tendermint/sig';
import util from '@static/js/util';
import {
  BRow,
  BCol,
  BButton,
} from 'bootstrap-vue';


import { MsgSend } from "cosmjs-types/cosmos/bank/v1beta1/tx";
import { TxRaw } from "cosmjs-types/cosmos/tx/v1beta1/tx";
import { Secp256k1Wallet, MultisigThresholdPubkey, StdFee, encodeSecp256k1Pubkey, createMultisigThresholdPubkey, pubkeyToAddress } from "@cosmjs/amino";
import { makeMultisignedTx, MsgSendEncodeObject, SignerData, SigningStargateClient, StargateClient, coins } from '@cosmjs/stargate'



export default {
  data() {
    return {
      denom: this.globalData.kichain.denom,
      password: 'password',
      wallet_pass_tmp: '',
      isLoading: true,
      sign: {
        alert: '',
        file: '',
        file_valid: false,
        file_content: '',
        summary: '',
        signature: '',
        onbehalf: '',
        onbehalf_name: '',
      },
    }
  },

  computed: {
    ...mapState({
      transactions: state => state.wallets.current.transactions,
      key: state => state.wallets.current.privatekey,
      account: state => state.account,
      wallet: state => state.wallets.current,
      chainId: state => state.app.chainId,
      selfWallets: state => state.wallets.list,
    }),
  },
  methods: {
    upload(e) {
      let file = e.dataTransfer.files[0];
      this.sign.file = file;
      if (!file) return;

      let reader = new FileReader();
      reader.readAsText(file, 'UTF-8');
      reader.onload = evt => {
        this.sign.file_content = evt.target.result;
        this.sign.file_valid = true;
        this.sign.summary = this.parseMessage(this.sign.file_content);
      };
      reader.onerror = evt => {
        console.error(evt);
      };
    },
    parseMessage(file) {
      try {
        let msg_ = JSON.parse(file).value.msg;

        switch (msg_[0].type) {
          case 'cosmos-sdk/MsgSend':
            var msg = msg_[0];
            this.sign.onbehalf = msg.value.from_address;
            this.sign.onbehalf_name = this.findWalletName(this.sign.onbehalf)
            return (
              'Send:\t ' +
              msg.value.amount[0].amount / Math.pow(10, 6) +
              this.denom + '\nfrom:\t ' +
              msg.value.from_address + '\t\t(' + this.findWalletName(msg.value.from_address) + ')' +
              ' \nto:\t\t ' +
              msg.value.to_address + '\t\t\t(' + this.findWalletName(msg.value.to_address) + ')'
            );
            break;

          case 'cosmos-sdk/MsgDelegate':
            var msg = msg_[0];
            this.sign.onbehalf = msg.value.delegator_address;
            this.sign.onbehalf_name = this.findWalletName(this.sign.onbehalf)
            return (
              'Delegate:\t ' +
              msg.value.amount.amount / Math.pow(10, 6) +
              this.denom + '\nfrom:\t\t ' +
              msg.value.delegator_address + '\t\t(' + this.findWalletName(msg.value.delegator_address) + ')' +
              '\nto:\t\t\t ' +
              msg.value.validator_address
            );
            break;

          case 'cosmos-sdk/MsgUndelegate':
            var msg = msg_[0];
            this.sign.onbehalf = msg.value.delegator_address;
            this.sign.onbehalf_name = this.findWalletName(this.sign.onbehalf)
            return (
              'Unbond:\t ' +
              msg.value.amount.amount / Math.pow(10, 6) +
              this.denom + '\nfrom:\t ' +
              msg.value.validator_address
            );
            break;

          case 'cosmos-sdk/MsgBeginRedelegate':
            var msg = msg_[0];
            this.sign.onbehalf = msg.value.delegator_address;
            this.sign.onbehalf_name = this.findWalletName(this.sign.onbehalf)
            return (
              'Redelagate:\t ' +
              msg.value.amount.amount / Math.pow(10, 6) +
              this.denom + '\nfrom:\t\t ' +
              msg.value.validator_src_address +
              ' \nto:\t\t\t ' +
              msg.value.validator_dst_address
            );
            break;

          case 'cosmos-sdk/MsgWithdrawDelegationReward':
            var msg = msg_[0];
            this.sign.onbehalf = msg.value.delegator_address;
            this.sign.onbehalf_name = this.findWalletName(this.sign.onbehalf)
            var output = 'Withdraw rewards ';
            if (!(msg_[1] === undefined)) {
              if (msg_[1].type == 'cosmos-sdk/MsgWithdrawValidatorCommission') {
                output = 'Withdraw rewards and commissions ';
              }
            }
            output = output + 'from ' + msg.value.validator_address;
            return output;
            break;

          case 'cosmos-sdk/MsgWithdrawValidatorCommission':
            var msg = msg_[0];
            this.sign.onbehalf = this.account.id;
            this.sign.onbehalf_name = this.findWalletName(this.sign.onbehalf)
            var output = 'Withdraw commissions';
            if (!(msg_[1] === undefined)) {
              if (msg_[1].type == 'cosmos-sdk/MsgWithdrawValidatorCommission') {
                output = 'Withdraw rewards and commissions ';
              }
            }
            output = output + 'from ' + msg.value.validator_address;
            return output;
            break;

          default:
            return 'The file does not seem to contain a valid transaction structure.';
        }
      } catch (error) {
        return 'The file does not seem to contain a valid transaction structure.';
      }
    },
    downloadSig() {
      return util.download("signed_" + this.account.name + "_" + this.sign.file.name.replace(".json", "")  + ".json", document, this.sign.signature);
    },


    async singleSign(signingInstruction, key) {
      const wallet = await Secp256k1Wallet.fromKey(key, "tki");
      const pubkey = encodeSecp256k1Pubkey((await wallet.getAccounts())[0].pubkey);
      const address = (await wallet.getAccounts())[0].address;

      const signingClient = await SigningStargateClient.offline(wallet);

      const signerData = {
        accountNumber: signingInstruction.accountNumber,
        sequence: signingInstruction.sequence,
        chainId: signingInstruction.chainId,
      };


      try{
        const { bodyBytes: bb, signatures } = await signingClient.sign(
          address,
          signingInstruction.msgs,
          signingInstruction.fee,
          signingInstruction.memo,
          signerData,
        );

        return {"address": address, "signature": signatures[0], "transaction": bb, "signingInstruction": signingInstruction};
      }
      catch (err){
        console.log(err);
      }
    },

    async signTxFile() {
      let transaction = JSON.parse(this.sign.file_content).value;

      let account =
        this.sign.onbehalf == '' ? this.account.id : this.sign.onbehalf;

      if (transaction.hasOwnProperty('signatures')) {
        delete transaction['signatures'];
      }

      const response = await services.auth.fetchAccount(account);

      let sequence_ = 0;
      let account_number_ = '';

      if (response.data.result.value) {
        let res = '';
        if (response.data.result.type == 'cosmos-sdk/ContinuousVestingAccount' || response.data.result.type == 'cosmos-sdk/DelayedVestingAccount') {
          res = response.data.result.value.base_vesting_account.base_account;
        } else {
          res = response.data.result.value;
        }
        sequence_ = res.sequence;
        account_number_ = res.account_number;
      }


      var msg = {
        typeUrl: '/cosmos.bank.v1beta1.MsgSend',
        value: {
          fromAddress: transaction.msg[0].value.from_address,
          toAddress: transaction.msg[0].value.to_address,
          amount: transaction.msg[0].value.amount
        }
      }

      const signingInstruction = {
          accountNumber: parseInt(account_number_),
          sequence: parseInt(sequence_),
          chainId: this.chainId,
          msgs: [msg],
          fee: transaction.fee,
          memo: transaction.memo,
        };

      try {

        var CryptoJS = require('crypto-js');
        var bytes = CryptoJS.AES.decrypt(this.key, this.wallet_pass_tmp);
        let key = Buffer.from(bytes.toString(CryptoJS.enc.Utf8), 'hex');
        let signedTransactionme = await this.singleSign(signingInstruction, key)
        this.sign.signature = JSON.stringify(signedTransactionme);
        this.wallet_pass_tmp = ''

      } catch (error) {
        let humanizedError;

        if (
          RegExp(
            `^RangeError: private key length is invalid|Malformed UTF-8 data`,
          ).test(error)
        ) {
          humanizedError = 'Wrong Password';
        }
        this.$bvToast.toast("Wrong Password", {
          title: `Transaction failed`,
          variant: 'danger',
          autoHideDelay: 2000,
          solid: true,
          toaster: 'b-toaster-bottom-center',
        });
      }
    },
    findWalletName(address) {
      var wallet_ = this.selfWallets.filter(w => {
        if (w.address == address) {
          return w
        }
      });
      if (wallet_.length > 0) {
        return wallet_[0].account
      } else {
        return ''
      }
    },

    removeFile() {
      this.sign.file = '';
      this.sign.summary = '';
      this.sign.onbehalf = '';
      this.sign.signature = '';
      this.sign.file_valid = false;
      this.sign.file_content = '';
      this.sign.onbehalf_name = '';
      this.wallet_pass_tmp = ''
    },
  }
};
</script>

<style scoped></style>
