<script>
import { useProductStore } from '@/stores'
import { useCartStore } from '@/stores'
import { onMounted, ref, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import Sidebar from '@/components/Sidebar.vue'

export default {
  name: 'CategoryView',
  components: {
    Sidebar,
  },
  setup() {
    const productStore = useProductStore()
    const cartStore = useCartStore()
    const route = useRoute()
    const router = useRouter()

    const loading = ref(true)
    const categorySlug = computed(() => route.params.slug)

    const category = computed(() => {
      return productStore.getCategoryBySlug(categorySlug.value)
    })

    const subcategories = computed(() => {
      if (!category.value) return []
      return productStore.getSubcategoriesByCategory(category.value.id)
    })

    const products = ref([])
    const categoryTitle = ref('')
    const categoryDescription = ref('')

    const addToCart = (product) => {
      cartStore.addToCart({
        id: product.id,
        name: product.name,
        price: parseFloat(product.price),
        image: product.image,
        quantity: 1,
      })
      alert(`Товар "${product.name}" додано до кошика!`)
    }

    onMounted(async () => {
      if (productStore.categories.length === 0) {
        await productStore.fetchCategories()
      }

      if (productStore.subcategories.length === 0) {
        await productStore.fetchSubcategories()
      }

      loading.value = false

      if (categorySlug.value === 'bolti-z-shestigrannoyu-golovkoyu') {
        categoryTitle.value = 'Болти з шестигранною головкою'
        categoryDescription.value =
          'Болти з шестигранною головкою застосовуються для створення нерозємних зєднань. Вони мають шестигранну головку та різьбу метричну.'
        products.value = [
          {
            id: 1,
            name: 'DIN 931',
            code: 'Болт з шестигранною головкою',
            price: '25.50',
            image: '/images/goods/DIN931.webp',
          },
          {
            id: 2,
            name: 'DIN 933',
            code: 'Болт з шестигранною головкою',
            price: '28.75',
            image: '/images/goods/DIN933.webp',
          },
        ]
      } else if (categorySlug.value === 'fasteners') {
        router.replace({ name: 'category', params: { slug: 'bolti-z-shestigrannoyu-golovkoyu' } })
      } else {
        if (!loading.value && !category.value) {
          router.push({ name: 'notFound' })
        }

        fetchProducts(categorySlug.value)
      }
    })

    const fetchProducts = (slug) => {
      console.log('Загрузка товаров для категории:', slug)
    }

    return {
      category,
      subcategories,
      loading,
      products,
      categoryTitle,
      categoryDescription,
      addToCart,
      fetchProducts,
    }
  },
}
</script>

<template>
  <div class="category-view">
    <div class="background-container"></div>

    <div class="content-overlay">
      <!-- Боковое меню с категориями -->
      <Sidebar />

      <div class="product-grid">
        <!-- Товары категории -->
        <div v-for="product in products" :key="product.id" class="product-card">
          <div class="product-image">
            <img :src="product.image" :alt="product.name" />
          </div>
          <div class="product-info">
            <h3>{{ product.name }}</h3>
          </div>
        </div>
      </div>

      <!-- Копирайт внизу -->
      <footer class="copyright">© FixStar 2024 Всі права захищені</footer>
    </div>
  </div>
</template>

<style scoped>
.category-view {
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
  padding: 80px 20px 60px;
}

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 15px;
  max-width: 1000px;
  margin: 0 auto 0 250px; /* Отступ слева от боковой панели */
  padding: 10px;
}

.product-card {
  border-radius: 4px;
  overflow: hidden;
  transition: all 0.3s ease;
  height: 100%;
  display: flex;
  flex-direction: column;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  position: relative;
  background-color: #ffffff;
}

.product-card:hover {
  transform: translateY(-5px) scale(1.02);
  box-shadow: 0 12px 20px rgba(0, 0, 0, 0.3);
}

.product-image {
  height: 250px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 10px;
}

.product-image img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  transition: transform 0.3s ease;
}

.product-card:hover .product-image img {
  transform: scale(1.1);
}

.product-info {
  padding: 10px;
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  text-align: center;
  background-color: #ffffff;
}

.product-info h3 {
  font-size: 1.7rem;
  font-weight: 700;
  margin: 0;
  color: #333;
}

.copyright {
  position: absolute;
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

@media (max-width: 768px) {
  .background-container {
    background-image: url('/images/bg-image5.jpg');
  }

  .product-grid {
    margin: 0 auto;
  }

  .copyright {
    position: fixed;
  }
}
</style>
