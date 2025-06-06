<template>
  <div class="login-modal-overlay" v-if="show" @click.self="closeModal">
    <div class="login-modal">
      <div class="login-container">
        <div class="left-side">
          <h1>Вхід</h1>
          <div class="input-group">
            <div class="input-wrapper">
              <i class="bi bi-person"></i>
              <input
                type="text"
                placeholder="Логін"
                v-model="credentials.username"
                ref="usernameInput"
              />
            </div>
            <div class="input-wrapper">
              <i class="bi bi-lock"></i>
              <input
                type="password"
                placeholder="Пароль"
                v-model="credentials.password"
                ref="passwordInput"
              />
              <i
                class="bi"
                :class="passwordVisible ? 'bi-eye-slash' : 'bi-eye'"
                @click="togglePasswordVisibility"
              ></i>
            </div>
          </div>
          <button class="login-button" @click="login" :disabled="isLoading">
            <span v-if="isLoading" class="spinner"></span>
            <span v-else>УВІЙТИ</span>
          </button>
          <div v-if="error" class="error-message">{{ error }}</div>
          <div class="register-link">
            Ще немає облікового запису?
            <a href="#" @click.prevent="goToRegister">Зареєструватися</a>
          </div>
        </div>
        <div class="right-side">
          <h1>ЛАСКАВО ПРОСИМО!</h1>
          <p>Ми раді бачити вас знову! Якщо вам потрібна допомога, ми завжди готові допомогти.</p>
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
  name: 'LoginModal',
  props: {
    show: {
      type: Boolean,
      required: true,
    },
  },
  emits: ['close', 'register'],
  setup(props, { emit }) {
    const userStore = useUserStore()
    const router = useRouter()

    const credentials = reactive({
      username: '',
      password: '',
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

    const login = async () => {
      // Валидация формы
      if (!credentials.username || !credentials.password) {
        error.value = 'Будь ласка, заповніть всі поля'
        return
      }

      isLoading.value = true
      error.value = null

      try {
        await userStore.login(credentials)
        closeModal()

        // Переход на главную страницу или страницу личного кабинета
        if (userStore.user && userStore.user.username === 'admin') {
          router.push('/admin')
        }
      } catch (err) {
        error.value = err.message || 'Помилка при вході'
        console.error(err)
      } finally {
        isLoading.value = false
      }
    }

    const closeModal = () => {
      emit('close')
    }

    const goToRegister = () => {
      emit('register')
    }

    return {
      credentials,
      isLoading,
      error,
      passwordVisible,
      usernameInput,
      passwordInput,
      togglePasswordVisibility,
      login,
      closeModal,
      goToRegister,
    }
  },
}
</script>

<style scoped>
.login-modal-overlay {
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

.login-modal {
  width: 100%;
  max-width: 800px;
  background: none;
  border-radius: 10px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
}

.login-container {
  display: flex;
  height: 500px;
}

.left-side {
  flex: 1;
  background-color: #0c1016;
  padding: 40px;
  display: flex;
  flex-direction: column;
  justify-content: center;
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
  margin-bottom: 40px;
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

.login-button {
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

.login-button:hover {
  background-color: #0096c8;
}

.login-button:disabled {
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

.register-link {
  margin-top: 30px;
  color: rgba(255, 255, 255, 0.7);
  text-align: center;
}

.register-link a {
  color: #00b7ea;
  text-decoration: none;
  margin-left: 5px;
}

.register-link a:hover {
  text-decoration: underline;
}

/* Адаптивность для мобильных устройств */
@media (max-width: 768px) {
  .login-container {
    flex-direction: column;
    height: auto;
  }

  .left-side,
  .right-side {
    padding: 30px 20px;
  }

  .right-side {
    display: none;
  }
}
</style>
