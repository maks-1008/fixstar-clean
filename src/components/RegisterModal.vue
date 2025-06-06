<template>
  <div class="register-modal-overlay" v-if="show" @click.self="closeModal">
    <div class="register-modal">
      <div class="register-container">
        <div class="left-side">
          <h1>Реєстрація</h1>
          <div class="input-group">
            <div class="input-wrapper">
              <i class="bi bi-person"></i>
              <input
                type="text"
                placeholder="Логін"
                v-model="userData.username"
                ref="usernameInput"
              />
            </div>
            <div class="input-wrapper">
              <i class="bi bi-envelope"></i>
              <input type="email" placeholder="Електронна пошта" v-model="userData.email" />
            </div>
            <div class="input-wrapper">
              <i class="bi bi-person-vcard"></i>
              <input type="text" placeholder="Ім'я" v-model="userData.firstName" />
            </div>
            <div class="input-wrapper">
              <i class="bi bi-person-vcard"></i>
              <input type="text" placeholder="Прізвище" v-model="userData.lastName" />
            </div>
            <div class="input-wrapper">
              <i class="bi bi-telephone"></i>
              <input type="tel" placeholder="Телефон" v-model="userData.phone" />
            </div>
            <div class="input-wrapper">
              <i class="bi bi-lock"></i>
              <input
                type="password"
                placeholder="Пароль"
                v-model="userData.password"
                ref="passwordInput"
              />
              <i
                class="bi"
                :class="passwordVisible ? 'bi-eye-slash' : 'bi-eye'"
                @click="togglePasswordVisibility"
              ></i>
            </div>
            <div class="input-wrapper">
              <i class="bi bi-lock"></i>
              <input
                type="password"
                placeholder="Повторіть пароль"
                v-model="userData.confirmPassword"
              />
            </div>
          </div>
          <button class="register-button" @click="register" :disabled="isLoading">
            <span v-if="isLoading" class="spinner"></span>
            <span v-else>ЗАРЕЄСТРУВАТИСЯ</span>
          </button>
          <div v-if="error" class="error-message">{{ error }}</div>
          <div class="login-link">
            Вже маєте обліковий запис?
            <a href="#" @click.prevent="goToLogin">Увійти</a>
          </div>
        </div>
        <div class="right-side">
          <h1>ЛАСКАВО ПРОСИМО!</h1>
          <p>
            Дякуємо що зареєструвалися в нашому магазині. Тут ви зможете відстежувати свої
            замовлення та отримувати доступ до спеціальних пропозицій.
          </p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, reactive, watch, nextTick } from 'vue'
import { useUserStore } from '@/stores'
import { useRouter } from 'vue-router'

export default {
  name: 'RegisterModal',
  props: {
    show: {
      type: Boolean,
      required: true,
    },
  },
  emits: ['close', 'login'],
  setup(props, { emit }) {
    const userStore = useUserStore()
    const router = useRouter()

    const userData = reactive({
      username: '',
      email: '',
      firstName: '',
      lastName: '',
      phone: '',
      password: '',
      confirmPassword: '',
    })

    const isLoading = ref(false)
    const error = ref(null)
    const passwordVisible = ref(false)
    const usernameInput = ref(null)
    const passwordInput = ref(null)

    watch(
      () => props.show,
      async (newValue) => {
        if (newValue) {
          error.value = null
          await nextTick()
          usernameInput.value?.focus()
        }
      },
    )

    const togglePasswordVisibility = () => {
      passwordVisible.value = !passwordVisible.value
      if (passwordInput.value) {
        passwordInput.value.type = passwordVisible.value ? 'text' : 'password'
      }
    }

    const validateForm = () => {
      if (
        !userData.username ||
        !userData.email ||
        !userData.firstName ||
        !userData.lastName ||
        !userData.phone ||
        !userData.password ||
        !userData.confirmPassword
      ) {
        error.value = 'Будь ласка, заповніть всі поля'
        return false
      }

      if (userData.password !== userData.confirmPassword) {
        error.value = 'Паролі не співпадають'
        return false
      }

      // Проверка формата email
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
      if (!emailRegex.test(userData.email)) {
        error.value = 'Вкажіть коректну електронну адресу'
        return false
      }

      // Проверка формата телефона
      const phoneRegex = /^\+?[0-9]{10,13}$/
      if (!phoneRegex.test(userData.phone)) {
        error.value = 'Вкажіть коректний номер телефону'
        return false
      }

      return true
    }

    const register = async () => {
      if (!validateForm()) {
        return
      }

      isLoading.value = true
      error.value = null

      try {
        await userStore.register(userData)
        closeModal()

        // Переход на главную страницу или страницу личного кабинета
        router.push('/')
      } catch (err) {
        error.value = err.message || 'Помилка при реєстрації'
        console.error(err)
      } finally {
        isLoading.value = false
      }
    }

    const closeModal = () => {
      emit('close')
    }

    const goToLogin = () => {
      emit('login')
    }

    return {
      userData,
      isLoading,
      error,
      passwordVisible,
      usernameInput,
      passwordInput,
      togglePasswordVisibility,
      register,
      closeModal,
      goToLogin,
    }
  },
}
</script>

<style scoped>
.register-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}

.register-modal {
  width: 100%;
  max-width: 800px;
  background: none;
  border-radius: 10px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
}

.register-container {
  display: flex;
  height: 650px;
}

.left-side {
  flex: 1;
  background-color: #0c1016;
  padding: 40px;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  max-height: 650px;
}

.right-side {
  flex: 1;
  background: linear-gradient(135deg, #0183c6 0%, #00b7ea 100%);
  padding: 40px;
  color: white;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

h1 {
  margin-bottom: 30px;
  font-weight: 600;
}

.left-side h1 {
  color: white;
}

.input-group {
  margin-bottom: 30px;
}

.input-wrapper {
  position: relative;
  margin-bottom: 20px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
}

.input-wrapper i {
  position: absolute;
  left: 0;
  top: 10px;
  color: rgba(255, 255, 255, 0.5);
  font-size: 18px;
}

.input-wrapper i.bi-eye,
.input-wrapper i.bi-eye-slash {
  right: 0;
  left: auto;
  cursor: pointer;
  color: rgba(255, 255, 255, 0.5);
  transition: color 0.3s ease;
}

.input-wrapper i.bi-eye:hover,
.input-wrapper i.bi-eye-slash:hover {
  color: rgba(255, 255, 255, 0.8);
}

.input-wrapper input {
  width: 100%;
  background: transparent;
  border: none;
  padding: 10px 30px 10px 30px;
  color: white;
  font-size: 16px;
}

.input-wrapper input:focus {
  outline: none;
}

.input-wrapper input::placeholder {
  color: rgba(255, 255, 255, 0.5);
}

.register-button {
  padding: 12px;
  background-color: #00b7ea;
  color: white;
  border: none;
  border-radius: 3px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.3s;
}

.register-button:hover {
  background-color: #0096c8;
}

.register-button:disabled {
  background-color: #7dbfd5;
  cursor: not-allowed;
}

.spinner {
  display: inline-block;
  width: 20px;
  height: 20px;
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top-color: white;
  animation: spin 1s ease-in-out infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.error-message {
  margin-top: 20px;
  color: #ff4d4d;
  font-size: 14px;
}

.login-link {
  margin-top: 30px;
  color: rgba(255, 255, 255, 0.7);
  text-align: center;
}

.login-link a {
  color: #00b7ea;
  text-decoration: none;
  margin-left: 5px;
}

.login-link a:hover {
  text-decoration: underline;
}

/* Адаптивность для мобильных устройств */
@media (max-width: 768px) {
  .register-container {
    flex-direction: column;
    height: auto;
    max-height: 90vh;
  }

  .left-side {
    padding: 30px 20px;
    max-height: 90vh;
  }

  .right-side {
    display: none;
  }
}
</style>
