<script>
import * as XLSX from 'xlsx'
import axios from 'axios'
import MockAdapter from 'axios-mock-adapter'
import { CIcon } from '@coreui/icons-vue'
import { cilCheckCircle, cilReload, cilSpreadsheet } from '@coreui/icons'
import {
  CRow,
  CCol,
  CCard,
  CCardHeader,
  CCardBody,
  CTable,
  CTableHead,
  CTableRow,
  CTableHeaderCell,
  CTableBody,
  CTableDataCell,
  CFormInput,
  CButton,
  CPagination,
} from '@coreui/vue'
export default {
  components: {
    CRow,
    CCol,
    CCard,
    CCardHeader,
    CCardBody,
    CTable,
    CTableHead,
    CTableRow,
    CTableHeaderCell,
    CTableBody,
    CTableDataCell,
    CFormInput,
    CButton,
    CPagination,
    CIcon,
  },
  data() {
    // Datos iniciales
    return {
      file: null,
      data: [],
      columns: [],
      currentPage: 1,
      rowsPerPage: 20,
      isProcessingComplete: false,
      buttonState: {
        process: { text: 'Procesar Archivo', icon: cilSpreadsheet, color: 'info', disabled: true },
        restart: { text: 'Reiniciar', icon: cilReload, color: 'warning', disabled: true },
      },
    }
  },
  computed: {
    // Logica simple para la paginación
    totalPages() {
      return Math.ceil(this.data.length / this.rowsPerPage) // Calculo de limites de paginación
    },
    paginatedData() {
      const start = (this.currentPage - 1) * this.rowsPerPage
      const end = this.currentPage * this.rowsPerPage
      return this.data.slice(start, end)
    },
  },
  methods: {
    // Obtención de archivo
    handleFileUpload(event) {
      const uploadedFile = event.target.files[0]
      if (uploadedFile && uploadedFile.name.endsWith('.xlsx')) {
        this.file = uploadedFile // Se obtiene el archivo
        this.buttonState.process.disabled = false
      } else {
        alert('Por favor, suba un archivo Excel válido.')
      }
    },
    getCellAlignmentStyle(value) {
      return {
        textAlign: typeof value === 'number' ? 'right' : 'center',
      }
    },
    processFile() {
      if (!this.file) return
      this.buttonState.process = {
        text: 'Procesando...',
        icon: cilCheckCircle,
        color: 'info',
        disabled: true,
      }
      const reader = new FileReader()
      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result)
        const workbook = XLSX.read(data, { type: 'array' })
        const firstSheetName =
          workbook.SheetNames['ExportData'] || workbook.SheetNames[0] || 'Sheet1' // Hoja a extraer
        const worksheet = workbook.Sheets[firstSheetName]
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { defval: null })
        console.log('Datos extraídos del archivo Excel:', jsonData)
        // Columnas que se necesitan
        const filteredData = jsonData.map((row) => {
          return {
            //Columna - Data
            Periodo: row['Periodo'],
            ID: row['ID'],
            Codigo: row['Codigo'],
            Fecha: this.serialToDate(row['Fecha']) || row['Fecha'],
            'Accion id': row['Accion id'],
            Comando: row['Comando'],
            'Numero de registro': row['Numero de registro'],
            Monto: row['Monto'],
            Moneda: row['Moneda'],
            'Tipo de cambio': row['Tipo de cambio'],
          }
        })
        this.data = filteredData // Datos procesados
        this.columns = [
          'Periodo',
          'ID',
          'Codigo',
          'Fecha',
          'Accion id',
          'Comando',
          'Numero de registro',
          'Monto',
          'Moneda',
          'Tipo de cambio',
        ]
        this.isProcessingComplete = true // Estado de procesamiento
        this.buttonState.process = {
          text: 'Archivo Procesado',
          icon: cilCheckCircle,
          color: 'success',
          disabled: true,
        }
        this.buttonState.restart.disabled = false
      }
      reader.readAsArrayBuffer(this.file)
    },
    serialToDate(serial) {
      if (serial === null || serial === undefined || serial === '') {
        return '-'
      }
      const date = new Date(Math.round((serial - 25569) * 86400 * 1000));
      return date.toLocaleDateString(); // Formateo de fecha
    },
    sendData() {
      // Instancia de Axios Mock Adapter
      const mock = new MockAdapter(axios)

      // Simular una respuesta exitosa (200 OK) después de 1 segundo
      mock.onPost('https://pruebaSubida/Excel').reply(200, {})

      // Validamos la data
      const modifiedData = this.data.map(obj => {
        // Recorrer cada propiedad del objeto
        Object.keys(obj).forEach(key => {
          // Verificar si el valor es '-' o está vacío, y cambiarlo a null
          if (obj[key] === '-' || obj[key] === '') {
            obj[key] = null;
          }
        });
        return obj;
      });
      
      // Simulación del envío de datos usando axios
      axios
        .post('https://pruebaSubida/Excel', this.data)
        .then((response) => {
          console.log('Datos enviados correctamente:', response.data)
          this.downloadJsonFile(this.data) // Llamada a la función para crear y descargar el archivo JSON
        })
        .catch((error) => {
          console.error('Error al enviar los datos:', error)
        })
        .finally(() => {
          // Eliminar la instancia de Axios Mock Adapter y limpiar el mock
          mock.restore()
        })
    },
    downloadJsonFile(data) {
      const jsonData = JSON.stringify(data, null, 2) // Formato de datos JSON
      const blob = new Blob([jsonData], { type: 'application/json' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = 'datos.json'
      a.click()
      URL.revokeObjectURL(url)
      console.log('Archivo JSON creado y descargado.')
    },
    restartProcess() {
      this.file = null
      this.data = []
      this.columns = []
      this.currentPage = 1
      this.isProcessingComplete = false
      this.buttonState.process = {
        text: 'Procesar Archivo',
        icon: cilSpreadsheet,
        color: 'info',
        disabled: true,
      }
      this.buttonState.restart = {
        text: 'Reiniciar',
        icon: cilReload,
        color: 'warning',
        disabled: true,
      }
      const input = document.getElementById('formFileLg')
      if (input) input.value = ''
    },
    updatePage(page) {
      this.currentPage = page // Actualización de la página
    },
  },
}
</script>

<template>
  <div>
    <CRow>
      <CCol :xs="12">
        <CCard class="mb-4">
          <CCardHeader>
            <strong>Formulario</strong>
            <div class="vr h-100 mx-2 text-body text-opacity-75"></div>
            <small>Carga de Archivo</small>
          </CCardHeader>
          <CCardBody>
            <CTable align="middle" class="mb-0 border" hover responsive>
              <CTableHead class="text-nowrap">
                <CTableRow>
                  <CTableHeaderCell class="bg-body-secondary text-center">
                    Subir un archivol Excel
                  </CTableHeaderCell>
                  <CTableHeaderCell class="bg-body-secondary text-center">
                    Acciones
                  </CTableHeaderCell>
                </CTableRow>
              </CTableHead>
              <CTableBody>
                <CTableRow>
                  <CTableDataCell class="text-center">
                    <div>
                      <CFormInput
                        id="formFileLg"
                        type="file"
                        size="lg"
                        accept=".xlsx"
                        @change="handleFileUpload"
                        :disabled="isProcessingComplete"
                      />
                    </div>
                  </CTableDataCell>
                  <CTableDataCell class="text-center">
                    <CButton
                      :color="buttonState.process.color"
                      @click="processFile"
                      :disabled="!file || buttonState.process.disabled || isProcessingComplete"
                    >
                      <CIcon :icon="buttonState.process.icon" class="me-2" />
                      {{ buttonState.process.text }}
                    </CButton>
                    <div class="vr h-100 mx-2 text-body text-opacity-75"></div>
                    <CButton
                      :color="buttonState.restart.color"
                      @click="restartProcess"
                      :disabled="buttonState.restart.disabled"
                    >
                      <CIcon :icon="buttonState.restart.icon" class="me-2" />
                      {{ buttonState.restart.text }}
                    </CButton>
                  </CTableDataCell>
                </CTableRow>
              </CTableBody>
            </CTable>
            <div v-if="data.length">
              <CTable align="middle" class="mt-4 border" hover responsive>
                <CTableHead>
                  <CTableRow>
                    <CTableHeaderCell v-for="col in columns" :key="col">{{ col }}</CTableHeaderCell>
                  </CTableRow>
                </CTableHead>
                <CTableBody>
                  <CTableRow v-for="(row, index) in paginatedData" :key="index">
                    <CTableDataCell
                      v-for="col in columns"
                      :key="col"
                      :style="getCellAlignmentStyle(row[col])"
                    >
                      {{ row[col] }}
                    </CTableDataCell>
                  </CTableRow>
                </CTableBody>
              </CTable>
              <CPagination
                :pages="totalPages"
                :current-page="currentPage"
                @update:currentPage="updatePage"
                align="end"
              />
              <CButton color="success" class="mt-4" @click="sendData"> Enviar Datos </CButton>
            </div>
          </CCardBody>
        </CCard>
      </CCol>
    </CRow>
  </div>
</template>
