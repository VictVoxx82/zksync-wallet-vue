<template>
  <i-modal :value="opened" class="prevent-close" size="md" data-cy="account_activation_modal" @hide="close()">
    <template slot="header">Account Activation</template>
    <div class="_text-center">
      <p v-if="state === false">Sign a message to activate your zkSync account.</p>
      <p v-else-if="state === 'processing'" class="_text-center">Processing...</p>
      <p v-else-if="state === 'waitingForUserConfirmation'" class="_text-center">Sign the message in your wallet to continue</p>
      <p v-else-if="state === 'updating'" class="_text-center">Loading account data...</p>
      <i-button
        :disabled="loading || requestingSigner"
        class="_margin-top-2"
        block
        size="lg"
        variant="secondary"
        data-cy="account_activation_sign_button"
        @click="signActivation()"
      >
        <div class="_display-flex _justify-content-center _align-items-center">
          <v-icon v-if="!hasSigner" name="md-vpnkey-round" />&nbsp;&nbsp;
          <div>{{ hasSigner ? "" : "Authorize to " }}Sign account activation</div>
          <loader v-if="loading || requestingSigner" class="_margin-left-1" size="xs" />
        </div>
      </i-button>

      <!-- Requesting signer -->
      <div v-if="requestingSigner" class="_text-center _margin-top-1" data-cy="requesting_signer_text">Follow the instructions in your Ethereum wallet</div>

      <div v-if="error" class="errorText _text-center _margin-top-1">
        {{ error }}
      </div>
    </div>
  </i-modal>
</template>

<script lang="ts">
import Vue from "vue";
import { ZkSignCPKState } from "matter-dapp-module/types";

export default Vue.extend({
  name: "SignPubkey",
  data() {
    return {
      loading: false,
      requestingSigner: false,
    };
  },
  computed: {
    opened(): boolean {
      return this.$accessor.currentModal === "SignPubkey";
    },
    state(): ZkSignCPKState {
      return this.$store.getters["zk-wallet/cpkSignState"];
    },
    hasSigner(): boolean {
      return this.$store.getters["zk-wallet/hasSigner"];
    },
    error(): string | undefined {
      return this.$store.getters["zk-wallet/cpkSignError"];
    },
  },
  methods: {
    close() {
      this.$accessor.closeActiveModal();
      if (this.$store.getters["zk-wallet/cpk"] !== false) {
        return;
      }
      const isForbiddenRoute = () => {
        const forbiddenRoutes = ["/transaction/transfer", "/transaction/withdraw", "/transaction/nft/transfer", "/transaction/nft/withdraw"];
        for (const route of forbiddenRoutes) {
          if (this.$accessor.getPreviousRoute?.path === route || this.$accessor.getPreviousRoute?.path === route + "/") {
            return true;
          }
        }
        return false;
      };
      if (!this.$accessor.getPreviousRoute || isForbiddenRoute()) {
        this.$router.push("/");
      } else {
        // @ts-ignore
        this.$router.push(this.$accessor.getPreviousRoute);
      }
    },
    async signActivation() {
      if (!this.hasSigner) {
        try {
          this.requestingSigner = true;
          await this.$store.dispatch("zk-wallet/requestSigner");
        } catch (err) {
          console.warn("Request signer error\n", err);
        }
        this.requestingSigner = false;
      } else {
        await this.$store.dispatch("zk-wallet/signCPK");
        if (this.$store.getters["zk-wallet/cpk"] !== false) {
          this.close();
        }
      }
    },
  },
});
</script>
