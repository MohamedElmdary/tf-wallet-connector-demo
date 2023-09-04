<template>
  <v-app>
    <div class="d-flex justify-center my-16 py-16" v-if="loading">
      <v-progress-circular indeterminate color="primary" />
    </div>
    <v-container v-else-if="!isInstalled">
      <v-alert type="error" variant="tonal">
        Threefold Wallet connector extension is not installed.
        <v-btn class="ml-auto" variant="text" color="primary" @click="reload"
          >Reload</v-btn
        >
      </v-alert>
    </v-container>
    <v-container v-else-if="!hasAccess">
      <v-alert type="error" variant="tonal">
        Failed to granted access permission.
        <v-btn
          class="ml-auto"
          variant="text"
          color="primary"
          @click="requestAccess"
          >Try Again</v-btn
        >
      </v-alert>
    </v-container>
    <v-container v-else>
      <v-card class="mb-10">
        <v-card-title>Selected Account</v-card-title>
        <v-card-text v-if="selectedAccount">
          <pre class="pl-4 pt-4 overflow-x-auto w-100"
            >{{ JSON.stringify(selectedAccount, undefined, 4) }}
            </pre
          >
        </v-card-text>
        <v-card-text v-else>
          <v-alert type="info" variant="tonal">
            Please select an account to decrypt it's seeds and show it
          </v-alert>
        </v-card-text>
      </v-card>

      <v-list>
        <template v-for="(account, index) in accounts" :key="account.address">
          <v-list-group>
            <template #activator="{ props }">
              <v-list-item v-bind="props" class="py-2">
                <v-list-item-title>{{ account.name }}</v-list-item-title>
                <v-list-item-subtitle
                  >{{ account.address }}
                </v-list-item-subtitle>
                <template #append>
                  <v-btn
                    :disabled="
                      selectedAccount?.address === account.address ||
                      (!!selectAccountLoading &&
                        selectAccountLoading !== account.mnemonic)
                    "
                    :loading="selectAccountLoading === account.mnemonic"
                    variant="tonal"
                    color="primary"
                    @click.stop="selectAccount(account)"
                    >{{
                      selectedAccount?.address === account.address
                        ? "Selected"
                        : "Select"
                    }}</v-btn
                  >
                </template>
              </v-list-item>
            </template>

            <pre class="pl-4 pt-4 overflow-x-auto w-100 bg-grey-darken-3"
              >{{ JSON.stringify(account, undefined, 4) }}
            </pre>
          </v-list-group>
          <v-divider class="mt-1 pb-1" v-if="accounts.length - 1 !== index" />
        </template>
      </v-list>
    </v-container>
  </v-app>
</template>

<script lang="ts">
import { onMounted } from "vue"
import {
  ThreefoldWalletConnectorApi,
  type Account,
} from "tf-wallet-connector-api"
import { ref } from "vue"

export default {
  name: "App",
  setup() {
    const isInstalled = ref(false)
    const hasAccess = ref(false)

    const loading = ref(false)
    const accounts = ref<Account[]>([])

    onMounted(init)
    async function init() {
      isInstalled.value = await ThreefoldWalletConnectorApi.isInstalled()

      if (isInstalled.value) {
        await requestAccess()
      }
    }

    const selectAccountLoading = ref("")
    const selectedAccount = ref<Account>()
    async function selectAccount(account: Account) {
      selectAccountLoading.value = account.mnemonic
      const mnemonic =
        await ThreefoldWalletConnectorApi.requestDecryptedAccount(
          account.mnemonic
        )
      selectedAccount.value = { ...account, mnemonic }
      selectAccountLoading.value = ""
    }

    async function requestAccess() {
      loading.value = true
      hasAccess.value = await ThreefoldWalletConnectorApi.requestAccess()
      console.log(hasAccess.value)

      if (hasAccess.value) {
        ThreefoldWalletConnectorApi.listenToPublicAccounts((accs) => {
          if (selectedAccount.value) {
            const newAccountsSet = new Set(accs.map((acc) => acc.address))
            if (!newAccountsSet.has(selectedAccount.value.address)) {
              selectedAccount.value = undefined
            }
          }
          accounts.value = accs
        })
      }
      loading.value = false
    }

    return {
      reload() {
        window.location.href = window.location.href
      },
      requestAccess,
      isInstalled,
      hasAccess,
      loading,
      accounts,
      selectAccount,
      selectedAccount,
      selectAccountLoading,
    }
  },
}
</script>
