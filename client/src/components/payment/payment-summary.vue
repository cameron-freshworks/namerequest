<template>
  <v-expand-transition>
    <!-- do not display until payments are fetched -->
    <section class="payment-summary" v-if="summary">
      <v-row class="py-5" no-gutters>
        <v-col cols="12" md="auto" lg="auto">
          <div class="align-self-center">{{receiptNumber}}</div>
        </v-col>
        <v-col cols="12" md="auto" lg="auto">
          <div class="align-self-center">{{receiptDate}}</div>
        </v-col>
        <v-col cols="12" md="auto" lg="auto">
          <div class="align-self-center">{{receiptDescription}}</div>
        </v-col>
        <v-col cols="12" md="2" lg="2">
          <div class="align-self-center">${{receiptAmount}}</div>
        </v-col>
        <v-col cols="12" md="auto" lg="auto">
          <v-btn class="download-receipt-btn" :class="{ 'float-right' : !isMobile }" :loading="loading"
                 @click="downloadReceipt()">Download PDF</v-btn>
        </v-col>
      </v-row>
    </section>
  </v-expand-transition>
</template>

<script lang="ts">
import { Component, Mixins, Prop, Watch } from 'vue-property-decorator'
import PaymentMixin from '@/components/payment/payment-mixin'

@Component({})
export default class PaymentSummary extends Mixins(PaymentMixin) {
  @Prop(Object)
  readonly summary: any

  /** Used to show loading state on button. */
  private loading = false

  get isMobile (): boolean {
    return window.innerWidth < this.$vuetify.breakpoint.thresholds.xs
  }

  private get receiptNumber (): string {
    const receiptNumber = this.summary?.receipt.receiptNumber || 'UNK'
    return `Receipt No. ${receiptNumber}`
  }

  private get receiptDate (): string {
    return this.summary?.receipt.receiptDate || 'UNK'
  }

  private get receiptDescription (): string {
    const lineItem = this.summary?.payment.sbcPayment.lineItems[0] // just look at first one
    return lineItem?.description
  }

  private get receiptAmount (): string {
    return `${this.summary?.receipt.receiptAmount.toFixed(2)} CAD`
  }

  public async downloadReceipt () {
    const id = this.summary?.id
    this.loading = true
    await this.downloadReceiptPdf(id)
    this.loading = false
  }

  @Watch('summary', { immediate: true })
  onSummaryChanged (val: any) {
    this.$nextTick(() => {
      // add classname to button text (for more detail in Sentry breadcrumbs)
      const receiptsDownloadBtn = this.$el.querySelector(".download-receipt-btn > span")
      if (receiptsDownloadBtn) receiptsDownloadBtn.classList.add("receipts-download-btn")
    })
  }
}
</script>

<style lang="scss" scoped>
@import "@/assets/scss/theme";

</style>
