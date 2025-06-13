<script>
import { mapGetters } from 'vuex';
import { useVuelidate } from '@vuelidate/core';
import { useAlert } from 'dashboard/composables';
import { required } from '@vuelidate/validators';
// import router from '../../../../index';
import PageHeader from '../../SettingsSubPageHeader.vue';
import NextButton from 'dashboard/components-next/button/Button.vue';

const shouldBeUrl = (value = '') => {
  if (!value) return false;
  try {
    const url = new URL(value);
    return url.protocol === 'http:' || url.protocol === 'https:';
  } catch {
    return false;
  }
};

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
      evolutionApiUrl: '',
      instanceName: '',
      apiKey: '',
      qrCode: '',
      isCreatingInstance: false,
      isGeneratingQR: false,
      isConnectingChatwoot: false,
      instanceCreated: false,
      qrGenerated: false,
      connectionEstablished: false,
      chatwootConfigured: false,
      step: 1,
      checkInterval: null,
      errorMessage: '',
      debugInfo: null,
    };
  },
  computed: {
    ...mapGetters({
      // uiFlags: 'inboxes/getUIFlags',
      accountId: 'getCurrentAccountId',
      // authToken: 'getAuthToken',
    }),
    chatwootBaseUrl() {
      return window.location.origin;
    },
    qrImageSrc() {
      // return this.qrCode ? `data:image/png;base64,${this.qrCode}` : '';
      if (!this.qrCode) return '';

      // Verificar si ya tiene el prefijo data:image
      if (this.qrCode.startsWith('data:image')) {
        return this.qrCode;
      }

      // Si no tiene el prefijo, añadirlo
      return `data:image/png;base64,${this.qrCode}`;
    },
  },
  validations: {
    channelName: { required },
    evolutionApiUrl: { required, shouldBeUrl },
    instanceName: { required },
    apiKey: { required },
  },
  beforeDestroy() {
    this.clearCheckInterval();
  },
  methods: {
    clearCheckInterval() {
      if (this.checkInterval) {
        clearInterval(this.checkInterval);
        this.checkInterval = null;
      }
    },

    // PASO 1: Crear instancia básica (sin configuraciones de Chatwoot)
    async createInstance() {
      // console.log('Iniciando creación de instancia...');

      this.v$.$touch();
      if (this.v$.$invalid) {
        // console.log('Formulario inválido');
        return;
      }

      try {
        this.isCreatingInstance = true;
        this.errorMessage = '';

        const cleanUrl = this.evolutionApiUrl.replace(/\/$/, '');

        // Crear instancia básica
        const requestBody = {
          instanceName: this.instanceName,
          token: '',
          qrcode: true,
          integration: 'WHATSAPP-BAILEYS',
          // Configuraciones básicas mínimas
          reject_call: false,
          groupsIgnore: true,
          alwaysOnline: false,
          readMessages: false,
          readStatus: false,
          syncFullHistory: false,
        };

        /* console.log('Creando instancia...',
          JSON.stringify(requestBody, null, 2)
        ); */

        const response = await fetch(`${cleanUrl}/instance/create`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            apikey: this.apiKey,
          },
          body: JSON.stringify(requestBody),
        });

        if (!response.ok) {
          const errorData = await response.json().catch(() => ({}));
          throw new Error(errorData.message || `Error HTTP ${response.status}`);
        }

        // const data = await response.json();
        // console.log('Instancia creada:', data);

        this.instanceCreated = true;
        this.step = 2;
        await this.generateQR();

        useAlert('Instancia creada exitosamente');
      } catch (error) {
        // console.error('Error creating instance:', error);
        this.errorMessage = error.message;
        useAlert(`Error al crear la instancia: ${error.message}`);
      } finally {
        this.isCreatingInstance = false;
      }
    },

    // PASO 2: Generar código QR
    async generateQR() {
      try {
        this.isGeneratingQR = true;
        this.errorMessage = '';

        const cleanUrl = this.evolutionApiUrl.replace(/\/$/, '');

        // Un solo endpoint para conectar y obtener QR
        // console.log('Conectando instancia y obteniendo QR...');
        const response = await fetch(
          `${cleanUrl}/instance/connect/${this.instanceName}`,
          {
            method: 'GET',
            headers: {
              'Content-Type': 'application/json',
              apikey: this.apiKey,
            },
          }
        );

        if (!response.ok) {
          const errorData = await response.json().catch(() => ({}));
          throw new Error(
            errorData.message || 'Error al conectar la instancia'
          );
        }

        const data = await response.json();
        // console.log('Respuesta de connect:', data);

        // Extraer el QR de la respuesta
        if (data.qrcode) {
          this.qrCode = data.qrcode.base64 || data.qrcode;
        } else if (data.base64) {
          this.qrCode = data.base64;
        } else if (data.qr) {
          this.qrCode = data.qr;
        } else {
          // Buscar en otros lugares de la respuesta
          /* console.log(
            'Estructura completa de respuesta:',
            JSON.stringify(data)
          ); */
          throw new Error('No se pudo encontrar el código QR en la respuesta');
        }

        this.qrGenerated = true;
        this.startConnectionCheck();

        useAlert('Código QR generado. Escanéalo con WhatsApp para conectar.');
      } catch (error) {
        // console.error('Error generating QR:', error);
        this.errorMessage = error.message;
        useAlert(`Error al generar el código QR: ${error.message}`);
      } finally {
        this.isGeneratingQR = false;
      }
    },

    // Verificar estado de conexión
    startConnectionCheck() {
      this.clearCheckInterval();
      // console.log('Iniciando verificación de conexión...');

      this.checkInterval = setInterval(async () => {
        try {
          const isConnected = await this.checkConnectionStatus();
          if (isConnected) {
            // console.log('Conexión establecida!');
            this.clearCheckInterval();
            this.connectionEstablished = true;
            this.step = 3;
            useAlert('¡Conexión establecida exitosamente!');
          }
        } catch (error) {
          // console.error('Error checking connection:', error);
        }
      }, 5000); // Verificar cada 5 segundos
    },

    async checkConnectionStatus() {
      try {
        const cleanUrl = this.evolutionApiUrl.replace(/\/$/, '');

        const response = await fetch(
          `${cleanUrl}/instance/connectionState/${this.instanceName}`,
          {
            method: 'GET',
            headers: {
              'Content-Type': 'application/json',
              apikey: this.apiKey,
            },
          }
        );

        if (!response.ok) {
          return false;
        }

        const data = await response.json();
        // console.log('Connection status:', data);

        return (
          data.state === 'open' ||
          data.status === 'connected' ||
          data.instance?.state === 'open' ||
          data.instance?.status === 'connected'
        );
      } catch (error) {
        // console.error('Error checking connection status:', error);
        return false;
      }
    },

    // PASO 3: Configurar integración con Chatwoot
    // a ver
    async configureEvolutionApiChatwoot() {
      try {
        this.isConnectingChatwoot = true;
        this.errorMessage = '';

        // console.log('Iniciando configuración de Chatwoot en Evolution API...');

        // Configuración para Evolution API según documentación
        const chatwootConfig = {
          enabled: true,
          accountId: `${this.accountId}`,
          token: 'M6w2VMrpUjivx6PC11v9KfM5', // Evolution API generará un token
          url: this.chatwootBaseUrl,
          signMsg: true,
          reopenConversation: true,
          conversationPending: true,
          nameInbox: this.channelName,
          mergeBrazilContacts: false,
          importContacts: true,
          importMessages: true,
          daysLimitImportMessages: 1,
          autoCreate: true, // Permitir que Evolution API cree el canal
          polling: {
            enabled: true,
            interval: 5000,
          },
        };

        // console.log('Configuración a enviar:', JSON.stringify(chatwootConfig));

        const cleanUrl = this.evolutionApiUrl.replace(/\/$/, '');
        const endpoint = `${cleanUrl}/chatwoot/set/${this.instanceName}`;

        // console.log('Enviando solicitud a:', endpoint);

        // Realizar la petición a Evolution API
        const response = await fetch(endpoint, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            apikey: this.apiKey,
          },
          body: JSON.stringify(chatwootConfig),
        });

        // Intentar obtener el cuerpo de la respuesta incluso si hay error
        let responseBody;
        try {
          responseBody = await response.json();
          // console.log('Respuesta recibida:', JSON.stringify(responseBody));
        } catch (parseError) {
          // console.error('Error al parsear respuesta:', parseError);
          responseBody = null;
        }

        // Verificar si hubo error en la respuesta
        if (!response.ok) {
          throw new Error(
            responseBody?.message ||
              responseBody?.error?.message ||
              `Error ${response.status}: ${response.statusText}`
          );
        }

        // Respuesta exitosa
        this.chatwootConfigured = true;
        useAlert('Canal de Evolution API configurado exitosamente');

        // Esperar un momento para que Evolution API termine de crear el inbox
        await new Promise(resolve => setTimeout(resolve, 2000));

        // Navegar al paso de añadir agentes (settings_inboxes_add_agents)
        // Primero obtener el listado de inboxes para buscar nuestro canal recién creado
        /* const inboxes = await this.$store.dispatch('inboxes/get');
        console.log('Inboxes disponibles:', inboxes);

        // Buscar el inbox recién creado por nombre
        const targetInbox = inboxes.find(
          inbox =>
            inbox.name === this.channelName ||
            inbox.name === `${this.channelName}` ||
            inbox.name === 'nueva'
        );

        if (targetInbox) {
          console.log('Inbox encontrado:', targetInbox);
          router.replace({
            name: 'settings_inboxes_add_agents',
            params: {
              page: 'new',
              inbox_id: targetInbox.id,
            },
          });
        } else {
          // Si no encontramos el inbox, redirigir al listado general
          console.log(
            'No se encontró el inbox específico, redirigiendo al listado'
          );
          router.replace({
            name: 'settings_inbox_show',
          });
        } */
      } catch (error) {
        // console.error('Error al configurar integración:', error);
        this.errorMessage = error.message;
        useAlert(`Error al configurar integración: ${error.message}`);
      } finally {
        this.isConnectingChatwoot = false;
      }
    },

    // Configurar integración Chatwoot en Evolution API
    // versión más estable
    // nuevo metodo

    goToNextStep() {
      if (this.step === 1) {
        this.createInstance();
      } else if (this.step === 2) {
        if (!this.qrCode) {
          this.generateQR();
        }
      } else if (this.step === 3) {
        this.configureEvolutionApiChatwoot();
      }
    },

    resetForm() {
      this.clearCheckInterval();
      this.step = 1;
      this.instanceCreated = false;
      this.qrGenerated = false;
      this.connectionEstablished = false;
      this.chatwootConfigured = false;
      this.qrCode = '';
      this.errorMessage = '';
      this.debugInfo = null;
    },

    regenerateQR() {
      this.qrCode = '';
      this.qrGenerated = false;
      this.generateQR();
    },
  },
};
</script>

<template>
  <div
    class="border border-n-weak bg-n-solid-1 rounded-t-lg border-b-0 h-full w-full p-6 col-span-6 overflow-auto"
  >
    <PageHeader
      :header-title="'Canal Evolution API WhatsApp'"
      :header-content="'Integra WhatsApp a través de Evolution API para gestionar tus conversaciones'"
    />

    <!-- Mostrar errores
    <div v-if="errorMessage" class="mb-4 p-3 bg-red-100 border border-red-400 text-red-700 rounded">
      <strong>Error:</strong> {{ errorMessage }}
      <button 
        v-if="step === 3" 
        @click="diagnoseConfiguration" 
        class="ml-2 px-2 py-1 bg-blue-500 text-white text-xs rounded">
        Diagnosticar
      </button>
    </div> -->

    <!-- Debug info -->
    <div v-if="debugInfo" class="mb-4 p-3 bg-gray-100 border rounded text-xs">
      <details>
        <summary class="cursor-pointer font-medium">
          Información de diagnóstico
        </summary>
        <pre class="mt-2 whitespace-pre-wrap">{{
          JSON.stringify(debugInfo, null, 2)
        }}</pre>
      </details>
    </div>

    <!-- Indicador de pasos -->
    <div class="mb-6">
      <div class="flex items-center w-full">
        <div class="flex flex-col items-center w-1/3">
          <div
            class="w-8 h-8 rounded-full flex items-center justify-center"
            :class="[step >= 1 ? 'bg-blue-500 text-white' : 'bg-gray-200']"
          >
            1
          </div>
          <div class="text-xs mt-1">Crear Instancia</div>
        </div>
        <div
          class="flex-1 h-1"
          :class="[step >= 2 ? 'bg-blue-500' : 'bg-gray-200']"
        ></div>
        <div class="flex flex-col items-center w-1/3">
          <div
            class="w-8 h-8 rounded-full flex items-center justify-center"
            :class="[step >= 2 ? 'bg-blue-500 text-white' : 'bg-gray-200']"
          >
            2
          </div>
          <div class="text-xs mt-1">Conectar WhatsApp</div>
        </div>
        <div
          class="flex-1 h-1"
          :class="[step >= 3 ? 'bg-blue-500' : 'bg-gray-200']"
        ></div>
        <div class="flex flex-col items-center w-1/3">
          <div
            class="w-8 h-8 rounded-full flex items-center justify-center"
            :class="[step >= 3 ? 'bg-blue-500 text-white' : 'bg-gray-200']"
          >
            3
          </div>
          <div class="text-xs mt-1">Configurar Chatwoot</div>
        </div>
      </div>
    </div>

    <form class="flex flex-wrap flex-col mx-0" @submit.prevent="goToNextStep()">
      <!-- Paso 1: Configuración básica -->
      <div v-if="step === 1">
        <h3 class="text-lg font-medium mb-4">Configuración de Evolution API</h3>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.channelName.$error }">
            Nombre del Canal
            <input
              v-model="channelName"
              type="text"
              placeholder="Mi WhatsApp Business"
              @blur="v$.channelName.$touch"
            />
            <span v-if="v$.channelName.$error" class="message">
              El nombre del canal es requerido
            </span>
          </label>
        </div>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.evolutionApiUrl.$error }">
            URL de Evolution API
            <input
              v-model="evolutionApiUrl"
              type="text"
              placeholder="https://tu-evolution-api.com"
              @blur="v$.evolutionApiUrl.$touch"
            />
            <span v-if="v$.evolutionApiUrl.$error" class="message">
              Se requiere una URL válida (debe empezar con http:// o https://)
            </span>
          </label>
        </div>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.apiKey.$error }">
            API Key
            <input
              v-model="apiKey"
              type="password"
              placeholder="API Key de Evolution API"
              @blur="v$.apiKey.$touch"
            />
            <span v-if="v$.apiKey.$error" class="message">
              La API Key es requerida
            </span>
          </label>
        </div>

        <div class="flex-shrink-0 flex-grow-0 mb-4">
          <label :class="{ error: v$.instanceName.$error }">
            Nombre de la Instancia
            <input
              v-model="instanceName"
              type="text"
              placeholder="whatsapp-instance"
              @blur="v$.instanceName.$touch"
            />
            <span v-if="v$.instanceName.$error" class="message">
              El nombre de la instancia es requerido
            </span>
            <small class="text-gray-500 block mt-1">
              Solo caracteres alfanuméricos, guiones y guiones bajos
            </small>
          </label>
        </div>
      </div>

      <!-- Paso 2: Código QR -->
      <div v-else-if="step === 2">
        <h3 class="text-lg font-medium mb-4">Conectar WhatsApp</h3>
        <p class="mb-4">Escanea el código QR con WhatsApp Web:</p>
        <ol class="list-decimal list-inside mb-4 text-sm space-y-1">
          <li>Abre WhatsApp en tu teléfono</li>
          <li>Ve a Menú > WhatsApp Web</li>
          <li>Escanea el código QR de abajo</li>
        </ol>

        <div class="flex justify-center mb-4">
          <div v-if="qrCode" class="border p-4 max-w-xs bg-white rounded">
            <img :src="qrImageSrc" alt="WhatsApp QR Code" class="w-full" />
          </div>
          <div
            v-else
            class="flex items-center justify-center h-48 w-48 border bg-gray-50 rounded"
          >
            <div class="text-center">
              <div
                v-if="isGeneratingQR"
                class="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-500 mx-auto mb-2"
              ></div>
              <span v-if="isGeneratingQR" class="text-sm"
                >Generando código QR...</span
              >
              <span v-else class="text-sm text-gray-500"
                >Código QR no disponible</span
              >
            </div>
          </div>
        </div>

        <div class="text-center mb-4">
          <p class="text-sm text-gray-600 mb-2">
            El código QR expira en 2 minutos.
          </p>
          <button
            type="button"
            @click="regenerateQR"
            class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 disabled:opacity-50"
            :disabled="isGeneratingQR"
          >
            {{ isGeneratingQR ? 'Generando...' : 'Regenerar QR' }}
          </button>
        </div>

        <div
          v-if="connectionEstablished"
          class="text-center p-4 bg-green-100 border border-green-400 text-green-700 rounded mb-4"
        >
          <svg
            class="h-5 w-5 text-green-500 mx-auto mb-2"
            fill="currentColor"
            viewBox="0 0 20 20"
          >
            <path
              fill-rule="evenodd"
              d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z"
              clip-rule="evenodd"
            />
          </svg>
          ¡WhatsApp conectado exitosamente!
        </div>
      </div>

      <!-- Paso 3: Configuración Chatwoot -->
      <div v-else-if="step === 3">
        <h3 class="text-lg font-medium mb-4">Configurar Canal en Chatwoot</h3>
        <div class="bg-green-50 border border-green-200 rounded-lg p-4 mb-4">
          <div class="flex">
            <div class="flex-shrink-0">
              <svg
                class="h-5 w-5 text-green-400"
                fill="currentColor"
                viewBox="0 0 20 20"
              >
                <path
                  fill-rule="evenodd"
                  d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z"
                  clip-rule="evenodd"
                />
              </svg>
            </div>
            <div class="ml-3">
              <h3 class="text-sm font-medium text-green-800">
                WhatsApp conectado exitosamente
              </h3>
              <p class="mt-1 text-sm text-green-700">
                La instancia <strong>{{ instanceName }}</strong> está conectada
                y lista.
              </p>
            </div>
          </div>
        </div>

        <div class="mb-4">
          <h4 class="font-medium mb-2">Se configurará automáticamente:</h4>
          <ul class="list-disc list-inside text-sm space-y-1 text-gray-600">
            <!-- <li>Webhook entre Evolution API y Chatwoot</li>
            <li>Integración bidireccional de mensajes</li> -->
            <li>Canal API en Chatwoot con el nombre "{{ channelName }}"</li>
            <li>Importación de contactos y conversaciones</li>
            <li>
              ¡Después de la confirmación dirígete a la bandeja de entrada!
            </li>
          </ul>
        </div>

        <p class="mb-4 text-sm">
          Haz clic en <strong>Finalizar Configuración</strong> para completar la
          integración.
        </p>
      </div>

      <div class="w-full mt-8">
        <NextButton
          v-if="step === 1"
          :is-loading="isCreatingInstance"
          :disabled="v$.$invalid"
          type="submit"
          solid
          blue
          label="Crear Instancia"
        />

        <NextButton
          v-else-if="step === 3"
          :is-loading="isConnectingChatwoot"
          type="submit"
          solid
          blue
          label="Finalizar Configuración"
        />

        <div v-if="step === 2" class="flex gap-2">
          <button
            type="button"
            @click="resetForm"
            class="px-4 py-2 bg-gray-300 text-gray-700 rounded hover:bg-gray-400"
          >
            Volver al inicio
          </button>
          <NextButton
            v-if="connectionEstablished"
            @click="step = 3"
            solid
            blue
            label="Continuar"
          />
        </div>
      </div>
    </form>
  </div>
</template>
