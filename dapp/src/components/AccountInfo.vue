<template>
  <v-content>
    <v-dialog v-model="dialog" persistent="true">
      <v-card>
        <v-card-text>
          <v-text-field
            ref="fromAddress"
            v-model="address"
            label="转账地址"
            clear-icon="mdi-close-circle"
            disabled
            box
          ></v-text-field>
          <v-text-field
            ref="amount"
            v-model="amount"
            :rules="[
                () => !!amount || 'This field is required']"
            label="数量"
            type="number"
            :hint="'最大:' + tokenBalance + ' ABST'"
            suffix="ABST"
            clear-icon="mdi-close-circle"
            required
            clearable
            box
          ></v-text-field>
          <v-text-field
            ref="toAddress"
            v-model="toAddress"
            prepend-inner-icon="mdi-qrcode-scan"
            @click:prepend-inner="scanFortoAddr()"
            :rules="[() => !!toAddress || 'This field is required',
              ()=>isAddress(toAddress) || 'addrress is invalid']"
            label="接收地址"
            clear-icon="mdi-close-circle"
            required
            clearable
            box
          ></v-text-field>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue darken-1" flat @click="dialog = false">取消</v-btn>
          <v-btn color="blue darken-1" flat @click="transfer()" :disabled="!isTransfer()">确认</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-dialog v-model="isQRCode">
      <v-card>
        <v-img :src="qrcodeUrl"/>
        <v-card-text class="text-xs-center">
          <div>
            <span class="caption font-weight-thin">{{address}}</span>
            <br>
            <v-btn
              flat
              multi-line
              v-clipboard:copy="address"
              v-clipboard:success="onCopy"
              v-clipboard:error="onError"
            >复制钱包地址</v-btn>
          </div>
        </v-card-text>
      </v-card>
    </v-dialog>
    <v-layout row wrap>
      <v-flex xs2>
        <v-snackbar v-model="snackbar" :multi-line="true" :top="true" :color="color">
          <div>{{message}}</div>
          <v-icon size="16" @click="snackbar = false">mdi-close-circle</v-icon>
        </v-snackbar>
      </v-flex>
    </v-layout>
    <div>
      <v-card-text primary-title class="text-xs-center">
        <div>
          <v-btn flat @click="createQRCode()">
            <v-card-text class="text-xs-center">
              <div style="margin-top: -20px;">
                <span :class="[isAdmin ? 'orange--text subheading':'subheading']">{{name}}</span>
                <br>
                <span class="caption font-weight-thin">{{subAddress}}</span>
              </div>
            </v-card-text>
          </v-btn>
        </div>
      </v-card-text>
      <v-avatar slot="offset" class="mx-auto d-block" size="130">
        <img :src="salonImg">
      </v-avatar>
    </div>
    <v-card-text class="text-xs-center">
      <div>
        <strong class="subheading">{{tokenBalance}}&#32;ABST&#32;/&#32;</strong>
        <strong class="subheading">{{balance}}&#32;{{isMoac}}</strong>
        <br>
        <span class="caption font-weight-thin">(totalSupply:&#32;{{totalSupply}}&#32;ABST)</span>
      </div>
    </v-card-text>
    <v-card-text class="text-xs-center">
      <v-btn color="success" round class="font-weight-light" @click="dialog = true">发送</v-btn>
      <v-btn
        color="success"
        v-show="isAdmin"
        round
        class="font-weight-light"
        @click="toNewCampaign()"
      >创建</v-btn>
    </v-card-text>
  </v-content>
</template>

<script>
import SalonToken from "../js/SalonToken";
import Salon from "../js/Salon";
import tp from "tp-js-sdk";
import QRCode from "qrcode";

export default {
  props: { isBalance: Boolean },
  data() {
    return {
      address: "",
      name: "",
      subAddress: "",
      toAddress: "",
      amount: "",
      tokenBalance: 0,
      balance: 0,
      totalSupply: 0,
      dialog: false,
      isAdmin: false,
      snackbar: false,
      isQRCode: false,
      qrcodeUrl: "",
      message: "",
      color: "",
      isMoac: "",
      salonImg: require("../assets/salon.png")
    };
  },
  beforeCreate: async function() {
    try {
      this.isAdmin = await Salon.init();
      await SalonToken.init();
      this.address = Salon.fromAddress;
      this.name = Salon.fromName;
      if (process.env.VUE_APP_NETWORK === "MOAC") {
        this.isMoac = "MOAC";
      } else {
        this.isMoac = "ETH";
      }
      this.splitAddr();
    } catch (e) {
      console.log(e);
    }
  },
  mounted: async function() {
    try {
      await SalonToken.init();
      this.getBalance();
      this.getTokenBalance();
      this.getTotalSupply();
    } catch (e) {
      console.log(e);
    }
  },
  methods: {
    getTotalSupply: async function() {
      this.totalSupply = await SalonToken.totalSupply();
    },
    getTokenBalance: async function() {
      this.tokenBalance = this.roundFun(await SalonToken.getTokenBalance(), 4);
    },
    getBalance: async function() {
      this.balance = this.roundFun(await SalonToken.getBalance(), 4);
    },
    transfer: async function() {
      let res = await SalonToken.transfer(this.toAddress, this.amount).catch(
        e => {
          console.log(e);
        }
      );
      if (res) {
        this.message = "转账成功!";
        this.color = "success";
        this.snackbar = true;
        this.getBalance();
        this.dialog = false;
      } else {
        this.color = "error";
        this.message = "转账失败!";
        this.snackbar = true;
      }
    },
    scanFortoAddr: async function() {
      let address = await tp.invokeQRScanner();
      if (!this.isAddress(address)) {
        if (process.env.VUE_APP_NETWORK === "MOAC") {
          this.message = "请使用MOAC钱包";
        } else {
          this.message = "请使用ETH钱包";
        }
        this.snackbar = true;
        return;
      }
      this.toAddress = address;
    },
    isAddress: function(address) {
      return SalonToken.isAddress(address);
    },
    isTransfer: function() {
      return this.isAddress(this.toAddress) && this.amount;
    },
    toNewCampaign: function() {
      this.$emit("toNewCampaign");
    },
    isAdministrator: async function() {
      this.isAdmin = await Salon.isAdministrator().catch(e => {
        console.log(e);
      });
    },
    roundFun: function(value, n) {
      return Math.round(value * Math.pow(10, n)) / Math.pow(10, n);
    },
    splitAddr: function() {
      let length = this.address.length;
      this.subAddress =
        this.address.slice(0, 7) + "..." + this.address.slice(length - 4);
    },
    onCopy: function() {
      this.message = "复制成功";
      this.color = "success";
      this.snackbar = true;
    },
    onError: function() {
      this.message = "复制失败";
      this.color = "error";
      this.snackbar = true;
    },
    createQRCode: async function() {
      let opts = {
        errorCorrectionLevel: "H",
        type: "image/png",
        rendererOpts: {
          quality: 0.3
        }
      };
      this.qrcodeUrl = await QRCode.toDataURL(this.address, opts).catch(err => {
        console.error(err);
      });
      this.isQRCode = true;
    }
  },
  watch: {
    isBalance: function() {
      this.getBalance();
      this.getTokenBalance();
    }
  }
};
</script>

<style>
</style>
