<template>
  <q-page>
    <h5 class="text-center title">{{ isEditing ? 'Editar' : 'Nueva' }} transacción</h5>
    <q-separator />
    <q-form ref="newEvent" @submit="onSubmit">
      <!-- TODO: Make a user as selected and populate this field -->
      <q-select v-model="payer" :options="payerOptions" label="¿Quien pagó?" :rules="[notEmpty]" />
      <q-input v-model="concept" label="¿En concepto de qué?" />
      <q-input v-model="amount" label="¿Cuánto dolió?" type="number" :rules="[notEmpty]" />
      <div class="q-pa-md currency-container" v-if="currenciesOptions.length > 1">
        <div v-for="currencyOption in currenciesOptions" :key="currencyOption.value">
          <q-radio v-model="currency" :val="currencyOption.value">
            <img :src="getCurrencyImage(currencyOption.value)" :alt="currencyOption.value" /> &nbsp;
            {{ currencyOption.value }}
          </q-radio>
        </div>
      </div>
      <q-input label="Fecha" v-model="date">
        <template v-slot:append>
          <q-icon name="event" class="cursor-pointer">
            <q-popup-proxy ref="qDateProxy" transition-show="scale" transition-hide="scale">
              <q-date v-model="date" @input="() => $refs.qDateProxy.hide()" :mask="dateMask" />
            </q-popup-proxy>
          </q-icon>
        </template>
      </q-input>
      <div class="split-between-container">
        <span class="text-subtitle1">A dividir entre:</span>
        <div class="split-between-container__options">
          <div v-for="leech in splitBetweenOptions" :key="leech">
            <q-checkbox v-model="splitBetween" :val="leech" :label="leech" />
          </div>
        </div>
      </div>
      <div class="flex form-buttons">
        <q-btn
          :label="isEditing ? 'Actualizar' : 'Crear'"
          type="submit"
          class="col-grow"
          color="primary"
        />
        <q-btn
          label="Cancelar"
          type="reset"
          class="col-grow q-ml-sm"
          @click="goBack"
          color="primary"
          flat
        />
      </div>
    </q-form>
  </q-page>
</template>

<script>
import dayjs from 'dayjs';
import { DATE_FORMAT } from '@/utils/constants';
import { notEmpty } from '@/utils/formValidations';

export default {
  name: 'PageNewTranscation',
  created() {
    if (this.$store.state.eventId !== this.$route.params.eventId) {
      //TODO: Move this to a mixin or something
      this.$store
        .dispatch('LOAD_EVENT', this.$route.params.eventId)
        .then(() => {
          this.checkEditing();
        })
        .catch(e => {
          this.$q.notify({
            color: 'red',
            textColor: 'white',
            icon: 'error',
            message: e,
          });
        });
    } else {
      this.checkEditing();
    }
  },
  updated() {
    // TODO: Check this when offline/local stores is activated
    if (!this.currency && this.$store.state.mainCurrency) {
      this.currency = this.$store.state.mainCurrency.value;
    }
  },
  data() {
    return {
      notEmpty,
      id: null,
      payer: null,
      concept: null,
      amount: null,
      currency: null,
      splitBetween: [],
      date: dayjs().format(DATE_FORMAT),
      dateMask: DATE_FORMAT,
    };
  },
  computed: {
    payerOptions() {
      return this.$store.state.participants;
    },
    currenciesOptions() {
      return this.$store.getters.allCurrencies;
    },
    splitBetweenOptions() {
      return this.payerOptions.filter(p => p !== this.payer);
    },
    isEditing() {
      return !!this.$route.params.transactionId;
    },
  },
  methods: {
    checkEditing() {
      const { transactionId } = this.$route.params;
      if (transactionId && !this.id) {
        const transaction = this.$store.state.transactions.find(t => t.id === transactionId);
        this.payer = transaction.payer;
        this.concept = transaction.concept;
        this.amount = transaction.amount;
        this.currency = transaction.currency;
        this.splitBetween = transaction.splitBetween;
        this.date = transaction.date;
      }
    },
    getCurrencyImage(value) {
      return require(`../assets/images/currencies/${value.toLowerCase()}.png`);
    },
    onSubmit() {
      if (this.isEditing) {
        this.updateTransaction();
      } else {
        this.addTransaction();
      }
    },
    addTransaction() {
      this.$store
        .dispatch('ADD_TRANSACTION', {
          payer: this.payer,
          concept: this.concept,
          amount: this.amount,
          currency: this.currency,
          splitBetween: this.splitBetween,
          date: this.date,
        })
        .then(() => {
          this.goBack();
        })
        .catch(e => {
          this.$store.commit('SET_IS_LOADING', false);
          console.error(e);
          this.$q.notify({
            color: 'red',
            textColor: 'white',
            icon: 'error',
            message: 'La transacción no pudo ser creada',
          });
        });
    },
    updateTransaction() {
      this.$store
        .dispatch('UPDATE_TRANSACTION', {
          id: this.$route.params.transactionId,
          payer: this.payer,
          concept: this.concept,
          amount: this.amount,
          currency: this.currency,
          splitBetween: this.splitBetween,
          date: this.date,
        })
        .then(() => {
          this.goBack();
        })
        .catch(e => {
          this.$store.commit('SET_IS_LOADING', false);
          console.error(e);
          this.$q.notify({
            color: 'red',
            textColor: 'white',
            icon: 'error',
            message: 'La transacción no pudo actualizada',
          });
        });
    },
    goBack() {
      window.history.length > 1
        ? this.$router.go(-1)
        : this.$router.push({ name: 'event', params: { id: this.$store.state.eventId } });
    },
  },
  watch: {
    payer() {
      if (!this.isEditing || this._updateSplitFlag) {
        this.splitBetween = this.splitBetweenOptions;
      }
      this._updateSplitFlag = true;
    },
  },
};
</script>

<style scoped>
.q-page {
  padding: 1rem;
}
.title {
  margin: 0.5rem 0;
}
.q-form > * {
  margin: 0.5rem 0;
}
.currency-container {
  display: flex;
  justify-content: space-evenly;
  padding: 0.5rem;
}
.split-between-container__options {
  display: flex;
}
.split-between-container__options > div {
  margin-right: 1rem;
}
.form-buttons {
  margin-top: 2rem;
}
</style>
