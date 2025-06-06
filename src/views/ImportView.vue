<template>
  <div class="import-view">
    <div class="background-container"></div>

    <div class="content-overlay">
      <div class="import-container">
        <h1>Імпорт товарів</h1>
        <p>Виберіть Excel-файл для імпорту товарів</p>

        <div class="form-group">
          <label for="category">Категорія:</label>
          <select v-model="selectedCategory" id="category" class="form-control">
            <option value="">Виберіть категорію</option>
            <option v-for="cat in categories" :key="cat.id" :value="cat.slug">
              {{ cat.name }}
            </option>
          </select>
        </div>

        <div class="form-group">
          <label for="file">Файл Excel (.xlsx):</label>
          <input
            type="file"
            id="file"
            ref="fileInput"
            class="form-control"
            @change="handleFileChange"
            accept=".xlsx"
          />
        </div>

        <div class="buttons">
          <button
            @click="importProducts"
            :disabled="!isFileSelected || !selectedCategory"
            class="import-btn"
          >
            Імпортувати товари
          </button>
          <button @click="goBack" class="cancel-btn">Скасувати</button>
        </div>

        <div v-if="isImporting" class="progress">
          <div class="progress-bar" :style="{ width: importProgress + '%' }">
            {{ importProgress }}%
          </div>
        </div>

        <div v-if="importResults" class="import-results">
          <h2>Результати імпорту:</h2>
          <ul>
            <li>Додано товарів: {{ importResults.added }}</li>
            <li>Оновлено товарів: {{ importResults.updated }}</li>
            <li>Помилки: {{ importResults.errors }}</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useProductStore } from '@/stores'
import ExcelJS from 'exceljs'

export default {
  name: 'ImportView',
  setup() {
    const router = useRouter()
    const productStore = useProductStore()
    const fileInput = ref(null)
    const selectedFile = ref(null)
    const isFileSelected = ref(false)
    const selectedCategory = ref('')
    const importProgress = ref(0)
    const isImporting = ref(false)
    const importResults = ref(null)

    const categories = computed(() => {
      return productStore.categories
    })

    onMounted(async () => {
      if (productStore.categories.length === 0) {
        await productStore.fetchCategories()
      }
    })

    const handleFileChange = (event) => {
      const file = event.target.files[0]
      if (file) {
        selectedFile.value = file
        isFileSelected.value = true
      } else {
        isFileSelected.value = false
      }
    }

    const importProducts = async () => {
      if (!selectedFile.value || !selectedCategory.value) return

      isImporting.value = true
      importProgress.value = 10
      importResults.value = null

      try {
        // Чтение Excel-файла
        const data = await readExcelFile(selectedFile.value)

        importProgress.value = 30

        // Подготовка данных товаров
        const products = processExcelData(data)

        importProgress.value = 60

        // Импорт товаров в хранилище
        const results = await importProductsToStore(products, selectedCategory.value)

        importProgress.value = 100
        importResults.value = results
      } catch (error) {
        console.error('Ошибка импорта:', error)
        importResults.value = {
          added: 0,
          updated: 0,
          errors: 'Помилка при імпорті: ' + error.message,
        }
      } finally {
        isImporting.value = false
      }
    }

    const readExcelFile = (file) => {
      return new Promise((resolve, reject) => {
        const reader = new FileReader()

        reader.onload = async (e) => {
          try {
            const buffer = e.target.result
            const workbook = new ExcelJS.Workbook()
            await workbook.xlsx.load(buffer)

            const worksheet = workbook.worksheets[0]
            const jsonData = []

            worksheet.eachRow({ includeEmpty: false }, (row) => {
              const rowData = []
              row.eachCell((cell) => {
                rowData.push(cell.value)
              })
              jsonData.push(rowData)
            })

            resolve(jsonData)
          } catch (error) {
            reject(error)
          }
        }

        reader.onerror = (error) => {
          reject(error)
        }

        reader.readAsArrayBuffer(file)
      })
    }

    const processExcelData = (data) => {
      // Предполагаем, что первая строка - заголовки
      const headers = data[0]
      const products = []

      // Проверяем, есть ли необходимые колонки
      const nameIndex = headers.findIndex(
        (h) => h.toLowerCase().includes('назва') || h.toLowerCase().includes('name'),
      )
      const codeIndex = headers.findIndex(
        (h) => h.toLowerCase().includes('код') || h.toLowerCase().includes('code'),
      )
      const priceIndex = headers.findIndex(
        (h) => h.toLowerCase().includes('ціна') || h.toLowerCase().includes('price'),
      )
      const imageIndex = headers.findIndex(
        (h) => h.toLowerCase().includes('зображення') || h.toLowerCase().includes('image'),
      )

      if (nameIndex === -1) {
        throw new Error('Колонка з назвою товару не знайдена')
      }

      // Обрабатываем строки данных, начиная со второй строки
      for (let i = 1; i < data.length; i++) {
        const row = data[i]
        if (row.length === 0 || !row[nameIndex]) continue

        const product = {
          id: Date.now() + i, // Временный ID
          name: row[nameIndex],
          code: codeIndex > -1 ? row[codeIndex] : '',
          price: priceIndex > -1 ? row[priceIndex] : 0,
          image: imageIndex > -1 ? row[imageIndex] : '/images/goods/default.webp',
        }

        products.push(product)
      }

      return products
    }

    const importProductsToStore = async (products, categorySlug) => {
      let added = 0
      let updated = 0
      let errors = 0

      try {
        // Здесь должна быть логика добавления товаров в хранилище
        // Для примера просто добавляем товары в хранилище
        await productStore.addBulkProducts(products, categorySlug)
        added = products.length

        return { added, updated, errors }
      } catch (error) {
        console.error('Ошибка добавления товаров:', error)
        errors = products.length
        return { added, updated, errors }
      }
    }

    const goBack = () => {
      router.back()
    }

    return {
      fileInput,
      selectedCategory,
      categories,
      isFileSelected,
      importProgress,
      isImporting,
      importResults,
      handleFileChange,
      importProducts,
      goBack,
    }
  },
}
</script>

<style scoped>
.import-view {
  position: fixed;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  margin: 0;
  padding: 0;
  top: 0;
  left: 0;
}

.background-container {
  position: fixed;
  width: 100%;
  height: 100%;
  background-image: url('/images/bg-image4.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: -1;
}

.content-overlay {
  position: relative;
  width: 100%;
  height: 100%;
  z-index: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.import-container {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  padding: 30px;
  max-width: 600px;
  width: 100%;
}

h1 {
  margin-top: 0;
  margin-bottom: 20px;
  color: #333;
}

.form-group {
  margin-bottom: 20px;
}

label {
  display: block;
  margin-bottom: 5px;
  font-weight: 500;
}

.form-control {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #ced4da;
  border-radius: 4px;
  font-size: 16px;
}

.buttons {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.import-btn,
.cancel-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.2s;
}

.import-btn {
  background-color: #28a745;
  color: white;
}

.import-btn:hover {
  background-color: #218838;
}

.import-btn:disabled {
  background-color: #94d3a2;
  cursor: not-allowed;
}

.cancel-btn {
  background-color: #6c757d;
  color: white;
}

.cancel-btn:hover {
  background-color: #5a6268;
}

.progress {
  margin-top: 20px;
  height: 20px;
  background-color: #e9ecef;
  border-radius: 4px;
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  background-color: #28a745;
  text-align: center;
  color: white;
  line-height: 20px;
  transition: width 0.3s;
}

.import-results {
  margin-top: 20px;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 4px;
  border-left: 4px solid #28a745;
}

.import-results h2 {
  margin-top: 0;
  font-size: 18px;
  color: #333;
}

.import-results ul {
  padding-left: 20px;
  margin-bottom: 0;
}
</style>
