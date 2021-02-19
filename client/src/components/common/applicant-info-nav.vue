<template>
  <v-col cols="12" md="5" lg="5" class="py-0" :class="isMobile ? 'text-center' : 'text-right'">
    <v-btn x-large
           id="submit-back-btn"
           :class="isMobile ? 'mobile-btn' : 'mr-3'"
           v-if="showBack"
           @click="back()">
      {{ backText }}
    </v-btn>
    <v-btn x-large
           :class="{ 'mobile-btn' : isMobile }"
           @click="nextAction()"
           :loading="isLoadingSubmission"
           id="submit-continue-btn">
      {{ nextText }}
    </v-btn>
  </v-col>
</template>

<script lang="ts">
import { Component, Emit, Vue } from 'vue-property-decorator'
import NameRequestMixin from '@/components/mixins/name-request-mixin'
import newReqModule from '@/store/new-request-module'

@Component({})
export default class ApplicantInfoNav extends NameRequestMixin {
  @Emit('nextAction')
  private nextAction () : void {}

  get isMobile (): boolean {
    return window.innerWidth < this.$vuetify.breakpoint.thresholds.xs
  }

  get isLoadingSubmission () : boolean {
    return newReqModule.isloadingSubmission
  }

  get backText () {
    if (this.editMode) {
      return 'Previous'
    }
    return 'Back'
  }

  get nextText () {
    if (this.submissionTabNumber === 3) {
      if (this.editMode) {
        return 'Submit Changes'
      }
      return 'Review and Confirm'
    }
    if (this.editMode) {
      return 'Next'
    }
    return 'Continue'
  }

  get showBack () {
    if (this.submissionTabNumber < 2) {
      return false
    }
    if (this.submissionTabNumber === 2) {
      return (this.type === 'examination' || this.nrState === 'DRAFT')
    }
    if (this.submissionTabNumber === 3) {
      return true
    }
    return false
  }
}
</script>
<style scoped lang="scss">
@import "@/assets/scss/theme.scss";

.mobile-btn {
  width: 60vw !important;
  margin: .5rem 0;
}
</style>
