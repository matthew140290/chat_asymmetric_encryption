<template>
  <div id="app">
    <Message-panel></Message-panel>
    <side-panel></side-panel>
    <message-input @sendMessage="sendMessage"></message-input>
  </div>
</template>

<script>
  import SidePanel from './components/SidePanel.vue';
  import MessagePanel from './components/MessagePanel';
  import MessageInput from './components/MessageInput';
  import io from 'socket.io-client';
  import CrypticoWorker from 'worker-loader!./workers/cryptico.worker';
  import HelperMixin from './mixins/helper';
  import {eventBus} from './main';
  import {mapGetters, mapActions} from 'vuex'

  export default {
    name: 'App',
    components: {MessagePanel, MessageInput, SidePanel},
    mixins: [HelperMixin],
    data() {
      return {
        webWorker: null,
        socket: null
      }
    },
    computed: mapGetters([
      'selectedRoom',
      'currentRoom',
      'originPublicKey',
      'destinationPublicKey'
    ]),
    async created() {
      this.webWorker = new CrypticoWorker();

      this.notify('Bienvenido! Generando el par de llaves RSA...');
      this.setOriginPublicKey(await this.postWebWorkerMessage('generate-keys'));
      this.notify('Llaves RSA generadas...');

      this.socket = io('http://localhost:3007');
      this.setupSocketListeners();

      this.setupEventListeners();
    },
    destroyed() {
      this.webWorker.terminate();
      this.webWorker = undefined;
    },
    methods: {
      ...mapActions([
        'setCurrentRoom',
        'setOriginPublicKey',
        'setDestinationPublicKey',
        'addMessage',
        'resetMessages',
        'addNotification'
      ]),

      setupEventListeners() {
        eventBus.$on('joinRoom', () => this.joinRoom());
      },

      notify(message) {
        this.addNotification({message: message, timestamp: this.getTimeStamp()});
      },

      postWebWorkerMessage(messageType, messagePayload) {
        return new Promise((resolve) => {
          this.webWorker.postMessage([messageType].concat(messagePayload));
          this.webWorker.addEventListener('message', (response) => {
            resolve(response.data);
          });
        })
      },

      setupSocketListeners() {
        this.socket.on('MESSAGE', async (encryptedMessage) => {
          const decryptedMessage = await this.postWebWorkerMessage('decrypt', encryptedMessage);
          this.addMessage({message: decryptedMessage, origin: false});
          console.log(encryptedMessage)
        });

        this.socket.on('NEW_CONNECTION', () => {
          this.notify('Otro usuario se Unió a la sala.');
          this.sendPublicKey()
        });

        this.socket.on('ROOM_JOINED', (room) => {
          this.setCurrentRoom(room);
          this.notify(`Conectado a la sala - ${this.currentRoom}`);
          this.sendPublicKey()
        });

        this.socket.on('PUBLIC_KEY', (key) => {
          this.notify(`Llave publica recibida - ${this.getKeySnippet(key)}`);
          this.setDestinationPublicKey(key);
        });

        this.socket.on('USER_DISCONNECTED', () => {
          this.notify(`User Disconnected - ${this.getKeySnippet(this.destinationPublicKey)}`);
          this.reset();
        });
        

        this.socket.on('connect', () => this.notify('Conectado al Servidor. Por favor unete a la sala!'));

        this.socket.on('ROOM_FULL', (room) => this.notify(`No te puedes unir ${room}, sala llena`));

        this.socket.on('disconnect', () => this.notify('Conexion perdida'));

        this.socket.on('INTRUSION_ATTEMPT', () => this.notify('intento de intrusión...'));
      },

      async sendMessage(message) {
        if (this.canSendMessage()) {
          this.addMessage({message: message, origin: true});
          const encryptedMessage = await this.postWebWorkerMessage('encrypt', [message, this.destinationPublicKey]);
          this.socket.emit('MESSAGE', encryptedMessage);
        }
      },

      joinRoom() {
        if (this.canJoinRoom()) {
          this.notify(`Conectando a la sala - ${this.selectedRoom}`);
          this.reset();
          this.socket.emit('JOIN', this.selectedRoom)
        }
      },

      reset() {
        this.resetMessages();
        this.setDestinationPublicKey(null);
      },

      sendPublicKey() {
        this.socket.emit('PUBLIC_KEY', this.originPublicKey)
      },

      canSendMessage() {
        return this.originPublicKey && this.destinationPublicKey;
      },

      canJoinRoom() {
        return this.selectedRoom !== this.currentRoom && this.originPublicKey;
      }
    }
  }
</script>

<style>
  :root {
    --black: #111111;
    --light-grey: #d6d6d6;
    --highlight: yellow;
  }

  #app {
    display: flex;
    flex-direction: row;
    flex: 1 0 100%;
    overflow-x: hidden;
  }

  body {
    background: var(--black);
    color: var(--light-grey);
    font-family: 'Roboto Mono', monospace;
    height: 100vh;
    display: flex;
    padding: 0;
    margin: 0;
  }

  div {
    box-sizing: border-box;
  }

  input, textarea, select {
    font-family: inherit;
    font-size: small;
  }

  textarea:focus, input:focus {
    outline: none;
  }

  h1, h2, h3 {
    font-family: 'Montserrat', sans-serif;
  }

  h1 {
    font-size: medium;
  }

  h2 {
    font-size: small;
    font-weight: 300;
  }

  h3 {
    font-size: x-small;
    font-weight: 300;
  }

  p {
    font-size: x-small;
  }
</style>
