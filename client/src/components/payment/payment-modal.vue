<template>
  <v-dialog max-width="45rem" :value="isVisible" persistent>
    <v-card>

      <v-card-title class="d-flex justify-space-between">
        <span :class="{ 'h4' : isMobile }">Confirm Name Request</span>
        <v-btn v-if="!isMobile" icon large class="dialog-close float-right" @click="hideModal()">
          <v-icon>mdi-close</v-icon>
        </v-btn>
      </v-card-title>

      <v-card-text class="copy-normal pt-0">
        <v-row>
          <v-col cols="12" md="12" lg="12">
            <request-details
                    :applicant="applicant"
                    :name="name"
                    :nameChoices="nameChoices"
            />
          </v-col>
          <v-col cols="12" md="12" lg="12">
            <fee-summary
                    :filingData="[...paymentDetails]"
                    :fees="[...paymentFees]"
            />
          </v-col>
        </v-row>
      </v-card-text>

      <v-card-actions class="pt-4">
        <v-row :class="{ 'text-center' : isMobile }">
          <v-col cols="12" md="7" lg="7" class="py-0">
            <v-btn v-if="allowCancel"
                   @click="cancelPayment()"
                   id="payment-cancel-btn"
                   class="button-red px-4"
                   :class="{ 'mobile-btn' : isMobile }"
                   text
                   :disabled="isLoadingPayment">Cancel Name Request</v-btn>
          </v-col>
          <v-spacer></v-spacer>
          <v-col cols="12" md="3" lg="3" class="py-0">
            <v-btn @click="confirmPayment()"
                   id="payment-pay-btn"
                   class="primary"
                   :class="{ 'mobile-btn' : isMobile }"
                   text
                   :loading="isLoadingPayment">Continue to Payment</v-btn>
          </v-col>
          <v-col cols="12" md="2" lg="2" class="py-0" :class="{ 'text-right' : !isMobile }">
            <v-btn @click="hideModal()"
                   id="payment-close-btn"
                   class="button-blue"
                   :class="{ 'mobile-btn' : isMobile }"
                   :disabled="isLoadingPayment">Close</v-btn>
          </v-col>
        </v-row>
      </v-card-actions>

    </v-card>
  </v-dialog>
</template>

<script lang='ts'>
import { Component, Mixins, Watch } from 'vue-property-decorator'
import FeeSummary from '@/components/payment/fee-summary.vue'
import RequestDetails from '@/components/common/request-details.vue'
import PaymentModule from '@/modules/payment'
import { CreatePaymentParams } from '@/modules/payment/models'
import * as PaymentTypes from '@/modules/payment/store/types'
import * as FilingTypes from '@/modules/payment/filing-types'
import * as Jurisdictions from '@/modules/payment/jurisdictions'
import { PaymentAction } from '@/enums'
import PaymentMixin from '@/components/payment/payment-mixin'
import PaymentSessionMixin from '@/components/payment/payment-session-mixin'
import NameRequestMixin from '@/components/mixins/name-request-mixin'
import DisplayedComponentMixin from '@/components/mixins/displayed-component-mixin'
import { getBaseUrl } from './payment-utils'
import newReqModule from '@/store/new-request-module'

@Component({
  components: {
    RequestDetails,
    FeeSummary
  },
  props: {
    onActivate: {
      type: Function,
      default: async () => undefined
    },
    onCancel: {
      type: Function,
      default: async () => {}
    }
  }
})
export default class PaymentModal extends Mixins(
  NameRequestMixin,
  PaymentMixin,
  PaymentSessionMixin,
  DisplayedComponentMixin
) {
  private isLoadingPayment: boolean = false

  /** The model value for the dialog component. */
  private isVisible = false

  private get allowCancel (): boolean {
    return (typeof this.$props.onCancel === 'function')
  }

  get isMobile (): boolean {
    return window.innerWidth < this.$vuetify.breakpoint.thresholds.xs
  }

  get viewWidth (): string {
    return this.isMobile ? '90vw' : '45rem'
  }

  /** Whether this modal should be shown (per store property). */
  private get showModal (): boolean {
    return PaymentModule[PaymentTypes.PAYMENT_MODAL_IS_VISIBLE]
  }

  /** Clears store property to hide this modal. */
  async hideModal () {
    this.isLoadingPayment = false
    await PaymentModule.togglePaymentModal(false)
  }

  /** Depending on value, fetches fees and makes this modal visible or hides it. */
  @Watch('showModal')
  async onShowModal (val: boolean): Promise<void> {
    if (val) {
      const paymentConfig = {
        filingType: FilingTypes.NM620,
        jurisdiction: Jurisdictions.BC,
        priorityRequest: this.priorityRequest || false
      }

      const { onActivate } = this.$props
      if (typeof onActivate === 'function') {
        onActivate(paymentConfig)
      }

      // only make visible on success, otherwise hide it
      if (await this.fetchFees(paymentConfig)) {
        this.isVisible = true
      } else {
        await this.hideModal()
      }
    } else {
      this.isVisible = false
    }
  }

  /** Called when user clicks "Continue to Payment" button. */
  private async confirmPayment () {
    this.isLoadingPayment = true
    const { nrId, priorityRequest } = this

    const onSuccess = (paymentResponse) => {
      const { paymentId, paymentToken } = this
      // Save response to session
      this.savePaymentResponseToSession(PaymentAction.CREATE, paymentResponse)
      // see if redirect is needed else go to existing NR screen
      const baseUrl = getBaseUrl()
      const redirectUrl = encodeURIComponent(`${baseUrl}/nr/${nrId}/?paymentId=${paymentId}`)
      if (paymentResponse.sbcPayment.isPaymentActionRequired) {
        this.redirectToPaymentPortal(paymentId, paymentToken, redirectUrl)
      } else {
        window.location.href = redirectUrl
      }
    }

    const success = await this.createPayment({
      action: PaymentAction.CREATE,
      nrId: nrId,
      filingType: FilingTypes.NM620,
      priorityRequest: priorityRequest
    } as CreatePaymentParams, onSuccess)

    // on error, close this modal so error modal is visible
    if (!success) {
      this.hideModal()
    }
  }

  /** Called when user clicks "Cancel Name Request" button. */
  async cancelPayment () {
    // rollback the NR
    this.$props.onCancel()
    // close this modal
    this.hideModal()
  }

  @Watch('isVisible')
  onVisibleChanged (val: boolean) {
    if (val) {
      this.$nextTick(() => {
        // add classname to button text (for more detail in Sentry breadcrumbs)
        const confirmNrCancelBtn = this.$el.querySelector("#payment-cancel-btn > span")
        if (confirmNrCancelBtn) confirmNrCancelBtn.classList.add("confirm-nr-cancel-btn")
        const confirmNrContinueBtn = this.$el.querySelector("#payment-pay-btn > span")
        if (confirmNrContinueBtn) confirmNrContinueBtn.classList.add("confirm-nr-continue-btn")
        const confirmNrCloseBtn = this.$el.querySelector("#payment-close-btn > span")
        if (confirmNrCloseBtn) confirmNrCloseBtn.classList.add("confirm-nr-close-btn")
      })
    }
  }
}
</script>
<style scoped lang="scss">
@import "@/assets/scss/theme.scss";

.mobile-btn {
  width: 60vw !important;
  margin: .5rem 0;
  padding: 0 !important;
}
</style>
