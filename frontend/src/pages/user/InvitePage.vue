<template>
  <div v-if="store.questionnaire && !error">
    <QuestionnairePage></QuestionnairePage>
  </div>
  <div v-if="error">
    <v-alert
      icon="mdi-alert-circle-outline"
      :text="errorMessage"
      color="error"
      rounded
      elevation="3"
      closable
      close-icon="mdi-close"
    ></v-alert>
  </div>

  <v-card
    v-if="!store.questionnaire || error"
    style="margin: 1rem auto 0 auto; padding: 1rem"
    max-width="50%"
    elevation="1"
  >
    <div class="text-h6 font-weight-regular mb-2">
      {{ t('questionnaire.navigation.provideCode') }}
    </div>
    <v-form @submit.prevent v-model="validForm" ref="codeForm">
      <v-text-field
        v-model="store.inviteCode"
        autocomplete="off"
        :label="t('questionnaire.navigation.invitationCode')"
        color="primary"
        validate-on="submit"
        :rules="[
          () => !!store.inviteCode || t('questionnaire.validation.fieldRequired'),
          () => store.inviteCode.length === 8 || t('questionnaire.validation.eightDigits')
        ]"
      ></v-text-field>
      <v-btn type="submit" @click="submitCode()" :loading="isFetching">
        {{ t('questionnaire.navigation.startQuestionnaire') }}</v-btn
      >
    </v-form>
  </v-card>
</template>

<script setup lang="ts">
import { useTypedI18n } from '@/composables/useTypedI18n'
import { useFetch } from '@vueuse/core'
import { ref } from 'vue'
import { useRoute } from 'vue-router'
import { useUserStore } from '@/stores/user'
import type { Questionnaire } from '@/data/interfaces'
import QuestionnairePage from '@/pages/user/QuestionnairePage.vue'
import { useI18n } from 'vue-i18n'
import { watch } from 'vue'
import { computed } from 'vue'
import { onMounted } from 'vue'

const { t } = useTypedI18n()
const i18n = useI18n()
const route = useRoute()
const store = useUserStore()

const validForm = ref(false)
const codeForm = ref<any>(null)
const pathParam = route.params.invitationCode
const errorMessage = ref('')

/**
 * Computed property for the url with the api endpoint, invite code and locale
 */
const url = computed(() => {
  return `${store.apiEndpoint}/api/invitation?invitationCode=${store.inviteCode}&language=${i18n.locale.value}`
})

/**
 * Fetches the api endpoint for questionnaires
 */
const { execute, error, statusCode, isFetching, data } = useFetch(url, { immediate: false })
  .post()
  .json<Questionnaire[]>()

/**
 * Fetches a new questionnaire if one is present in the store and the locale changed
 */
watch(i18n.locale, () => {
  if (store.questionnaire) {
    loadQuestionnaire()
  }
})

/**
 * Validates the form on page reload if an invite code is present
 */
onMounted(() => {
  if (store.inviteCode && store.inviteCode.length != 0 && codeForm.value) codeForm.value.validate()
})

/**
 * If present the function reads the invite code from the path parameter and loads the questionnaire
 */
async function initPage() {
  if (pathParam != null) {
    if (typeof pathParam == 'string') {
      store.inviteCode = pathParam
    }
    if (Array.isArray(pathParam) && pathParam.length != 0) {
      store.inviteCode = pathParam[0]
    }
    await loadQuestionnaire()
  }
}

/**
 * Loads the questionnaire. If multiple are present, it picks the first in the list
 */
async function loadQuestionnaire() {
  await execute()
  if (data.value && !error.value) {
    if (statusCode.value == 202) {
      store.resetSnackbarConfig()
      store.snackbarConfig.color = 'error'
      store.snackbarConfig.message = t('questionnaire.navigation.languageNotAvailable')
      store.snackbarConfig.timeout = '5000'
      store.snackbarConfig.visible = true
    }
    store.questionnaire = data.value[0]
    return
  }
  if (error.value) {
    if (statusCode.value === 404) {
      errorMessage.value = `${t('questionnaire.navigation.invitationCode')} 
      "${store.inviteCode}"
      ${t('questionnaire.common.404')}`
      return
    }
    if (statusCode.value === 400) {
      errorMessage.value = t('questionnaire.navigation.codeAlreadyUsed')
    } else {
      errorMessage.value = t('questionnaire.common.500')
    }
    store.abortQuestionnaire()
  }
}

/**
 * Submits the code for loading a new questionnaire. Validates the form first and doesnt submit if invalid
 */
async function submitCode() {
  await codeForm.value.validate()
  if (validForm.value) {
    await loadQuestionnaire()
  }
}

initPage()
</script>

<style scoped>
.mdi-close {
  color: black !important;
}

@media only screen and (max-width: 600px) {
  .v-card {
    min-width: 95%;
  }
}
</style>
