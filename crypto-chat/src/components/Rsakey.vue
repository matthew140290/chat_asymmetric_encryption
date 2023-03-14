<template>
  <div class="keys full-width">
    <h2>SU LLAVE PUBLICA PARCIAL</h2>
    <div class="red" v-if="destinationPublicKey">
      <h3>{{ getKeySnippet(destinationPublicKey) }}</h3>
    </div>
    <h2 v-else>Esperando que un segundo usurio se conecte...</h2>

    <div class="divider"></div>

    <h2>TU LLAVE PUBLICA PARCIAL</h2>
    <div class="green" v-if="originPublicKey">
      <h3>{{ getKeySnippet(originPublicKey) }}</h3>
    </div>
    <div class="full-width" v-else>
      <div class="center-x loader"></div>
      <h2 class="center-text">Generando par de llaves...</h2>
    </div>
  </div>
</template>

<script>
  import helperMixin from '../mixins/helper';
  import {mapGetters} from 'vuex'

  export default {
    name: 'RsaKey',
    mixins: [helperMixin],
    computed: mapGetters([
      'originPublicKey',
      'destinationPublicKey'
    ])
  }
</script>

<style scoped>
  .keys > .divider {
    width: 75%;
    min-width: 0;
    margin: 16px auto;
  }

  .center-x {
    margin: 0 auto;
  }

  .center-text {
    width: 100%;
    text-align: center;
  }

  .divider {
    padding-top: 1px;
    max-height: 0;
    min-width: 200%;
    background: var(--light-grey);
    margin: 12px -12px;
  }

  .green {
    color: green;
  }

  .red {
    color: red;
  }

  .loader {
    border-top: 4px solid var(--highlight);
    border-bottom: 4px solid var(--highlight);
    border-radius: 50%;
    width: 40px;
    height: 40px;
    animation: spin 2s linear infinite;
  }

  @keyframes spin {
    0% {
      transform: rotate(0deg);
    }
    100% {
      transform: rotate(360deg);
    }
  }

  .full-width {
    width: 100%;
  }
</style>
