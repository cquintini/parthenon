<template>
  <div class="h-full flex flex-col ">
    <h1 class="page-title">{{ $t('app.plan.main.title') }}</h1>

    <div class="alert-error mt-5" v-if="error_message">
      {{ error_message }}
    </div>

    <div class="my-5 text-end">
      {{ $t('app.plan.main.payment_schedule_label') }}
      <select v-model="paymentSchedule">
        <option value="yearly">{{ $t('app.plan.main.payment_schedule_yearly') }}</option>
        <option value="monthly">{{ $t('app.plan.main.payment_schedule_monthly') }}</option>
      </select>
    </div>

    <div class="columns md:flex md:flex-row gap-4 ">
      <div class="column" v-for="(plan, planName) in plans" :class="{'border-2 border-red-500': isCurrentPlan(planName)}">
        <h2 class="h2">{{ plan.name }}</h2>
        <h3 class="text-xl text-red-500" v-if="isCurrentPlan(planName)">{{ $t('app.plan.main.your_current_plan') }}</h3>
        <div class="plan_head_rgt my-3">
          <h4 v-if="paymentSchedule === 'monthly'" class="h1">${{ plan.prices.monthly }}<span>/{{ $t('app.plan.main.payment_schedule_monthly') }}</span></h4>
          <h4 v-else class="h1">${{ plan.prices.yearly }}<span>/{{ $t('app.plan.main.payment_schedule_yearly') }}</span></h4>
      </div>
      <div class="plans_bdy">
        <h6 class="mb-5">{{ $t('app.plan.main.features') }}:</h6>

        <div class="media" v-for="limit in plan.limits">
          <i class="fa fa-check"></i>  {{ limit.limit }} {{ limit.description }}
        </div>
      </div>
      <div class="button-container" v-if="in_progress === false">
        <button class="btn--main block w-full" @click="select(planName, paymentSchedule)" v-if="current_plan === null || !(this.current_plan.status == 'active' || this.current_plan.status == 'pending')">{{ $t('app.plan.main.select_plan') }}</button>
        <button class="btn--main block w-full" @click="change(planName, paymentSchedule)" v-else-if="!isCurrentPlan(planName) || current_plan.payment_schedule !== paymentSchedule">{{ $t('app.plan.main.change') }}</button>
        <button class="btn--main--disabled w-full" disabled v-else>{{ $t('app.plan.main.selected_plan') }}</button>
      </div>
      <div class="button-container" v-else>
        <button class="btn--main--disabled w-full" disabled >
          <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-white inline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          {{ $t('app.plan.main.in_progress') }}</button>
      </div>
    </div>
    </div>

    <div class="bottom-body" v-if="(this.current_plan.status == 'active' || this.current_plan.status == 'pending')">
      <h3 class="h3">{{ $t('app.plan.main.plan_options') }}</h3>

      <a href="/api/payments/portal" class="btn--main mb-3 text-center block">{{ $t('app.plan.main.payment_settings') }}</a>

      <a @click="cancel" class="btn--danger block text-center cursor-pointer	"  v-if="in_progress === false">{{ $t('app.plan.main.cancel_button') }}</a>
      <a @click="cancel" class="btn--danger--disabled block text-center "  v-else>
        <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-white inline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
          <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
          <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
        </svg>
        {{ $t('app.plan.main.in_progress') }}
      </a>
    </div>
  </div>
</template>

<script>
import { planservice } from "../../services/planservice";
import { stripeservice } from "../../services/stripeservice";

export default {
  name: "Plan.vue",
  data() {
    return {
      loading: false,
      sessionId: undefined,
      plans: [],
      current_plan: {},
      paymentSchedule: "yearly",
      showYearly: true,
      stripe: {},
      error_message: undefined,
      in_progress: false,
    }
  },
  mounted() {
    planservice.fetchPlanInfo().then(response =>{
      this.plans = response.data.plans;
      this.current_plan = response.data.current_plan;
      this.stripe = response.data.stripe;
    })
  },
  methods: {
    select: function (planName, paymentSchedule) {
      planservice.createCheckout(planName, paymentSchedule).then(response => {
          stripeservice.redirectToCheckout(this.stripe.api_key, response.data.id)
      })
    },
    isCurrentPlan: function (planName) {
        return planName === this.current_plan.plan_name && (this.current_plan.status == 'active' || this.current_plan.status == 'pending');
    },
    change: function (planName, paymentSchedule) {
      this.in_progress = true;
      planservice.changePlan(planName, paymentSchedule).then(
          response => {
              this.current_plan = response.data.plan;
              this.in_progress = false;
          },
          error => {
              this.error_message = error;
              this.in_progress = false;
          }
      )
    },
    cancel: function () {
      this.in_progress = true;
      planservice.cancel().then(
          response => {
            this.in_progress = false;
          },
          error => {
            this.error_message = error;
            this.in_progress = false;
          }
      )
    }
  }
}
</script>

<style scoped>
.column {
  @apply w-full md:w-2/6 bg-white rounded-xl grow shadow p-5 relative;
  min-height: 600px;
}
.button-container {
  @apply absolute bottom-0 left-0 mb-3 w-full px-5;
}

.bottom-body {
  @apply mt-5 bg-white rounded-xl shadow p-5;
}
</style>