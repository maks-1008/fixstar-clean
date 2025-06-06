<script>
import { ref, reactive, onMounted, nextTick } from 'vue'
import { useRouter } from 'vue-router'
import { useUserStore } from '@/stores'

export default {
  name: 'LoginView',
  setup() {
    const router = useRouter()
    const userStore = useUserStore()

    // Состояния для переключения между формами
    const isSignupMode = ref(false)
    const isLoading = ref(false)
    const showLoginPassword = ref(false)
    const showSignupPassword = ref(false)

    // Ссылки на элементы DOM
    const usernameInput = ref(null)
    const passwordInput = ref(null)
    const signupPasswordInput = ref(null)

    // Формы и ошибки
    const loginForm = reactive({
      username: '',
      password: '',
    })

    const signupForm = reactive({
      username: '',
      email: '',
      phone: '',
      password: '',
      firstName: '',
      lastName: '',
    })

    const loginErrors = reactive({
      username: '',
      password: '',
      general: '',
    })

    const signupErrors = reactive({
      username: '',
      email: '',
      phone: '',
      password: '',
      general: '',
    })

    // Переключение между формами входа и регистрации
    const switchMode = () => {
      isSignupMode.value = !isSignupMode.value

      // Очистка ошибок при переключении
      Object.keys(loginErrors).forEach((key) => (loginErrors[key] = ''))
      Object.keys(signupErrors).forEach((key) => (signupErrors[key] = ''))

      // Фокус на первое поле активной формы
      nextTick(() => {
        if (isSignupMode.value) {
          // focus на поле username формы регистрации
        } else {
          usernameInput.value?.focus()
        }
      })
    }

    // Переключение видимости пароля
    const togglePasswordVisibility = (inputRef) => {
      if (inputRef === 'passwordInput') {
        showLoginPassword.value = !showLoginPassword.value
        if (passwordInput.value) {
          passwordInput.value.type = showLoginPassword.value ? 'text' : 'password'
        }
      } else if (inputRef === 'signupPasswordInput') {
        showSignupPassword.value = !showSignupPassword.value
        if (signupPasswordInput.value) {
          signupPasswordInput.value.type = showSignupPassword.value ? 'text' : 'password'
        }
      }
    }

    // Валидация формы регистрации
    const validateSignupForm = () => {
      let isValid = true

      // Сброс ошибок
      Object.keys(signupErrors).forEach((key) => (signupErrors[key] = ''))

      // Проверка логина
      if (!signupForm.username.trim()) {
        signupErrors.username = "Логін обов'язковий"
        isValid = false
      }

      // Проверка email
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
      if (!signupForm.email.trim()) {
        signupErrors.email = "Email обов'язковий"
        isValid = false
      } else if (!emailRegex.test(signupForm.email)) {
        signupErrors.email = 'Некоректний формат email'
        isValid = false
      }

      // Проверка телефона
      const phoneRegex = /^\+?[0-9]{10,13}$/
      if (!signupForm.phone.trim()) {
        signupErrors.phone = "Телефон обов'язковий"
        isValid = false
      } else if (!phoneRegex.test(signupForm.phone)) {
        signupErrors.phone = 'Некоректний формат телефону'
        isValid = false
      }

      // Проверка пароля
      if (!signupForm.password) {
        signupErrors.password = "Пароль обов'язковий"
        isValid = false
      } else if (signupForm.password.length < 8) {
        signupErrors.password = 'Пароль має бути не менше 8 символів'
        isValid = false
      }

      return isValid
    }

    // Авторизация
    const login = async () => {
      // Сброс ошибок
      Object.keys(loginErrors).forEach((key) => (loginErrors[key] = ''))

      if (!loginForm.username.trim()) {
        loginErrors.username = "Логін обов'язковий"
        return
      }

      if (!loginForm.password) {
        loginErrors.password = "Пароль обов'язковий"
        return
      }

      try {
        isLoading.value = true
        await userStore.login(loginForm)
        router.push('/')
      } catch (error) {
        console.error('Ошибка входа:', error)
        loginErrors.general = error.message || 'Помилка входу. Перевірте логін та пароль.'
      } finally {
        isLoading.value = false
      }
    }

    // Регистрация
    const register = async () => {
      if (!validateSignupForm()) {
        return
      }

      try {
        isLoading.value = true

        // Подготовка данных для регистрации
        const userData = {
          username: signupForm.username,
          email: signupForm.email,
          phone: signupForm.phone,
          password: signupForm.password,
          firstName: signupForm.firstName || '',
          lastName: signupForm.lastName || '',
        }

        await userStore.register(userData)
        router.push('/')
      } catch (error) {
        console.error('Ошибка регистрации:', error)
        signupErrors.general = error.message || 'Помилка реєстрації. Спробуйте пізніше.'
      } finally {
        isLoading.value = false
      }
    }

    // Инициализация
    onMounted(() => {
      // Если пользователь уже авторизован, перенаправляем на главную
      if (userStore.isAuthenticated) {
        router.push('/')
        return
      }

      // Фокус на поле логина при загрузке
      nextTick(() => {
        usernameInput.value?.focus()
      })
    })

    // Возвращаем все необходимые данные и функции
    return {
      isSignupMode,
      isLoading,
      loginForm,
      signupForm,
      loginErrors,
      signupErrors,
      usernameInput,
      passwordInput,
      signupPasswordInput,
      showLoginPassword,
      showSignupPassword,
      switchMode,
      login,
      register,
      togglePasswordVisibility,
    }
  },
}
</script>

<template>
  <div class="main-container">
    <!-- Используем класс signup-mode для переключения между формами -->
    <div class="login-signup-container" :class="{ 'signup-mode': isSignupMode }">
      <!-- Форма входа (левая сторона) -->
      <div id="login-section">
        <h2 class="form-title">Вхід</h2>
        <form @submit.prevent="login" autocomplete="off">
          <div class="input-field">
            <div class="field-wrapper">
              <i class="bi bi-person"></i>
              <input
                type="text"
                v-model="loginForm.username"
                placeholder="Логін"
                required
                autocomplete="off"
              />
            </div>
            <div v-if="loginErrors.username" class="alert alert-danger">
              {{ loginErrors.username }}
            </div>
          </div>

          <div class="input-field">
            <div class="field-wrapper">
              <i class="bi bi-lock"></i>
              <input
                :type="showLoginPassword ? 'text' : 'password'"
                v-model="loginForm.password"
                placeholder="Пароль"
                required
                autocomplete="new-password"
              />
              <button
                type="button"
                class="password-toggle"
                @click="togglePasswordVisibility('passwordInput')"
              >
                <i class="bi" :class="showLoginPassword ? 'bi-eye-slash' : 'bi-eye'"></i>
              </button>
            </div>
            <div v-if="loginErrors.password" class="alert alert-danger">
              {{ loginErrors.password }}
            </div>
          </div>

          <div v-if="loginErrors.general" class="alert alert-danger">{{ loginErrors.general }}</div>

          <button type="submit" class="submit-btn" :disabled="isLoading">
            <span v-if="isLoading" class="spinner"></span>
            <span v-else>УВІЙТИ</span>
          </button>

          <p class="switch-text">
            Ще немає облікового запису?
            <a href="#" @click.prevent="switchMode">Зареєструватися</a>
          </p>
        </form>
      </div>

      <!-- Приветственный текст (правая сторона для логина) -->
      <div id="login-welcome-panel" class="welcome-panel">
        <div class="welcome-content">
          <h3>ЛАСКАВО ПРОСИМО!</h3>
          <p>Ми раді бачити вас знову! Якщо вам потрібна допомога, ми завжди готові допомогти.</p>
        </div>
      </div>

      <!-- Приветственный текст (левая сторона для регистрации) -->
      <div id="signup-welcome-panel" class="welcome-panel">
        <div class="welcome-content">
          <h3>ВІТАЄМО!</h3>
          <p>Ми раді вітати вас! Якщо вам потрібна допомога, звертайтеся до нас.</p>
        </div>
      </div>

      <!-- Форма регистрации (правая сторона) -->
      <div id="signup-section">
        <h2 class="form-title">Реєстрація</h2>
        <form @submit.prevent="register" class="compact-form" autocomplete="off">
          <div class="input-field">
            <div class="field-wrapper">
              <i class="bi bi-person"></i>
              <input
                type="text"
                v-model="signupForm.username"
                placeholder="Логін"
                required
                autocomplete="off"
              />
            </div>
            <div v-if="signupErrors.username" class="alert alert-danger">
              {{ signupErrors.username }}
            </div>
          </div>

          <div class="input-field">
            <div class="field-wrapper">
              <i class="bi bi-envelope"></i>
              <input
                type="email"
                v-model="signupForm.email"
                placeholder="Електронна пошта"
                required
                autocomplete="off"
              />
            </div>
            <div v-if="signupErrors.email" class="alert alert-danger">{{ signupErrors.email }}</div>
          </div>

          <div class="input-field">
            <div class="field-wrapper">
              <i class="bi bi-telephone"></i>
              <input
                type="tel"
                v-model="signupForm.phone"
                placeholder="Номер телефону"
                required
                autocomplete="off"
              />
            </div>
            <div v-if="signupErrors.phone" class="alert alert-danger">{{ signupErrors.phone }}</div>
          </div>

          <div class="input-field">
            <div class="field-wrapper">
              <i class="bi bi-lock"></i>
              <input
                :type="showSignupPassword ? 'text' : 'password'"
                v-model="signupForm.password"
                placeholder="Пароль"
                ref="signupPasswordInput"
                required
                minlength="8"
                autocomplete="new-password"
              />
              <button
                type="button"
                class="password-toggle"
                @click="togglePasswordVisibility('signupPasswordInput')"
              >
                <i class="bi" :class="showSignupPassword ? 'bi-eye-slash' : 'bi-eye'"></i>
              </button>
            </div>
            <div v-if="signupErrors.password" class="alert alert-danger">
              {{ signupErrors.password }}
            </div>
          </div>

          <div v-if="signupErrors.general" class="alert alert-danger">
            {{ signupErrors.general }}
          </div>

          <button type="submit" class="submit-btn" :disabled="isLoading">
            <span v-if="isLoading" class="spinner"></span>
            <span v-else>ЗАРЕЄСТРУВАТИСЯ</span>
          </button>

          <div class="switch-text">
            Вже маєте обліковий запис? <a href="#" @click.prevent="switchMode">Увійти</a>
          </div>
        </form>
      </div>
    </div>
    <!-- Добавляем футер -->
    <div class="login-footer">© FixStar 2024 Всі права захищені</div>
  </div>
</template>

<style scoped>
/* Основной контейнер */
.main-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  padding: 20px;
  position: relative;
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

.login-signup-container {
  position: relative;
  width: 100%;
  max-width: 768px;
  height: 480px;
  margin: 5vh auto;
  box-shadow: 0 0 35px 8px rgba(0, 210, 255, 0.8);
  border-radius: 15px;
  overflow: hidden;
  display: flex;
  border: 2px solid rgba(0, 210, 255, 1);
  background-color: #071423;
  z-index: 1;
}

/* Стиль для футера - как на главной */
.login-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  text-align: center;
  background-color: #212529;
  color: white;
  font-size: 16px;
  padding: 6px 0;
  margin: 0;
  line-height: 1.2;
  height: 35px;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

/* Медиа-запросы для мобильной адаптации */
@media (max-width: 768px) {
  .main-container::before {
    background-image: url('/images/bg-image5.jpg');
  }

  .login-footer {
    font-size: 14px;
    height: 30px;
  }
}

/* Общие стили для секций форм */
#login-section,
#signup-section {
  width: 50%;
  padding: 30px 40px;
  background-color: #071423; /* Тёмно-синий фон */
  color: #ffffff;
  z-index: 2;
  display: flex;
  flex-direction: column;
  position: relative;
  transition: all 0.6s ease-in-out;
}

/* Позиционирование форм для анимации */
#login-section {
  left: 0;
}

#signup-section {
  position: absolute;
  right: 0;
  opacity: 0;
  visibility: hidden;
  transform: translateX(100%);
}

/* Когда контейнер в режиме регистрации */
.login-signup-container.signup-mode #login-section {
  transform: translateX(-100%);
  opacity: 0;
  visibility: hidden;
}

.login-signup-container.signup-mode #signup-section {
  transform: translateX(0);
  opacity: 1;
  visibility: visible;
}

/* Стили для приветственных панелей */
.welcome-panel {
  width: 50%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  position: absolute;
  transition: all 0.6s ease-in-out;
  overflow: hidden;
  background: linear-gradient(135deg, #00a8d9, #00e5ff);
}

/* Приветственная панель для login формы (справа) */
#login-welcome-panel {
  right: 0;
  background: linear-gradient(to bottom right, #00d4ff, #0081a3);
  clip-path: polygon(100% 0%, 100% 100%, 0% 100%, 42% 0%);
}

/* Приветственная панель для signup формы (слева) */
#signup-welcome-panel {
  left: 0;
  transform: translateX(-100%);
  background: linear-gradient(to top left, #00d4ff, #0081a3);
  clip-path: polygon(0% 0%, 60% 0%, 100% 100%, 0% 100%);
}

/* Когда контейнер в режиме регистрации */
.login-signup-container.signup-mode #login-welcome-panel {
  transform: translateX(100%);
}

.login-signup-container.signup-mode #signup-welcome-panel {
  transform: translateX(0);
}

/* Контейнер текста в приветственной панели */
.welcome-content {
  max-width: 80%;
  position: relative;
  z-index: 1;
  padding: 20px;
  background-color: rgba(0, 15, 30, 0.5); /* Увеличиваем непрозрачность для контраста */
  border-radius: 15px;
  box-shadow:
    0 0 20px rgba(0, 0, 0, 0.3),
    inset 0 0 15px rgba(0, 210, 255, 0.3);
  backdrop-filter: blur(5px);
  border: 1px solid rgba(0, 210, 255, 0.3);
  transition: all 0.6s ease-in-out; /* Добавляем анимацию для контента */
  opacity: 1;
}

#login-welcome-panel .welcome-content {
  text-align: right;
  margin-left: 80px;
  transition:
    opacity 0.6s ease-in-out,
    transform 0.6s ease-in-out;
  transform: translateX(0);
  opacity: 1;
}

#signup-welcome-panel .welcome-content {
  text-align: left;
  margin-right: 80px;
  opacity: 0;
  transform: translateX(-20px);
  transition:
    opacity 0.6s ease-in-out,
    transform 0.6s ease-in-out;
}

/* Анимация для содержимого приветственных панелей */
.login-signup-container.signup-mode #login-welcome-panel .welcome-content {
  opacity: 0;
  transform: translateX(20px);
}

.login-signup-container.signup-mode #signup-welcome-panel .welcome-content {
  opacity: 1;
  transform: translateX(0);
  transition-delay: 0.1s;
}

/* Заголовок формы */
.form-title {
  color: #00d4ff;
  margin-bottom: 20px; /* Уменьшаем отступ */
  font-size: 2rem; /* Уменьшаем размер заголовка */
  font-weight: 600;
  text-align: center;
  text-shadow: 0 0 15px rgba(0, 210, 255, 0.7);
}

/* Заголовок приветствия */
.welcome-panel h3 {
  font-size: 3rem;
  font-weight: 700;
  color: white;
  margin-bottom: 15px;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

/* Текст приветствия */
.welcome-panel p {
  font-size: 1.2rem;
  line-height: 1.4;
  margin-top: 0;
  color: #ffffff;
}

/* Стили для полей ввода */
.input-field {
  position: relative;
  margin-bottom: 15px; /* Уменьшаем отступ между полями */
}

.input-field label {
  display: block;
  margin-bottom: 5px; /* Уменьшаем отступ */
  color: #ffffff;
  font-size: 1rem;
}

/* Стилизация инпутов в стиле изображений */
.field-wrapper {
  position: relative;
  display: flex;
  align-items: center;
  border: 1px solid rgba(0, 210, 255, 0.5);
  border-radius: 8px;
  margin: 8px 0; /* Уменьшаем отступы */
  height: 45px; /* Уменьшаем высоту полей */
  background-color: rgba(0, 15, 30, 0.5);
}

.field-wrapper i.bi {
  position: absolute;
  left: 15px;
  color: rgba(255, 255, 255, 0.8);
  font-size: 1.1rem; /* Уменьшаем размер иконок */
  line-height: 45px;
  transition: all 0.3s ease;
  z-index: 1;
}

.field-wrapper input {
  width: 100%;
  height: 45px; /* Уменьшаем высоту полей */
  background: none;
  border: none;
  padding: 0 15px 0 45px;
  color: #fff;
  font-size: 0.95rem; /* Уменьшаем размер шрифта */
  border-radius: 8px;
}

/* Стили для автозаполненных полей */
input:-webkit-autofill,
input:-webkit-autofill:hover,
input:-webkit-autofill:focus,
input:-webkit-autofill:active {
  -webkit-box-shadow: 0 0 0 30px #071423 inset !important;
  -webkit-text-fill-color: #ffffff !important;
  transition: background-color 5000s ease-in-out 0s;
  caret-color: #ffffff;
}

/* Для Firefox и других браузеров */
input:autofill,
input:autofill:hover,
input:autofill:focus,
input:autofill:active {
  box-shadow: 0 0 0 30px #071423 inset !important;
  -webkit-text-fill-color: #ffffff !important;
  filter: none;
  color: #ffffff !important;
  background-color: transparent !important;
}

.input-field input::placeholder {
  color: rgba(255, 255, 255, 0.7);
}

/* Стиль кнопки отправки */
.submit-btn {
  width: 100%;
  padding: 0.5rem 2rem;
  background: linear-gradient(135deg, #0081a3, #00d4ff);
  border: none;
  border-radius: 8px;
  color: #fff;
  font-weight: 600;
  cursor: pointer;
  text-transform: uppercase;
  box-shadow: 0 0 15px rgba(0, 130, 160, 0.5);
  transition: all 0.3s ease;
  height: 45px; /* Уменьшаем высоту кнопки */
  margin-top: 15px; /* Уменьшаем отступ */
  font-size: 0.95rem; /* Уменьшаем размер шрифта */
}

.submit-btn:hover {
  box-shadow: 0 0 20px rgba(0, 210, 255, 0.7); /* Усиливаем свечение при наведении */
  background: linear-gradient(135deg, #00a8d9, #00e5ff);
}

/* Стили для текста переключения */
.switch-text {
  margin-top: 15px; /* Уменьшаем отступ */
  color: rgba(255, 255, 255, 0.8);
  text-align: center;
  font-size: 0.95rem; /* Уменьшаем размер шрифта */
}

.switch-text a {
  color: #00d4ff;
  font-weight: 600;
  text-decoration: none;
}

.switch-text a:hover {
  text-decoration: underline;
}

/* Стили для ошибок валидации */
.alert-danger {
  background-color: rgba(220, 53, 69, 0.1);
  border: 1px solid rgba(220, 53, 69, 0.3);
  color: #dc3545;
  border-radius: 4px;
  margin: 5px 0;
  padding: 5px 10px;
  font-size: 0.8rem;
}

/* Стиль для невалидного ввода */
.field-wrapper input.is-invalid {
  border-color: #dc3545;
  box-shadow: 0 0 0 0.2rem rgba(220, 53, 69, 0.25);
}

/* Стиль для кнопки переключения видимости пароля */
.password-toggle {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  background: transparent;
  border: none;
  color: rgba(255, 255, 255, 0.7);
  cursor: pointer;
  padding: 0;
  height: 100%;
  width: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2;
}

.password-toggle:hover {
  color: #00d4ff;
}

.password-toggle:focus {
  outline: none;
}

.field-wrapper input[type='password'],
.field-wrapper input[type='text'] {
  padding-right: 40px !important;
}

/* Спиннер для кнопки при загрузке */
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

/* Медиа-запросы для мобильных устройств */
@media (max-width: 768px) {
  .login-signup-container {
    flex-direction: column;
    height: auto;
    max-width: 95%;
  }

  #login-section,
  #signup-section {
    width: 100%;
    padding: 20px;
    position: relative;
  }

  #login-section {
    left: auto;
  }

  #signup-section {
    right: auto;
    position: relative;
    transform: none;
    display: none;
  }

  .login-signup-container.signup-mode #login-section {
    transform: none;
    display: none;
  }

  .login-signup-container.signup-mode #signup-section {
    display: flex;
  }

  .welcome-panel {
    width: 100%;
    height: auto;
    position: relative;
    text-align: center;
    transform: none;
  }

  #login-welcome-panel,
  #signup-welcome-panel {
    display: none;
  }

  .welcome-content {
    max-width: 100%;
    margin: 0 auto;
    padding: 0;
    text-align: center;
  }
}
</style>
