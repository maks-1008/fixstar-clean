<template>
  <div class="admin-view">
    <!-- Верхняя панель админки, как в Django -->
    <div class="admin-header">
      <div class="admin-title">Адміністрування сайту</div>
      <div class="admin-user-actions">
        <span class="welcome">ВІТАЄМО, {{ userName }}</span>
        <a href="/" class="admin-link">ДИВИТИСЯ САЙТ</a> /
        <a @click="changePassword" class="admin-link">ЗМІНИТИ ПАРОЛЬ</a> /
        <a @click="logout" class="admin-link">ВИЙТИ</a>
        <button class="theme-toggle" @click="toggleTheme">
          <i class="bi bi-moon"></i>
        </button>
      </div>
    </div>

    <div class="content-overlay">
      <div class="admin-container">
        <div class="admin-navigation">
          <div
            v-for="section in sections"
            :key="section.id"
            :class="['nav-item', { active: activeSection === section.id }]"
            @click="activeSection = section.id"
          >
            {{ section.name }}
          </div>
        </div>

        <div class="admin-content">
          <!-- Товары -->
          <div v-if="activeSection === 'products'" class="section-content">
            <h2>Управління товарами</h2>

            <div class="admin-actions">
              <button @click="goToImport" class="action-btn import-btn">Імпортувати товари</button>
              <button @click="addProduct" class="action-btn">Додати товар</button>
            </div>

            <div class="search-bar">
              <input
                type="text"
                v-model="productSearch"
                placeholder="Пошук товарів..."
                class="search-input"
              />
            </div>

            <table class="data-table">
              <thead>
                <tr>
                  <th>ID</th>
                  <th>Назва</th>
                  <th>Код</th>
                  <th>Ціна</th>
                  <th>Категорія</th>
                  <th>Дії</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="product in filteredProducts" :key="product.id">
                  <td>{{ product.id }}</td>
                  <td>{{ product.name }}</td>
                  <td>{{ product.code || '-' }}</td>
                  <td>{{ product.price }} грн.</td>
                  <td>{{ getCategoryName(product.categoryId) }}</td>
                  <td class="actions">
                    <button @click="editProduct(product)" class="edit-btn">
                      <i class="bi bi-pencil"></i>
                    </button>
                    <button @click="deleteProduct(product.id)" class="delete-btn">
                      <i class="bi bi-trash"></i>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Категории -->
          <div v-if="activeSection === 'categories'" class="section-content">
            <h2>Управління категоріями</h2>

            <div class="admin-actions">
              <button @click="addCategory" class="action-btn">Додати категорію</button>
            </div>

            <table class="data-table">
              <thead>
                <tr>
                  <th>ID</th>
                  <th>Назва</th>
                  <th>Slug</th>
                  <th>Дії</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="category in categories" :key="category.id">
                  <td>{{ category.id }}</td>
                  <td>{{ category.name }}</td>
                  <td>{{ category.slug }}</td>
                  <td class="actions">
                    <button @click="editCategory(category)" class="edit-btn">
                      <i class="bi bi-pencil"></i>
                    </button>
                    <button @click="deleteCategory(category.id)" class="delete-btn">
                      <i class="bi bi-trash"></i>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Заказы -->
          <div v-if="activeSection === 'orders'" class="section-content">
            <h2>Управління замовленнями</h2>

            <table class="data-table">
              <thead>
                <tr>
                  <th>ID</th>
                  <th>Дата</th>
                  <th>Сума</th>
                  <th>Статус</th>
                  <th>Дії</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="order in orders" :key="order.id">
                  <td>{{ order.id }}</td>
                  <td>{{ formatDate(order.created_at) }}</td>
                  <td>{{ order.total }} грн.</td>
                  <td>
                    <select v-model="order.status" @change="updateOrderStatus(order)">
                      <option
                        v-for="status in orderStatuses"
                        :key="status.value"
                        :value="status.value"
                      >
                        {{ status.label }}
                      </option>
                    </select>
                  </td>
                  <td class="actions">
                    <button @click="viewOrder(order)" class="view-btn">
                      <i class="bi bi-eye"></i>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useProductStore, useOrderStore, useUserStore } from '@/stores'

export default {
  name: 'AdminView',
  setup() {
    const router = useRouter()
    const productStore = useProductStore()
    const orderStore = useOrderStore()
    const userStore = useUserStore()

    const activeSection = ref('products')
    const productSearch = ref('')
    const darkTheme = ref(false)

    const sections = [
      { id: 'products', name: 'Товари' },
      { id: 'categories', name: 'Категорії' },
      { id: 'orders', name: 'Замовлення' },
    ]

    const orderStatuses = [
      { value: 'CREATED', label: 'Створено' },
      { value: 'PROCESSING', label: 'В обробці' },
      { value: 'PAID', label: 'Оплачено' },
      { value: 'SHIPPED', label: 'Відправлено' },
      { value: 'DELIVERED', label: 'Доставлено' },
      { value: 'CANCELED', label: 'Скасовано' },
    ]

    // Имя пользователя
    const userName = computed(() => {
      return userStore.user?.username || 'АДМІН'
    })

    // Загружаем данные при монтировании компонента
    onMounted(async () => {
      if (productStore.categories.length === 0) {
        await productStore.fetchCategories()
      }

      if (productStore.products.length === 0) {
        await productStore.fetchProducts()
      }

      await orderStore.fetchOrders()

      // Загружаем товары из localStorage (для демо)
      loadLocalStorageProducts()
    })

    // Загрузка товаров из localStorage (для демо)
    const localProducts = ref([])

    const loadLocalStorageProducts = () => {
      const allProducts = []

      // Для каждой категории проверяем наличие товаров в localStorage
      productStore.categories.forEach((category) => {
        const storedProducts = JSON.parse(localStorage.getItem(`products_${category.slug}`) || '[]')
        allProducts.push(...storedProducts)
      })

      localProducts.value = allProducts
    }

    // Все товары (из хранилища + из localStorage)
    const allProducts = computed(() => {
      return [...productStore.products, ...localProducts.value]
    })

    // Отфильтрованные товары по поиску
    const filteredProducts = computed(() => {
      if (!productSearch.value) return allProducts.value

      const searchTerm = productSearch.value.toLowerCase()
      return allProducts.value.filter(
        (product) =>
          product.name.toLowerCase().includes(searchTerm) ||
          (product.code && product.code.toLowerCase().includes(searchTerm)),
      )
    })

    // Получение имени категории по ID
    const getCategoryName = (categoryId) => {
      const category = productStore.categories.find((cat) => cat.id === categoryId)
      return category ? category.name : 'Не вказано'
    }

    // Форматирование даты
    const formatDate = (dateString) => {
      const date = new Date(dateString)
      return new Intl.DateTimeFormat('uk-UA', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
      }).format(date)
    }

    // Методы для работы с товарами
    const goToImport = () => {
      router.push('/import')
    }

    const addProduct = () => {
      // TODO: Реализовать добавление товара
      alert('Функція додавання товару буде доступна пізніше')
    }

    const editProduct = (product) => {
      // TODO: Реализовать редактирование товара
      alert(`Редагування товару ID: ${product.id} буде доступне пізніше`)
    }

    const deleteProduct = (productId) => {
      if (confirm('Ви дійсно бажаєте видалити цей товар?')) {
        // Удаление товара (для демо просто удаляем из временного массива)
        const index = localProducts.value.findIndex((p) => p.id === productId)
        if (index !== -1) {
          localProducts.value.splice(index, 1)
          alert('Товар видалено')
        }
      }
    }

    // Методы для работы с категориями
    const addCategory = () => {
      // TODO: Реализовать добавление категории
      alert('Функція додавання категорії буде доступна пізніше')
    }

    const editCategory = (category) => {
      // TODO: Реализовать редактирование категории
      alert(`Редагування категорії ID: ${category.id} буде доступне пізніше`)
    }

    const deleteCategory = (categoryId) => {
      if (confirm('Ви дійсно бажаєте видалити цю категорію?')) {
        // TODO: Реализовать удаление категории с использованием categoryId
        console.log('Видалення категорії:', categoryId)
        alert('Функція видалення категорії буде доступна пізніше')
      }
    }

    // Методы для работы с заказами
    const viewOrder = (order) => {
      router.push(`/order/${order.id}`)
    }

    const updateOrderStatus = (order) => {
      // TODO: Реализовать обновление статуса заказа
      alert(`Статус замовлення ID: ${order.id} змінено на ${order.status}`)
    }

    // Методы для пользователя
    const logout = () => {
      userStore.logout()
      router.push('/login')
    }

    const changePassword = () => {
      alert('Функція зміни пароля буде доступна пізніше')
    }

    const toggleTheme = () => {
      darkTheme.value = !darkTheme.value
      document.body.classList.toggle('dark-mode', darkTheme.value)
    }

    return {
      activeSection,
      sections,
      productSearch,
      filteredProducts,
      categories: computed(() => productStore.categories),
      orders: computed(() => orderStore.orders),
      orderStatuses,
      userName,
      getCategoryName,
      formatDate,
      goToImport,
      addProduct,
      editProduct,
      deleteProduct,
      addCategory,
      editCategory,
      deleteCategory,
      viewOrder,
      updateOrderStatus,
      logout,
      changePassword,
      toggleTheme,
    }
  },
}
</script>

<style scoped>
.admin-view {
  position: fixed;
  width: 100%;
  height: 100vh;
  margin: 0;
  padding: 0;
  top: 0;
  left: 0;
  display: flex;
  flex-direction: column;
}

.admin-header {
  background-color: #417690;
  color: white;
  padding: 10px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 50px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  z-index: 100;
}

.admin-title {
  font-size: 18px;
  font-weight: 500;
}

.admin-user-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
}

.admin-link {
  color: white;
  text-decoration: none;
  cursor: pointer;
}

.admin-link:hover {
  text-decoration: underline;
}

.welcome {
  margin-right: 10px;
}

.theme-toggle {
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  font-size: 18px;
  margin-left: 10px;
}

.content-overlay {
  position: relative;
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  background-color: #f8f9fa;
}

.admin-container {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  padding: 30px;
  max-width: 1200px;
  margin: 0 auto;
}

h2 {
  color: #444;
  margin-top: 0;
}

.admin-navigation {
  display: flex;
  margin-bottom: 30px;
  border-bottom: 1px solid #e1e1e1;
}

.nav-item {
  padding: 10px 20px;
  cursor: pointer;
  transition: all 0.2s;
  border-bottom: 2px solid transparent;
  font-weight: 500;
}

.nav-item:hover {
  background-color: #f8f9fa;
}

.nav-item.active {
  border-bottom: 2px solid #417690;
  color: #417690;
}

.section-content {
  margin-top: 20px;
}

.admin-actions {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.action-btn {
  padding: 8px 15px;
  background-color: #417690;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.action-btn:hover {
  background-color: #2d5369;
}

.import-btn {
  background-color: #79aec8;
}

.import-btn:hover {
  background-color: #609ab6;
}

.search-bar {
  margin-bottom: 20px;
}

.search-input {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #ced4da;
  border-radius: 4px;
  font-size: 16px;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 20px;
}

.data-table th,
.data-table td {
  padding: 10px;
  text-align: left;
  border-bottom: 1px solid #e1e1e1;
}

.data-table th {
  background-color: #f8f9fa;
  font-weight: 500;
}

.data-table tr:hover {
  background-color: #f8f9fa;
}

.actions {
  display: flex;
  gap: 5px;
}

.edit-btn,
.delete-btn,
.view-btn {
  padding: 5px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.edit-btn {
  background-color: #ffc107;
}

.edit-btn:hover {
  background-color: #e0a800;
}

.delete-btn {
  background-color: #dc3545;
  color: white;
}

.delete-btn:hover {
  background-color: #c82333;
}

.view-btn {
  background-color: #79aec8;
  color: white;
}

.view-btn:hover {
  background-color: #609ab6;
}

select {
  padding: 5px;
  border: 1px solid #ced4da;
  border-radius: 4px;
}

/* Стили для темной темы */
:global(.dark-mode) .admin-container {
  background-color: #2d3748;
  color: #e2e8f0;
}

:global(.dark-mode) .data-table th {
  background-color: #4a5568;
}

:global(.dark-mode) .data-table th,
:global(.dark-mode) .data-table td {
  border-color: #4a5568;
  color: #e2e8f0;
}

:global(.dark-mode) .data-table tr:hover {
  background-color: #4a5568;
}

:global(.dark-mode) .nav-item {
  color: #e2e8f0;
}

:global(.dark-mode) .nav-item:hover {
  background-color: #4a5568;
}

:global(.dark-mode) .search-input {
  background-color: #2d3748;
  color: #e2e8f0;
  border-color: #4a5568;
}

:global(.dark-mode) select {
  background-color: #2d3748;
  color: #e2e8f0;
  border-color: #4a5568;
}
</style>
