<script>
import { useUserStore } from '@/stores'
import { useCartStore } from '@/stores'
import { onMounted, ref, computed } from 'vue'
import { useRouter } from 'vue-router'

export default {
  name: 'ProfileView',
  setup() {
    const userStore = useUserStore()
    const cartStore = useCartStore()
    const router = useRouter()

    const user = computed(() => userStore.user)
    const cartItems = computed(() => cartStore.items)
    const totalCartPrice = computed(() => cartStore.totalPrice)

    const isUpdating = ref(false)
    const updateSuccess = ref(false)
    const error = ref(null)
    const showPassword = ref(false)
    const showNewPassword = ref(false)

    // Заказы пользователя
    const orders = ref([
      {
        id: '1.200525',
        order_number: '1.200525',
        created_at: '2023-05-20 20:26',
        status: 'Скасовано',
        items: [
          {
            product: { code: '5662000005M6X0A020', name: 'DIN933 Болт M10x100 5.8 цб пр' },
            quantity: 3,
            price: 5.0,
            get_cost: 15.0,
          },
        ],
        get_total_cost: 15.0,
      },
      {
        id: '9.090525',
        order_number: '9.090525',
        created_at: '2023-05-09 15:41',
        status: 'Скасовано',
        items: [
          {
            product: { code: '5M62000005M6X01020', name: 'DIN933 Болт M10x10 8.8 цб пр' },
            quantity: 10,
            price: 3.0,
            get_cost: 30.0,
          },
        ],
        get_total_cost: 30.0,
      },
      {
        id: '7.090525',
        order_number: '7.090525',
        created_at: '2023-05-09 15:29',
        status: 'Скасовано',
        items: [
          {
            product: { code: '5N60000005N6805000', name: 'DIN931 Болт M8x50 10.9 БП пр' },
            quantity: 5,
            price: 5.0,
            get_cost: 25.0,
          },
        ],
        get_total_cost: 25.0,
      },
    ])

    const form = ref({
      first_name: '',
      last_name: '',
      username: '',
      email: '',
      phone_number: '',
      old_password: '',
      password1: '',
    })

    const togglePasswordVisibility = (field) => {
      if (field === 'old_password') {
        showPassword.value = !showPassword.value
      } else if (field === 'password1') {
        showNewPassword.value = !showNewPassword.value
      }
    }

    const updateQuantity = (productId, newQuantity) => {
      if (newQuantity > 0) {
        cartStore.updateQuantity(productId, newQuantity)
      }
    }

    const removeFromCart = (productId) => {
      cartStore.removeFromCart(productId)
    }

    onMounted(() => {
      // Загружаем корзину
      cartStore.loadCart()

      // Проверяем авторизацию
      if (userStore.isAuthenticated) {
        // Заполняем форму только теми данными пользователя, которые не пустые
        form.value = {
          first_name: user.value?.firstName || '',
          last_name: user.value?.lastName || '',
          username: user.value?.username || '',
          email: user.value?.email || '',
          phone_number: user.value?.phone || '',
          old_password: '',
          password1: '',
        }

        // Удаляем пустые значения
        Object.keys(form.value).forEach((key) => {
          if (form.value[key] === '') {
            form.value[key] = ''
          }
        })
      } else {
        // Перенаправляем на страницу входа
        router.push('/login')
      }
    })

    const saveProfile = async () => {
      try {
        isUpdating.value = true

        // Если выбран новый аватар, загружаем его на GitHub
        if (selectedFile.value) {
          const fileName = `${user.value.username}-${Date.now()}.${selectedFile.value.name.split('.').pop()}`

          // Конвертируем файл в base64
          const reader = new FileReader()
          const base64File = await new Promise((resolve) => {
            reader.onload = () => {
              const base64 = reader.result.split(',')[1]
              resolve(base64)
            }
            reader.readAsDataURL(selectedFile.value)
          })

          // Загружаем файл на GitHub
          try {
            const response = await fetch(
              'https://api.github.com/repos/maks-1008/fixstar-avatars/contents/avatars/' + fileName,
              {
                method: 'PUT',
                headers: {
                  Authorization:
                    'token ' + (import.meta.env.VITE_GITHUB_TOKEN || 'YOUR_TOKEN_HERE'),
                  'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                  message: `Upload avatar for ${user.value.username}`,
                  content: base64File,
                }),
              },
            )

            if (!response.ok) {
              const errorData = await response.json()
              throw new Error(errorData.message || 'Помилка завантаження аватару')
            }

            // Сохраняем URL аватара
            const avatarUrl = `https://raw.githubusercontent.com/maks-1008/fixstar-avatars/main/avatars/${fileName}`

            // Обновляем данные в хранилище
            userStore.user = {
              ...userStore.user,
              firstName: form.value.first_name,
              lastName: form.value.last_name,
              email: form.value.email,
              phone: form.value.phone_number,
              avatar: avatarUrl,
            }

            // Обновляем localStorage
            localStorage.setItem('user', JSON.stringify(userStore.user))

            updateSuccess.value = true
            selectedFile.value = null
          } catch (err) {
            error.value = 'Помилка завантаження аватару: ' + err.message
            isUpdating.value = false
            return
          }
        } else {
          // Если аватар не менялся, просто обновляем остальные данные
          userStore.user = {
            ...userStore.user,
            firstName: form.value.first_name,
            lastName: form.value.last_name,
            email: form.value.email,
            phone: form.value.phone_number,
          }

          // Обновляем localStorage
          localStorage.setItem('user', JSON.stringify(userStore.user))

          updateSuccess.value = true
        }

        isUpdating.value = false

        // Скрываем сообщение об успехе через 3 секунды
        setTimeout(() => {
          updateSuccess.value = false
        }, 3000)
      } catch (err) {
        error.value = err.message || 'Помилка оновлення профілю'
        isUpdating.value = false
      }
    }

    const logout = async () => {
      await userStore.logout()
      router.push('/login')
    }

    const avatarPreview = ref(null)
    const selectedFile = ref(null)

    const handleAvatarChange = (event) => {
      const file = event.target.files[0]
      if (file) {
        const reader = new FileReader()
        reader.onload = (e) => {
          avatarPreview.value = e.target.result
        }
        reader.readAsDataURL(file)
        selectedFile.value = file
      }
    }

    return {
      user,
      cartItems,
      totalCartPrice,
      orders,
      form,
      isUpdating,
      updateSuccess,
      error,
      showPassword,
      showNewPassword,
      saveProfile,
      logout,
      togglePasswordVisibility,
      updateQuantity,
      removeFromCart,
      avatarPreview,
      selectedFile,
      handleAvatarChange,
    }
  },
}
</script>

<template>
  <div class="main-container">
    <div class="content-area">
      <div class="container-fluid">
        <div class="row justify-content-center g-0">
          <!-- Профиль с данными пользователя -->
          <div class="col-md-4">
            <div class="bg-white p-4 mb-4 me-2 rounded custom-shadow">
              <h3 class="text-center mb-4">Профіль користувача</h3>
              <form @submit.prevent="saveProfile" autocomplete="off">
                <div class="row">
                  <div class="col-md-12 mb-3 text-center">
                    <img
                      :src="avatarPreview || '/images/baseavatar.jpg'"
                      alt="Аватар користувача"
                      class="img-fluid rounded-circle mb-2"
                      style="max-width: 150px"
                    />
                    <div class="avatar-upload">
                      <label
                        for="avatar-input"
                        class="btn btn-outline-dark btn-sm d-block mx-auto"
                        style="max-width: 150px"
                      >
                        Вибрати файл
                      </label>
                      <input
                        type="file"
                        id="avatar-input"
                        class="d-none"
                        accept="image/*"
                        @change="handleAvatarChange"
                      />
                      <div v-if="selectedFile" class="selected-file mt-2 text-muted small">
                        {{ selectedFile.name }}
                      </div>
                    </div>
                  </div>
                  <div class="col-md-12 mb-3">
                    <label for="id_first_name" class="form-label">Ім'я*</label>
                    <input
                      type="text"
                      class="form-control"
                      id="id_first_name"
                      v-model="form.first_name"
                      placeholder="Введіть ваше ім'я"
                      required
                    />
                  </div>
                  <div class="col-md-12 mb-3">
                    <label for="id_last_name" class="form-label">Прізвище*</label>
                    <input
                      type="text"
                      class="form-control"
                      id="id_last_name"
                      v-model="form.last_name"
                      placeholder="Введіть ваше прізвище"
                      required
                    />
                  </div>
                  <div class="col-md-12 mb-3">
                    <label for="id_username" class="form-label">Логін*</label>
                    <input
                      type="text"
                      class="form-control"
                      id="id_username"
                      v-model="form.username"
                      placeholder="Введіть ваше ім'я користувача"
                      required
                    />
                  </div>
                  <div class="col-md-12 mb-3">
                    <label for="id_email" class="form-label">Email*</label>
                    <input
                      type="email"
                      class="form-control"
                      id="id_email"
                      v-model="form.email"
                      placeholder="Введіть ваш email *youremail@example.com"
                      required
                    />
                  </div>
                  <div class="col-md-12 mb-3">
                    <label for="id_phone_number" class="form-label">Номер телефону</label>
                    <input
                      type="tel"
                      class="form-control"
                      id="id_phone_number"
                      v-model="form.phone_number"
                      placeholder="Введіть ваш номер телефону"
                    />
                  </div>
                  <div class="col-md-12 mb-2">
                    <h5 class="mb-1">Зміна паролю</h5>
                    <p class="text-muted small password-info">
                      Залиште поля порожніми, якщо не бажаєте змінювати пароль
                    </p>
                  </div>
                  <div class="col-md-12 mb-3">
                    <div class="password-field">
                      <input
                        :type="showPassword ? 'text' : 'password'"
                        class="form-control"
                        id="id_old_password"
                        v-model="form.old_password"
                        placeholder="Введіть поточний пароль"
                        autocomplete="new-password"
                      />
                      <button
                        type="button"
                        class="password-toggle"
                        @click="togglePasswordVisibility('old_password')"
                      >
                        <i class="bi" :class="showPassword ? 'bi-eye-slash' : 'bi-eye'"></i>
                      </button>
                    </div>
                  </div>
                  <div class="col-md-12 mb-3">
                    <div class="password-field">
                      <input
                        :type="showNewPassword ? 'text' : 'password'"
                        class="form-control"
                        id="id_password1"
                        v-model="form.password1"
                        placeholder="Введіть новий пароль"
                      />
                      <button
                        type="button"
                        class="password-toggle"
                        @click="togglePasswordVisibility('password1')"
                      >
                        <i class="bi" :class="showNewPassword ? 'bi-eye-slash' : 'bi-eye'"></i>
                      </button>
                    </div>
                  </div>
                </div>
                <button type="submit" class="btn btn-dark" :disabled="isUpdating">
                  <span v-if="isUpdating" class="spinner-border spinner-border-sm me-2"></span>
                  Зберегти
                </button>

                <div v-if="updateSuccess" class="alert alert-success mt-3">
                  Профіль успішно оновлено!
                </div>

                <div v-if="error" class="alert alert-danger mt-3">
                  {{ error }}
                </div>
              </form>
            </div>
          </div>

          <!-- Корзина -->
          <div class="col-md-6">
            <div class="bg-white p-4 mb-4 ms-2 rounded custom-shadow">
              <h3 class="text-center mb-4">Кошик</h3>
              <div class="container px-0" id="cart-items-container">
                <div v-if="cartItems.length === 0" class="alert alert-info">Ваш кошик порожній</div>
                <div v-else class="table-responsive">
                  <table class="product-table table table-striped table-hover mb-3">
                    <thead class="table-light">
                      <tr>
                        <th style="width: 40%">АРТИКУЛ, НАЗВА ТОВАРУ</th>
                        <th style="width: 15%">КІЛЬКІСТЬ</th>
                        <th style="width: 20%">ЦІНА</th>
                        <th style="width: 25%">СУМА</th>
                        <th></th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr v-for="item in cartItems" :key="item.id">
                        <td>
                          <div class="product-info">
                            <span style="font-weight: 500">{{ item.id }}</span
                            ><br />
                            <span style="color: #555">{{ item.name }}</span>
                          </div>
                        </td>
                        <td>
                          <div class="input-group">
                            <button
                              class="btn btn-cart-quantity"
                              @click="updateQuantity(item.id, item.quantity - 1)"
                            >
                              -
                            </button>
                            <input
                              type="number"
                              class="form-control text-center cart-quantity-input"
                              v-model.number="item.quantity"
                              min="1"
                              @change="updateQuantity(item.id, item.quantity)"
                            />
                            <button
                              class="btn btn-cart-quantity"
                              @click="updateQuantity(item.id, item.quantity + 1)"
                            >
                              +
                            </button>
                          </div>
                        </td>
                        <td style="font-weight: 500">{{ item.price.toFixed(2) }} грн</td>
                        <td style="font-weight: 500">
                          {{ (item.price * item.quantity).toFixed(2) }} грн
                        </td>
                        <td>
                          <button class="btn btn-sm btn-danger" @click="removeFromCart(item.id)">
                            <i class="bi bi-trash"></i>
                          </button>
                        </td>
                      </tr>
                    </tbody>
                  </table>

                  <div class="card shadow-lg" v-if="cartItems.length > 0">
                    <div class="card-footer">
                      <p class="float-left">
                        Всього позицій товарів <strong>{{ cartItems.length }}</strong> на суму
                      </p>
                      <h4 class="float-left">
                        <strong>{{ totalCartPrice.toFixed(2) }} грн</strong>
                      </h4>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Оформленные заказы -->
          <div class="col-md-10">
            <div class="bg-white p-4 mb-4 rounded custom-shadow" style="margin-left: 0px">
              <h3 class="text-center mb-4">Мої замовлення</h3>
              <div class="container">
                <div class="accordion" id="accordionExample">
                  <div v-for="(order, index) in orders" :key="order.id" class="accordion-item">
                    <h2 class="accordion-header" :id="'heading' + order.id">
                      <button
                        class="accordion-button"
                        :class="{ collapsed: index !== 0 }"
                        type="button"
                        data-bs-toggle="collapse"
                        :data-bs-target="'#collapse' + order.id"
                        :aria-expanded="index === 0 ? 'true' : 'false'"
                        :aria-controls="'collapse' + order.id"
                      >
                        Замовлення № {{ order.order_number }} - {{ order.created_at }} | Статус:
                        <strong class="mx-2">{{ order.status }}</strong>
                      </button>
                    </h2>
                    <div
                      :id="'collapse' + order.id"
                      class="accordion-collapse collapse"
                      :class="{ show: index === 0 }"
                      :aria-labelledby="'heading' + order.id"
                      data-bs-parent="#accordionExample"
                    >
                      <div class="accordion-body">
                        <div class="table-responsive" style="max-width: 100%; overflow-x: auto">
                          <table
                            class="product-table table table-striped table-hover mb-3"
                            style="width: 100%"
                          >
                            <thead class="table-light">
                              <tr>
                                <th style="width: 40%">АРТИКУЛ, НАЗВА ТОВАРУ</th>
                                <th style="width: 15%">КІЛЬКІСТЬ</th>
                                <th style="width: 20%">ЦІНА</th>
                                <th style="width: 25%">СУМА</th>
                              </tr>
                            </thead>
                            <tbody>
                              <tr v-for="item in order.items" :key="item.product.code">
                                <td>
                                  <div class="product-info">
                                    <span style="font-weight: 500">{{ item.product.code }}</span
                                    ><br />
                                    <span style="color: #555">{{ item.product.name }}</span>
                                  </div>
                                </td>
                                <td style="text-align: center; font-weight: 500">
                                  {{ item.quantity }}
                                </td>
                                <td style="font-weight: 500">{{ item.price.toFixed(2) }} грн</td>
                                <td style="font-weight: 500">{{ item.get_cost.toFixed(2) }} грн</td>
                              </tr>
                            </tbody>
                          </table>
                        </div>
                        <div class="card shadow-lg">
                          <div class="card-footer">
                            <p class="float-left">
                              Всього позицій товарів<strong> {{ order.items.length }}</strong> на
                              суму
                            </p>
                            <h4 class="float-left">
                              <strong>{{ order.get_total_cost.toFixed(2) }} грн</strong>
                            </h4>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Стили для страницы профиля */
.main-container {
  position: relative;
  min-height: 100vh;
  padding-top: 80px;
}

/* Уменьшаем отступы между контейнерами */
.row > [class*='col-'] {
  padding-right: 5px;
  padding-left: 5px;
}

/* Фон как на главной странице */
.main-container::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url('/images/bg-image4.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  z-index: -1;
}

/* Фиксированные кнопки слева */
:deep(.btn-group-vertical) {
  position: fixed !important;
  left: 20px;
  top: 80px;
  z-index: 1000;
}

/* Основное содержимое */
.content-area {
  padding: 20px;
  margin-left: 200px;
  width: calc(100% - 140px);
}

.custom-shadow {
  box-shadow: 5px 5px 25px 19px rgba(0, 0, 0, 0.5);
  border-radius: 8px;
  background-color: white;
  margin-bottom: 20px;
  padding: 20px;
}

.password-field {
  position: relative;
}

.password-toggle {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  color: #6c757d;
  cursor: pointer;
  z-index: 10;
  padding: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.password-info {
  margin-bottom: 5px;
  font-size: 12px;
}

/* Стили для кнопок корзины */
.product-table th,
.product-table td {
  text-align: center;
  vertical-align: middle;
}

.product-table .product-info {
  text-align: left;
}

.btn-cart-quantity {
  width: 38px;
  background-color: #ffa023;
  border-color: #ffa023;
  color: white;
}

.btn-cart-quantity:hover {
  background-color: #e89020;
  border-color: #e89020;
  color: white;
}

.cart-quantity-input {
  border-radius: 4px;
  text-align: center;
  transition: all 0.2s ease;
}

.cart-quantity-input:focus {
  border-color: #80bdff;
  box-shadow: 0 0 0 0.2rem rgba(0, 123, 255, 0.25);
  outline: none;
}

/* Медиа-запросы для мобильных устройств */
@media (max-width: 768px) {
  .main-container::before {
    background-image: url('/images/bg-image5.jpg');
  }

  .content-area {
    margin-left: 120px;
    width: calc(100% - 120px);
  }

  .custom-shadow {
    margin-top: 10px;
  }
}
</style>
