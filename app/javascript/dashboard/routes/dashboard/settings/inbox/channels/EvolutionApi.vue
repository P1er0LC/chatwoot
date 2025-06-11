<script>
import { mapGetters } from 'vuex';
import { useVuelidate } from '@vuelidate/core';
import { useAlert } from 'dashboard/composables';
import { required } from '@vuelidate/validators';
import router from '../../../../index';
import PageHeader from '../../SettingsSubPageHeader.vue';
import NextButton from 'dashboard/components-next/button/Button.vue';

const shouldBeWebhookUrl = (value = '') =>
  value ? value.startsWith('http') : true;

const shouldBeEvolutionApiUrl = (value = '') =>
  value ? value.startsWith('http') : true;

export default {
  components: {
    PageHeader,
    NextButton,
  },
  setup() {
    return { v$: useVuelidate() };
  },
  data() {
    return {
      channelName: '',
      webhookUrl: '',
      evolutionApiUrl: '',
      evolutionApiKey: '',
      instanceName: '',
      qrCode: '',
      isCreatingInstance: false,
      isGeneratingQR: false,
      instanceCreated: false,
      qrGenerated: false,
      step: 1, // 1: Config básica, 2: Crear instancia, 3: Generar QR, 4: Conectar con Chatwoot
      instanceData: null,
    };
  },
  computed: {
    ...mapGetters({
      uiFlags: 'inboxes/getUIFlags',
    }),
    canProceedToStep2() {
      return this.channelName && this.evolutionApiUrl && this.evolutionApiKey;
    },
    canProceedToStep3() {
      return this.instanceCreated && this.instanceName;
    },
    canProceedToStep4() {
      return this.qrGenerated && this.qrCode;
    },
  },
  validations: {
    channelName: { required },
    webhookUrl: { shouldBeWebhookUrl },
    evolutionApiUrl: { required, shouldBeEvolutionApiUrl },
    evolutionApiKey: { required },
    instanceName: { required },
  },
  methods: {
    async createInstance() {
      this.v$.$touch();
      if (this.v$.$invalid) {
        return;
      }

      this.isCreatingInstance = true;
      
      try {
        // Llamada a la API de Evolution para crear una instancia
        const response = await fetch(`${this.evolutionApiUrl}/instance/create`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'apikey': this.evolutionApiKey,
          },
          body: JSON.stringify({
            instanceName: this.instanceName,
            token: this.evolutionApiKey,
            qrcode: true,
            webhook: this.webhookUrl,
            webhook_by_events: false,
            webhook_base64: false,
            events: [
              'APPLICATION_STARTUP',
              'QRCODE_UPDATED',
              'MESSAGES_UPSERT',
              'MESSAGES_UPDATE',
              'MESSAGES_DELETE',
              'SEND_MESSAGE',
              'CONTACTS_SET',
              'CONTACTS_UPSERT',
              'CONTACTS_UPDATE',
              'PRESENCE_UPDATE',
              'CHATS_SET',
              'CHATS_UPSERT',
              'CHATS_UPDATE',
              'CHATS_DELETE',
              'GROUPS_UPSERT',
              'GROUP_UPDATE',
              'GROUP_PARTICIPANTS_UPDATE',
              'CONNECTION_UPDATE'
            ]
          }),
        });

        if (!response.ok) {
          throw new Error('Error al crear la instancia');
        }

        const data = await response.json();
        this.instanceData = data;
        this.instanceCreated = true;
        this.step = 2;
        
        useAlert('Instancia creada exitosamente');
      } catch (error) {
        console.error('Error creating instance:', error);
        useAlert('Error al crear la instancia de Evolution API');
      } finally {
        this.isCreatingInstance = false;
      }
    },

    async generateQR() {
      this.isGeneratingQR = true;
      
      try {
        // Conectar la instancia para generar QR
        const connectResponse = await fetch(`${this.evolutionApiUrl}/instance/connect/${this.instanceName}`, {
          method: 'GET',
          headers: {
            'apikey': this.evolutionApiKey,
          },
        });

        if (!connectResponse.ok) {
          throw new Error('Error al conectar la instancia');
        }

        // Obtener el QR code
        const qrResponse = await fetch(`${this.evolutionApiUrl}/instance/qrcode/${this.instanceName}`, {
          method: 'GET',
          headers: {
            'apikey': this.evolutionApiKey,
          },
        });

        if (!qrResponse.ok) {
          throw new Error('Error al obtener el código QR');
        }

        const qrData = await qrResponse.json();
        this.qrCode = qrData.base64; // o qrData.code dependiendo de cómo devuelva la API
        this.qrGenerated = true;
        this.step = 3;
        
        useAlert('Código QR generado. Escanéalo con WhatsApp para conectar.');
      } catch (error) {
        console.error('Error generating QR:', error);
        useAlert('Error al generar el código QR');
      } finally {
        this.isGeneratingQR = false;
      }
    },

    async createChannel() {
      this.v$.$touch();
      if (this.v$.$invalid) {
        return;
      }

      try {
        const evolutionChannel = await this.$store.dispatch('inboxes/createChannel', {
          name: this.channelName,
          channel: {
            type: 'evolution_api',
            webhook_url: this.webhookUrl,
            evolution_api_url: this.evolutionApiUrl,
            evolution_api_key: this.evolutionApiKey,
            instance_name: this.instanceName,
            instance_data: this.instanceData,
          },
        });

        router.replace({
          name: 'settings_inboxes_add_agents',
          params: {
            page: 'new',
            inbox_id: evolutionChannel.id,
          },
        });
      } catch (error) {
        useAlert('Error al crear el canal de Evolution API');
      }
    },

    goToNextStep() {
      if (this.step === 1) {
        this.createInstance();
      } else if (this.step === 2) {
        this.generateQR();
      } else if (this.step === 3) {
        this.createChannel();
      }
    },

    resetForm() {
      this.step = 1;
      this.instanceCreated = false;
      this.qrGenerated = false;
      this.qrCode = '';
      this.instanceData = null;
    },
  },
};
</script>

<template>
  <div
    class="border border-n-weak bg-n-solid-1 rounded-t-lg border-b-0 h-full w-full p-6 col-span-6 overflow-auto"
  >
    <PageHeader
      :header-title="$t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.TITLE')"
      :header-content="$t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.DESC')"
    />

    <!-- Progress Steps -->
    <div class="mb-6">
      <div class="flex justify-between items-center">
        <div class="flex items-center space-x-4">
          <div :class="['w-8 h-8 rounded-full flex items-center justify-center text-sm font-medium', 
                       step >= 1 ? 'bg-blue-500 text-white' : 'bg-gray-200 text-gray-600']">
            1
          </div>
          <span :class="step >= 1 ? 'text-blue-600' : 'text-gray-500'">Configuración</span>
        </div>
        <div class="flex items-center space-x-4">
          <div :class="['w-8 h-8 rounded-full flex items-center justify-center text-sm font-medium', 
                       step >= 2 ? 'bg-blue-500 text-white' : 'bg-gray-200 text-gray-600']">
            2
          </div>
          <span :class="step >= 2 ? 'text-blue-600' : 'text-gray-500'">Crear Instancia</span>
        </div>
        <div class="flex items-center space-x-4">
          <div :class="['w-8 h-8 rounded-full flex items-center justify-center text-sm font-medium', 
                       step >= 3 ? 'bg-blue-500 text-white' : 'bg-gray-200 text-gray-600']">
            3
          </div>
          <span :class="step >= 3 ? 'text-blue-600' : 'text-gray-500'">Generar QR</span>
        </div>
        <div class="flex items-center space-x-4">
          <div :class="['w-8 h-8 rounded-full flex items-center justify-center text-sm font-medium', 
                       step >= 4 ? 'bg-green-500 text-white' : 'bg-gray-200 text-gray-600']">
            4
          </div>
          <span :class="step >= 4 ? 'text-green-600' : 'text-gray-500'">Conectar</span>
        </div>
      </div>
    </div>

    <form
      class="flex flex-wrap flex-col mx-0"
      @submit.prevent="goToNextStep()"
    >
      <!-- Step 1: Basic Configuration -->
      <div v-if="step === 1">
        <h3 class="text-lg font-medium mb-4">Configuración Básica</h3>
        
        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.channelName.$error }">
            {{ $t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.CHANNEL_NAME.LABEL') }}
            <input
              v-model="channelName"
              type="text"
              :placeholder="$t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.CHANNEL_NAME.PLACEHOLDER')"
              @blur="v$.channelName.$touch"
            />
            <span v-if="v$.channelName.$error" class="message">{{
              $t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.CHANNEL_NAME.ERROR')
            }}</span>
          </label>
        </div>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.evolutionApiUrl.$error }">
            URL de Evolution API
            <input
              v-model="evolutionApiUrl"
              type="text"
              placeholder="https://your-evolution-api-url.com"
              @blur="v$.evolutionApiUrl.$touch"
            />
            <span v-if="v$.evolutionApiUrl.$error" class="message">
              La URL de Evolution API es requerida
            </span>
          </label>
        </div>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.evolutionApiKey.$error }">
            API Key de Evolution
            <input
              v-model="evolutionApiKey"
              type="password"
              placeholder="Tu API Key de Evolution"
              @blur="v$.evolutionApiKey.$touch"
            />
            <span v-if="v$.evolutionApiKey.$error" class="message">
              La API Key es requerida
            </span>
          </label>
        </div>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.webhookUrl.$error }">
            {{ $t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.WEBHOOK_URL.LABEL') }}
            <input
              v-model="webhookUrl"
              type="text"
              :placeholder="$t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.WEBHOOK_URL.PLACEHOLDER')"
              @blur="v$.webhookUrl.$touch"
            />
          </label>
          <p class="help-text">
            {{ $t('INBOX_MGMT.ADD.EVOLUTION_API_CHANNEL.WEBHOOK_URL.SUBTITLE') }}
          </p>
        </div>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.instanceName.$error }">
            Nombre de la Instancia
            <input
              v-model="instanceName"
              type="text"
              placeholder="mi-instancia-whatsapp"
              @blur="v$.instanceName.$touch"
            />
            <span v-if="v$.instanceName.$error" class="message">
              El nombre de la instancia es requerido
            </span>
          </label>
          <p class="help-text">
            Nombre único para identificar esta instancia de WhatsApp
          </p>
        </div>
      </div>

      <!-- Step 2: Instance Created -->
      <div v-else-if="step === 2">
        <h3 class="text-lg font-medium mb-4">Instancia Creada</h3>
        <div class="bg-green-50 border border-green-200 rounded-md p-4 mb-4">
          <div class="flex">
            <div class="flex-shrink-0">
              <svg class="h-5 w-5 text-green-400" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
              </svg>
            </div>
            <div class="ml-3">
              <h3 class="text-sm font-medium text-green-800">
                Instancia creada exitosamente
              </h3>
              <div class="mt-2 text-sm text-green-700">
                <p>La instancia "{{ instanceName }}" ha sido creada correctamente.</p>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Step 3: QR Code -->
      <div v-else-if="step === 3">
        <h3 class="text-lg font-medium mb-4">Código QR</h3>
        <div v-if="qrCode" class="text-center mb-4">
          <img :src="`data:image/png;base64,${qrCode}`" alt="QR Code" class="mx-auto mb-4" />
          <p class="text-sm text-gray-600">
            Escanea este código QR con WhatsApp para conectar tu cuenta
          </p>
        </div>
        <div v-else class="text-center">
          <div class="animate-spin rounded-full h-32 w-32 border-b-2 border-blue-500 mx-auto mb-4"></div>
          <p>Generando código QR...</p>
        </div>
      </div>

      <div class="w-full mt-4 flex space-x-2">
        <NextButton
          v-if="step > 1"
          @click="resetForm"
          type="button"
          :label="'Reiniciar'"
        />
        <NextButton
          :is-loading="isCreatingInstance || isGeneratingQR || uiFlags.isCreating"
          type="submit"
          solid
          blue
          :disabled="(step === 1 && !canProceedToStep2) || 
                    (step === 2 && !canProceedToStep3) || 
                    (step === 3 && !canProceedToStep4)"
          :label="step === 1 ? 'Crear Instancia' : 
                  step === 2 ? 'Generar QR' : 
                  step === 3 ? 'Conectar WhatsApp' : 
                  'Finalizar'"
        />
      </div>
    </form>
  </div>
</template>