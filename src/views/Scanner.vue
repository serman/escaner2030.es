<template>
  <div>
    <div id="scanner" class="o-container o-section u-margin-bottom-10">
      <tipi-header
        :title="$t('scanner.title')"
        :subtitle="$t('scanner.subtitle')"
      />

      <div class="o-grid u-margin-bottom-4">
        <div class="o-grid__col u-12 u-6@sm">
          <tipi-message type="info" icon>
            <div v-html="$t('scanner.info')"></div>
          </tipi-message>
        </div>

        <div class="o-grid__col u-12 u-6@sm">
          <p>
            <textarea
              :placeholder="$t('scanner.form.placeholder')"
              v-model="inputText"
              rows="9"
            />
          </p>
          <div class="c-input-label c-input-label--file u-block">
            <label for="file">{{ $t('scanner.form.file') }}</label>
            <input
              type="file"
              id="file"
              name="file"
              @change="loadSelectedFile"
              placeholder="PDF, doc o txt"
            />
            <small class="u-color-secondary">{{
              $t('scanner.form.weight')
            }}</small
            ><br />
            <small class="u-color-secondary"
              >pdf, txt, doc, docx, odt, xls, xlsx, ppt, pptx, jpg, png, gif,
              html</small
            >
          </div>
          <p>
            <a
              id="start"
              class="c-button c-button--primary"
              @click.prevent="annotate"
              >{{
                this.inProgress
                  ? $t('scanner.form.buttonProgress')
                  : $t('scanner.form.button')
              }}</a
            >
            <a
              class="c-button"
              :class="{ disabled: inProgress }"
              v-if="hasInput"
              @click="cleanTextAndResult"
              >Limpiar</a
            >
          </p>
        </div>
      </div>

      <div id="result" class="o-section o-grid">
        <div v-if="inProgress || errors" class="o-grid__col u-12">
          <tipi-message v-if="errors" type="error" icon>{{
            errors
          }}</tipi-message>
          <tipi-loader
            v-if="inProgress"
            :title="$t('scanner.result.scanning')"
            :subtitle="subtitle"
          />
        </div>
        <div class="o-grid__col u-12 result" v-if="result">
          <tipi-message v-if="!result.topics.length" type="error" icon>
            {{ $t('scanner.result.notFound') }}
          </tipi-message>

          <div v-else>
            <scanner-visualizations :result="result"></scanner-visualizations>
          </div>

          <!-- Begin CTAs -->
          <div
            class="o-grid o-grid--wide o-grid--center u-bg-primary-light u-padding-top-8 u-padding-bottom-8 u-margin-top-8"
            v-if="result.topics.length"
          >
            <div class="o-grid__col u-12 u-12@xs u-10@sm u-text-center">
              <h5>{{ $t('scanner.save.title') }}</h5>
              <p>
                {{ $t('scanner.save.body') }}
              </p>
              <a @click="saveResult" class="c-button c-button--primary">
                {{ $t('scanner.save.button') }}
              </a>
            </div>
          </div>
          <!-- End CTAs -->
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {
  TipiMessage,
  TipiHeader,
  TipiLoader,
} from '@politicalwatch/tipi-uikit';
import ScannerVisualizations from '@/components/scanner-visualizations.vue';
import Swal from 'sweetalert2';
import VueScrollTo from 'vue-scrollto';
import api from '@/api';

export default {
  name: 'scanner',
  components: {
    TipiHeader,
    TipiMessage,
    TipiLoader,
    ScannerVisualizations,
  },
  data() {
    return {
      inputText: '',
      inputFile: null,
      result: null,
      errors: null,
      inProgress: false,
      estimatedTime: 0,
      csvItems: [],
      csvFields: ['topic', 'subtopic', 'tag', 'times'],
      excerptText: '',
      scanned: {},
      tagsInWordCloud: 25,
    };
  },
  computed: {
    subtitle() {
      return this.estimatedTime
        ? `Tardaremos aproximadamente ${this.estimatedTime} segundos en presentarte los resultados. No te vayas...`
        : 'Ten paciencia. Estamos trabajando en ello.';
    },
    hasInput() {
      return this.inputText != '' || this.inputFile != null;
    },
  },
  methods: {
    cleanText() {
      this.inputText = '';
      this.inputFile = null;
      document.getElementById('file').value = '';
    },
    cleanResult() {
      this.result = null;
      this.errors = null;
    },
    cleanTextAndResult() {
      this.cleanText();
      this.cleanResult();
    },
    loadSelectedFile(event) {
      this.inputFile = event.target.files[0];
    },
    annotate() {
      this.cleanResult();
      this.inProgress = true;
      api
        .annotate(this.inputText, this.inputFile)
        .then((response) => {
          if (response.data.status === 'SUCCESS') {
            this.result = response.data.result;
            this.excerptText = response.data.excerpt;
            this.inProgress = false;
            VueScrollTo.scrollTo('#result', 1500);
          } else if (response.data.status === 'PROCESSING') {
            this.estimatedTime = response.data.estimated_time;
            setTimeout(() => {
              this.getAsyncResults(response.data.task_id);
            }, response.data.estimated_time * 1000);
          }
        })
        .catch((error) => {
          if (error.response.status == 429)
            this.errors =
              'Has sobrepasado el límite de escaneos por hora. Vuelve a intentarlo de aquí a un rato.';
          else if (error.response.status == 413)
            this.errors =
              'Archivo demasiado pesado para procesarlo. Prueba con uno más ligero.';
          else this.errors = error.response.data.message;
          this.inProgress = false;
        });
    },
    saveResult: function () {
      Swal.fire({
        focusConfirm: true,
        showCancelButton: true,
        confirmButtonText: 'Guardar',
        confirmButtonAriaLabel: 'Guardar',
        confirmButtonColor: '#d64949',
        cancelButtonText: 'Cancelar',
        cancelButtonAriaLabel: 'Cancelar',
        html:
          '<label for="scan-name"><h4>Ponle nombre</h4></label>' +
          '<input id="scan-name" type="text"></input>' +
          '<label class="scan-expiration-label" for="scan-expiration"><h4>Se borrará en:</h4></label>' +
          '<select id="scan-expiration">' +
          '<option value="1m" selected>Un mes</option>' +
          '<option value="3m">Tres meses</option>' +
          '<option value="1y">Un año</option>' +
          '</select>',
        preConfirm: () => {
          const name = Swal.getPopup().querySelector('#scan-name').value;
          const expiry =
            Swal.getPopup().querySelector('#scan-expiration').value;
          if (!name || !expiry) {
            Swal.showValidationMessage('Por favor, rellena el formulario');
          }
          return { name, expiry };
        },
      })
        .then(({ value }) => {
          api
            .saveScanned(
              value.name,
              value.expiry,
              this.excerptText,
              this.result
            )
            .then((response) => {
              Swal.fire({
                title: 'Guardado!',
                text: 'Texto escaneado guardado satisfactoriamente',
                html:
                  '<p>Texto escaneado guardado satisfactoriamente</p>' +
                  '<input id="url-to-copy" value="' +
                  window.location.origin +
                  '/scanner/' +
                  response.data.id +
                  '" readonly />',
                focusConfirm: false,
                confirmButtonClass: 'clipboard-button',
                confirmButtonText: 'Copiar enlace y continuar',
                confirmButtonAriaLabel: 'Copiar enlace y continuar',
                confirmButtonColor: '#d64949',
                type: 'success',
                onOpen: () => {
                  const clipboard = new ClipboardJS('.clipboard-button', {
                    target: (trigger) => document.getElementById('url-to-copy'),
                  });
                },
              }).then(
                function () {
                  this.scanned.title = response.data.title;
                  this.scanned.excerpt = response.data.excerpt;
                  this.$router.replace({
                    name: 'scanned',
                    params: {
                      id: response.data.id,
                    },
                  });
                }.bind(this, response.data)
              );
            })
            .catch((error) => {
              const limited = error.response.status === 429;
              Swal.fire({
                title: limited
                  ? 'Límite por hora excedido '
                  : 'Error al guardar el texto escaneado',
                text: 'Intentalo de nuevo más tarde',
                focusConfirm: false,
                type: 'error',
              });
            });
        })
        .catch((error) => {
          this.errors = error;
        });
    },
    getAsyncResults: function (taskID) {
      api
        .getScannerResult(taskID)
        .then((response) => {
          if (response.data.status === 'SUCCESS') {
            this.result = response.data.result;
            this.excerptText = response.data.excerpt;
            this.inProgress = false;
            document.getElementById('start').text =
              this.$i18n.locale === 'es' ? 'Iniciar proceso' : 'Start process';
            VueScrollTo.scrollTo('#result', 1500);
          } else if (response.data.status === 'PENDING') {
            setTimeout(() => {
              this.getAsyncResults(taskID);
            }, 3000);
          }
        })
        .catch(() => {
          setTimeout(() => {
            this.getAsyncResults(taskID);
          }, 3000);
        });
    },
  },
};
</script>

<style lang="scss">
#scan-expiration {
  width: 100%;
  height: 56px;
  background: white;
  border: solid 1px #9cb0bf;
  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
  background-image: url("data:image/svg+xml;utf8,<svg fill='rgb(156,176,191)' height='24' viewBox='0 0 24 24' width='24' xmlns='http://www.w3.org/2000/svg'><path d='M7 10l5 5 5-5z'/><path d='M0 0h24v24H0z' fill='none'/></svg>");
  background-repeat: no-repeat;
  background-position-x: 100%;
  background-position-y: 14px;
  padding: 10px;
}

.scan-expiration-label h4 {
  margin-top: 15px;
}

#url-to-copy {
  height: 53px;
  margin-right: 1px;
  width: 100%;
}
</style>
