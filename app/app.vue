<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const supabase = useSupabaseClient()
const user = useSupabaseUser() 
const adminPanelRef = ref(null) 

// Datos Globales
const products = ref([])
const cart = ref([])
const isCartOpen = ref(false)
const showToast = ref(false)

// Variable para el Modal de Carrito Vacío
const showEmptyCartModal = ref(false)

// Variable para el cartelito de horario (Nuevo)
const showTimeWarning = ref(true)

// Variables para el Modal de Borrar
const showRemoveModal = ref(false)
const itemToRemoveIndex = ref(null)

// Variables para el Modal de Stock (NUEVO)
const showStockModal = ref(false)
const stockLimit = ref(0)

// NUEVAS VARIABLES PARA ERROR
const showError = ref(false)
const errorMessage = ref('')

// CAMBIA ESTO POR TU DIRECCIÓN REAL O COORDENADAS
const mapLink = "https://maps.app.goo.gl/WfH6D5vtzfJk6J4x7"
// Filtros
const searchQuery = ref('')
const selectedCategory = ref('Todos')
const selectedSize = ref('Todos')
const selectedMarca = ref('Todos')
const showOnlyOffers = ref(false) 

const categoriesList = ['Todos', 'Pañales', 'Toallitas', 'Cuidado Capilar', 'Oleos' ,'Jabónes', 'Cremas','Guantes','Algodones','Apositos','Jabones','Accesorios']
const talleList = ['Todos', 'PR', 'RN', 'P', 'J','M', 'G', 'XG', 'XXG', 'XXXG'] 
const marcasList = ['Todos', 'Pampers', 'Huggies', 'Estrella', 'Babysec']

// Función para subir la pantalla suavemente
const subirArriba = () => {
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

// Vigilar el buscador: si el cliente escribe, reseteamos los demás filtros
watch(searchQuery, (nuevoTexto) => {
  if (nuevoTexto.length > 0) {
    selectedCategory.value = 'Todos'
    selectedMarca.value = 'Todos'
    selectedSize.value = 'Todos'
    showOnlyOffers.value = false 
    
    // ESTA ES LA LÍNEA NUEVA: Sube la pantalla suavemente hasta arriba
    window.scrollTo({ top: 0, behavior: 'smooth' })
  }
})

// Variables para el modal de imagen en grande
const showImageModal = ref(false)
const selectedImage = ref('')

const openImageModal = (imageUrl) => {
  if (!imageUrl) return // Por si el producto no tiene foto
  selectedImage.value = imageUrl
  showImageModal.value = true
}

// Variables de estado para los menús
const isBrandMenuOpen = ref(false)
const isSizeMenuOpen = ref(false)
const menuPosition = ref({ top: '0px', left: '0px' })

// Esta es la nueva función que calcula dónde dibujar la lista
const openMenu = (event, type) => {
  const rect = event.currentTarget.getBoundingClientRect()
  const menuWidth = type === 'brand' ? 180 : 120
  
  let posX = rect.left
  // Si el menú se sale de la pantalla por la derecha, lo corremos hacia la izquierda
  if (posX + menuWidth > window.innerWidth) {
    posX = window.innerWidth - menuWidth - 20 
  }

  // Guardamos la posición (abajo del botón)
  menuPosition.value = { 
    top: `${rect.bottom + 8}px`, 
    left: `${posX}px` 
  }
  
  // Abrimos el menú correcto
  if (type === 'brand') {
    isBrandMenuOpen.value = !isBrandMenuOpen.value
    isSizeMenuOpen.value = false
  } else {
    isSizeMenuOpen.value = !isSizeMenuOpen.value
    isBrandMenuOpen.value = false
  }
}

// Funciones para elegir las opciones y cerrar
const chooseBrand = (marca) => {
  selectedMarca.value = marca
  selectedCategory.value = 'Pañales'
  selectedSize.value = 'Todos'
  searchQuery.value = '' // Limpiar el buscador para que no interfiera
  isBrandMenuOpen.value = false
  subirArriba() // Subir la pantalla al elegir marca, para mejorar UX
}

const chooseSize = (talle) => {
  selectedSize.value = talle
  isSizeMenuOpen.value = false
  searchQuery.value = '' // Limpiar el buscador para que no interfiera
  subirArriba() // Subir la pantalla al elegir talle, para mejorar UX
}

// --- CARGAR DATOS ---
const fetchProducts = async () => {
  const { data } = await supabase.from('productos').select('*').order('created_at', { ascending: false })
  if (data) products.value = data
}
await fetchProducts()

// --- FILTRADO ---
// --- FILTRADO DE PRODUCTOS ---
const filteredProducts = computed(() => {
  let temp = [...products.value] 

  // 1. Filtra por Categoría (Ej: Pañales)
  if (selectedCategory.value !== 'Todos') {
    temp = temp.filter(p => p.category?.toLowerCase() === selectedCategory.value.toLowerCase())
  }
  
  // 2. Filtra por Marca (Ej: Pampers) - ¡Acá está la clave para que se sume al talle!
  if (selectedCategory.value === 'Pañales' && selectedMarca.value !== 'Todos') {
    temp = temp.filter(p => p.marca === selectedMarca.value)
  }

  // 3. Filtra por Talle (Ej: M) - Se suma a la marca que ya elegiste arriba
  if (selectedSize.value !== 'Todos') {
    temp = temp.filter(p => p.talle?.toUpperCase() === selectedSize.value.toUpperCase())
  }

  // 4. Filtra por Buscador (Si escribiste algo)
  if (searchQuery.value) {
    temp = temp.filter(p => p.name.toLowerCase().includes(searchQuery.value.toLowerCase()))
  }
  
  // 5. Filtra por Ofertas
  if (showOnlyOffers.value) {
    temp = temp.filter(p => p.is_promo === true)
  }

  // Ordena para mostrar las ofertas primero
  temp.sort((a, b) => {
    if (a.is_promo && !b.is_promo) return -1
    if (!a.is_promo && b.is_promo) return 1
    return 0
  })

  return temp
})

// --- CARRITO ---
onMounted(() => { const s = localStorage.getItem('cart_v5'); if(s) cart.value = JSON.parse(s) })
watch(cart, (n) => localStorage.setItem('cart_v5', JSON.stringify(n)), { deep: true })

const addToCart = (p) => {
  const item = cart.value.find(i => i.id === p.id)
  if (item) {
    if (item.quantity < p.stock) { 
        item.quantity++ 
        showNotif() 
    } else {
        // CAMBIO AQUÍ: En vez de alert, abrimos el modal
        stockLimit.value = p.stock
        showStockModal.value = true
    }
  } else {
    if (p.stock > 0) { 
        cart.value.push({ ...p, quantity: 1 })
        showNotif() 
    }
  }
}

const incrementQty = (item) => {
  if (item.quantity < item.stock) {
      item.quantity++
  } else {
      // CAMBIO AQUÍ: En vez de alert, abrimos el modal
      stockLimit.value = item.stock
      showStockModal.value = true
  }
}

const decrementQty = (index) => {
  const item = cart.value[index]
  if (item.quantity > 1) {
      item.quantity--
  } else {
      // En vez del alert feo, abrimos nuestro modal lindo
      itemToRemoveIndex.value = index
      showRemoveModal.value = true
  }
}

const confirmRemoveItem = () => {
    if (itemToRemoveIndex.value !== null) {
        cart.value.splice(itemToRemoveIndex.value, 1) // Borramos el producto
        showRemoveModal.value = false // Cerramos el cartel
        itemToRemoveIndex.value = null
    }
}

// Función para el botón de basura directo
const askToRemove = (index) => {
    itemToRemoveIndex.value = index
    showRemoveModal.value = true
}

const showNotif = () => { showToast.value = true; setTimeout(() => showToast.value = false, 2000) }
const totalItemsCount = computed(() => cart.value.reduce((t, i) => t + i.quantity, 0))
const cartTotal = computed(() => cart.value.reduce((t, i) => t + (i.price * i.quantity), 0))

const checkout = async () => {
  if(cart.value.length === 0) {
    showEmptyCartModal.value = true 
    return
  }
  const { error } = await supabase.from('ventas').insert({ items: cart.value, total: cartTotal.value, estado: 'Pendiente' })
  if(error) return alert('Error al guardar pedido')
  
  for (const item of cart.value) {
    const nuevoStock = item.stock - item.quantity
    await supabase.from('productos').update({ stock: nuevoStock }).eq('id', item.id)
  }

  const phone = "542975011424" 
  let text = "Hola! Pedido Web:%0A"

  // ACÁ ESTÁ LA MAGIA: Armamos el mensaje incluyendo el link de la foto
  cart.value.forEach(i => {
    text += `(Cantidad: ${i.quantity}) ${i.name} - $${i.price*i.quantity}%0A`
    if (i.image) {
       // encodeURIComponent asegura que el link de la imagen no rompa el enlace de WhatsApp
       text += `Ver foto: ${encodeURIComponent(i.image)}%0A`
    }
    text += `%0A` // Espacio extra entre productos para que se lea mejor
  })

  text += `%0ATotal: $${cartTotal.value}`
  
  cart.value = []
  isCartOpen.value = false
  await fetchProducts()
  window.open(`https://wa.me/${phone}?text=${text}`, '_blank')
}

// LOGIN ADMIN
const showLogin = ref(false); const email = ref(''); const password = ref('')
const handleLogin = async () => { 
  const { error } = await supabase.auth.signInWithPassword({ email: email.value, password: password.value })
  
  if (!error) {
      showLogin.value = false 
  } else {
      // TRADUCCIÓN DEL ERROR PARA QUE SE VEA LINDO
      if (error.message.includes('Invalid login credentials')) {
          errorMessage.value = '❌ Email o contraseña incorrectos'
      } else {
          errorMessage.value = '⚠️ Error: ' + error.message
      }
      
      // Mostrar el cartel rojo por 3 segundos
      showError.value = true
      setTimeout(() => showError.value = false, 3000)
  }
}
const logout = async () => await supabase.auth.signOut()
const formatoMoneda = (v) => new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS' }).format(v)
</script>

<template>
  <div class="min-h-screen bg-gray-50 pb-0 font-sans text-gray-800">
    <div v-if="showToast" class="fixed top-20 right-5 z-50 bg-gray-800 text-white px-6 py-3 rounded-xl shadow-2xl animate-bounce-in flex items-center gap-2"><span>✅</span> Agregado</div>
    <div v-if="showError" class="fixed top-20 right-5 z-[70] bg-red-500 text-white px-6 py-3 rounded-xl shadow-2xl animate-bounce-in flex items-center gap-2 border-2 border-red-400">
      <span class="font-bold">{{ errorMessage }}</span>
    </div>
    <header class="bg-sky-400 text-white px-4 h-20 sticky top-0 z-50 shadow-lg flex justify-between items-center">
      <div class="flex items-center gap-3">
      <img 
        src="/imagen-bebe.jpg"
        alt="Logo Bebé" 
        class="w-12 h-12 bg-white rounded-full p-1 shadow-sm object-cover"
      >
      <h1 class="text-lg font-bold leading-tight">
        Pañalera Gianluca
      </h1>
  </div>
      <div class="flex gap-2">
         <button v-if="user" @click="adminPanelRef.openNew()" class="bg-white text-sky-400 px-3 py-1 rounded text-sm font-bold shadow hover:bg-gray-100"> Agregar</button>
         <button v-if="user" @click="logout" class="text-xs bg-sky-800 px-2 rounded">Salir</button>
         <button @click="isCartOpen = !isCartOpen" class="relative group hover:scale-105 transition-transform duration-200 p-1">
    
            <div class="bg-white w-10 h-10 rounded-full flex items-center justify-center shadow-md group-hover:shadow-lg transition-shadow">
                <span class="text-2xl transform -translate-y-0.5">🛒</span>
            </div>

            <span v-if="totalItemsCount" class="absolute -top-1 -right-1 bg-red-500 text-white text-[10px] font-bold rounded-full w-5 h-5 flex items-center justify-center border-2 border-white shadow-sm">
                {{ totalItemsCount }}
            </span>

        </button>
      </div>
    </header>

    <div class="max-w-[1400px] mx-auto flex flex-col lg:flex-row gap-4 items-start pb-4">
      
      <aside class="hidden lg:block w-1/5 sticky top-24 h-fit transition-all space-y-3">
    
    <a :href="mapLink" target="_blank" class="block bg-white p-3 rounded-2xl shadow-sm border border-gray-100 hover:shadow-md transition group relative overflow-hidden cursor-pointer">
        <h3 class="font-bold text-sm mb-2 text-gray-700 flex justify-between">
            📍 Retiros
            <span class="text-xs text-sky-500 group-hover:underline">Ir ahora ↗</span>
        </h3>
        
        <div class="w-full h-48 bg-gray-200 rounded-xl overflow-hidden shadow-inner relative">
            <iframe src="https://www.google.com/maps/embed?pb=!1m10!1m8!1m3!1d2092.0525232936952!2d-67.54652908523627!3d-45.88601087901825!3m2!1i1024!2i768!4f13.1!5e1!3m2!1ses!2sar!4v1767656985269!5m2!1ses!2sar" width="100%" height="100%" style="border:0;" allowfullscreen="" loading="lazy" referrerpolicy="no-referrer-when-downgrade"></iframe>
            <div class="absolute inset-0 bg-black/10 group-hover:bg-black/20 transition flex items-center justify-center">
                <span class="bg-white text-gray-800 px-4 py-2 rounded-full font-bold text-xs shadow-lg transform scale-95 group-hover:scale-105 transition flex items-center gap-1">
                    🚀 Cómo llegar
                </span>
            </div>
        </div>
        <p class="text-[10px] text-center text-gray-400 mt-2">Toca para abrir GPS</p>
    </a>

    </aside>

      <main class="flex-1 w-full lg:w-3/5">  
        <div class="bg-white p-4 rounded-b-2xl shadow-sm border-b border-gray-100 mb-6 sticky top-[80px] z-40 w-full">
            <div class="relative mb-4">
              <span class="absolute left-3 top-3 text-gray-400">🔍</span>
              <input v-model="searchQuery" placeholder="Buscar..." class="w-full pl-10 pr-4 py-3 rounded-xl border bg-gray-50 focus:bg-white outline-none focus:border-sky-500 transition">
            </div>
            
            <div class="mb-4 overflow-x-auto scroll-elegante pb-2">
              <div class="flex gap-2">
                <template v-for="c in categoriesList" :key="c">

                    <button v-if="c !== 'Pañales'" 
                      @click="selectedCategory=c; selectedMarca='Todos'; selectedSize='Todos'; searchQuery=''; subirArriba()" 
                      :class="['px-3 py-1.5 rounded-full text-xs font-bold transition whitespace-nowrap', selectedCategory===c?'bg-sky-400 text-white shadow-md':'bg-gray-100 text-gray-500 hover:bg-gray-200']">
                      {{ c }}
                    </button>

                    <div v-else class="flex gap-2 flex-shrink-0">
                        
                        <button 
                            @click="(e) => openMenu(e, 'brand')"
                            :class="['px-3 py-1.5 rounded-full text-xs font-bold transition flex items-center gap-2 whitespace-nowrap outline-none', selectedCategory==='Pañales' ? 'bg-sky-400 text-white shadow-md' : 'bg-gray-100 text-gray-500 hover:bg-gray-200']">
                            <span>Pañales ({{ selectedMarca }})</span>
                            <span :class="['text-[9px] transition-transform duration-300', isBrandMenuOpen ? 'rotate-180' : '']">▼</span>
                        </button>

                        <teleport to="body">
                            <div v-if="isBrandMenuOpen" class="fixed z-[120] bg-white border border-gray-100 shadow-2xl rounded-2xl overflow-hidden py-2 min-w-[180px]" :style="menuPosition">
                                <div @click="chooseBrand('Todos')" :class="['px-4 py-2.5 text-sm cursor-pointer transition flex justify-between items-center', selectedMarca === 'Todos' ? 'bg-sky-50 text-sky-600 font-bold' : 'text-gray-600 hover:bg-gray-50']">
                                    Todas las marcas <span v-if="selectedMarca === 'Todos'">✓</span>
                                </div>
                                <div v-for="m in marcasList.filter(x => x !== 'Todos')" :key="m" @click="chooseBrand(m)" :class="['px-4 py-2.5 text-sm cursor-pointer transition flex justify-between items-center', selectedMarca === m ? 'bg-sky-50 text-sky-600 font-bold' : 'text-gray-600 hover:bg-gray-50']">
                                    {{ m }} <span v-if="selectedMarca === m">✓</span>
                                </div>
                            </div>
                            <div v-if="isBrandMenuOpen" @click="isBrandMenuOpen = false" class="fixed inset-0 z-[110]"></div>
                        </teleport>

                        <button v-if="selectedCategory === 'Pañales'"
                            @click="(e) => openMenu(e, 'size')"
                            :class="['px-3 py-1.5 rounded-full text-xs font-bold transition flex items-center gap-2 whitespace-nowrap outline-none animate-fade-in', selectedSize !== 'Todos' ? 'bg-gray-800 text-white shadow-md' : 'bg-white border border-gray-200 text-gray-600 hover:bg-gray-50']">
                            <span>Talle: {{ selectedSize }}</span>
                            <span :class="['text-[9px] transition-transform duration-300', isSizeMenuOpen ? 'rotate-180' : '']">▼</span>
                        </button>

                        <teleport to="body">
                            <div v-if="isSizeMenuOpen" class="fixed z-[120] bg-white border border-gray-100 shadow-2xl rounded-2xl overflow-hidden py-2 min-w-[120px]" :style="menuPosition">
                                <div v-for="s in talleList" :key="s" @click="chooseSize(s)" :class="['px-4 py-2 text-sm cursor-pointer transition flex justify-between items-center', selectedSize === s ? 'bg-gray-100 text-gray-800 font-bold' : 'text-gray-600 hover:bg-gray-50']">
                                    {{ s }} <span v-if="selectedSize === s">✓</span>
                                </div>
                                <div v-if="talleList?.length === 1" class="px-4 py-2 text-xs text-red-400 italic">Sin stock</div>
                            </div>
                            <div v-if="isSizeMenuOpen" @click="isSizeMenuOpen = false" class="fixed inset-0 z-[110]"></div>
                        </teleport>

                    </div>
                </template>
              </div>
            </div>

            <div class="border-t border-gray-100 pt-3 flex justify-between sm:justify-start items-center">
                <button @click="showOnlyOffers = !showOnlyOffers ; subirArriba()" :class="['flex items-center gap-2 px-3 py-1.5 rounded-lg text-xs font-bold transition border w-full sm:w-auto', showOnlyOffers ? 'bg-orange-100 text-orange-500 border-orange-200' : 'bg-white text-gray-500 border-gray-200 hover:bg-gray-50']">
                  <span class="flex items-center gap-1">🔥 Solo Ofertas</span>
                  <div :class="['w-8 h-4 rounded-full p-0.5 flex transition-all duration-400', showOnlyOffers ? 'bg-orange-500 justify-end' : 'bg-gray-400 justify-start']">
                    <div class="w-3 h-3 bg-white rounded-full shadow-sm"></div>
                  </div>
                </button>
            </div>

        </div>
        
        <div v-if="filteredProducts.length === 0" class="text-center py-10 text-gray-400">No hay productos.</div>
        
        <div class="grid grid-cols-2 lg:grid-cols-3 gap-3">
           <ProductCard 
              v-for="product in filteredProducts" 
              :key="product.id" 
              :product="product" 
              :user="user"
              @add-to-cart="addToCart"
              @edit="(p) => adminPanelRef.openEdit(p)"
              @delete="(id) => adminPanelRef.deleteProduct(id)"
              @image-click="openImageModal(product.image)"
           />
        </div>

        <div class="lg:hidden mt-8 pt-6 border-t border-gray-200 pb-10 space-y-4">
          <a :href="mapLink" target="_blank" class="block bg-white p-4 rounded-xl shadow-sm border border-gray-100 active:scale-95 transition">
                  <h3 class="font-bold text-gray-800 mb-2 text-sm flex justify-between items-center">
                      <span>📍 Retiros</span>
                      <span class="bg-sky-100 text-sky-600 px-2 py-0.5 rounded text-[10px]">Ir con GPS ↗</span>
                  </h3>
                  <div class="w-full h-40 bg-gray-200 rounded-lg overflow-hidden shadow-inner mb-2 relative">
                      <iframe src="https://www.google.com/maps/embed?pb=!1m10!1m8!1m3!1d2092.0525232936952!2d-67.54652908523627!3d-45.88601087901825!3m2!1i1024!2i768!4f13.1!5e1!3m2!1ses!2sar!4v1767656985269!5m2!1ses!2sar" width="100%" height="100%" style="border:0;" allowfullscreen="" loading="lazy" referrerpolicy="no-referrer-when-downgrade"></iframe>                      
                      <div class="absolute inset-0 flex items-center justify-center bg-black/5">
                          <span class="bg-white/90 px-3 py-1.5 rounded-full shadow text-xs font-bold text-gray-700">📍 Tocar para ir</span>
                      </div>
                  </div>
                  <p class="text-[10px] text-center text-gray-400">Te esperamos</p>
          </a>
        </div>
      </main>

      <aside class="hidden lg:block w-1/5 sticky top-24 h-fit transition-all space-y-3">
         <div class="bg-gradient-to-br from-orange-400 to-red-500 text-white p-4 rounded-2xl shadow-lg text-center transform hover:scale-105 transition duration-400">
            <p class="text-2xl mb-1">⏰</p>
            <h3 class="font-bold text-sm mb-0.5">Pedidos hasta</h3>
            <p class="text-3xl font-black my-1">18:00</p>
            <p class="text-[10px] text-white/80 leading-tight">Luego de esa hora pasan al día siguiente.</p>
         </div>
         <div class="bg-white p-4 rounded-2xl shadow-sm border border-gray-100">
            <h3 class="font-bold text-sm text-gray-700 mb-2">🚚 Envíos</h3>
            <div class="space-y-2 text-xs text-gray-400">
              <p class="flex gap-2"><span class="text-sky-500 font-bold">•</span> Costo según distancia.</p>
              <p class="flex gap-2"><span class="text-sky-500 font-bold">•</span> Coordinamos por WhatsApp.</p>
            </div>
         </div>
      </aside>

    </div>

    <div v-if="showTimeWarning && !user" class="lg:hidden fixed bottom-4 right-4 z-50 animate-bounce-in">
        <div class="bg-gradient-to-r from-orange-500 to-red-400 text-white pl-3 pr-8 py-2 rounded-full shadow-2xl flex items-center gap-2 relative">
            <span class="text-xl">⏰</span>
            <div class="flex flex-col leading-none">
                <span class="font-bold text-xs">Pedidos hasta 18:00</span>
                <span class="text-[9px] opacity-90">Luego pasan a mañana</span>
            </div>
            <button @click="showTimeWarning = false" class="absolute top-1 right-1 w-5 h-5 bg-white/20 hover:bg-white/40 rounded-full flex items-center justify-center text-[10px] text-white">✕</button>
        </div>
    </div>

    <footer class="mt-0 py-6 bg-gray-50 text-center text-xs text-gray-400 border-t w-full">
        <p class="mb-2">© 2026 Pañalera Gianluca</p>
        
        <button v-if="!user" @click="showLogin = true" class="hover:text-sky-400 underline">Soy Admin 🔒</button>
    </footer>
    <AdminPanel ref="adminPanelRef" :user="user" :products="products" @refresh="fetchProducts" />

    <div v-if="showLogin" class="fixed inset-0 bg-black/80 z-[60] flex items-center justify-center p-4">
       <div class="bg-white p-6 rounded-xl w-64 shadow-2xl">
         <h2 class="font-bold mb-4">Login</h2>
         <input v-model="email" placeholder="Email" class="w-full border p-2 rounded mb-2">
         <input v-model="password" type="password" placeholder="Pass" class="w-full border p-2 rounded mb-4">
         <button @click="handleLogin" class="bg-sky-400 text-white w-full py-2 rounded font-bold hover:bg-sky-700">Entrar</button>
         <button @click="showLogin=false" class="w-full mt-2 text-gray-500 text-sm">Cancelar</button>
       </div>
    </div>

    <div v-if="showRemoveModal" class="fixed inset-0 bg-black/50 z-[80] flex items-center justify-center p-4 animate-fade-in">
      <div class="bg-white rounded-xl p-6 shadow-2xl w-full max-w-xs text-center transform scale-100 transition-all">
          <div class="text-4xl mb-3">🗑️</div>
          <h3 class="font-bold text-lg mb-2 text-gray-800">¿Quitar producto?</h3>
          <p class="text-xs text-gray-500 mb-5 leading-relaxed">¿Estás segura que quieres sacar este artículo del carrito?</p>
          
          <div class="flex gap-3">
              <button @click="showRemoveModal=false" class="flex-1 bg-gray-100 text-gray-600 py-2.5 rounded-lg font-bold text-sm hover:bg-gray-200 transition">Cancelar</button>
              <button @click="confirmRemoveItem" class="flex-1 bg-red-500 text-white py-2.5 rounded-lg font-bold text-sm hover:bg-red-600 shadow-md transition transform active:scale-95">Sí, quitar</button>
          </div>
      </div>
    </div>

    <div v-if="showStockModal" class="fixed inset-0 bg-black/50 z-[90] flex items-center justify-center p-4 animate-fade-in">
      <div class="bg-white rounded-xl p-6 shadow-2xl w-full max-w-xs text-center transform scale-100 transition-all border-t-4 border-orange-400">
          <div class="text-4xl mb-3">✋</div>
          <h3 class="font-bold text-lg mb-2 text-gray-800">Stock Máximo</h3>
          <p class="text-sm text-gray-500 mb-5 leading-relaxed">
              ¡Ups! Solamente tenemos <b>{{ stockLimit }}</b> unidades disponibles de este producto.
          </p>
          
          <button @click="showStockModal=false" class="w-full bg-orange-400 text-white py-2.5 rounded-lg font-bold text-sm hover:bg-orange-500 shadow-md transition transform active:scale-95">
              Entendido
          </button>
      </div>
    </div>

    <div v-if="showEmptyCartModal" class="fixed inset-0 bg-black/50 z-[90] flex items-center justify-center p-4 animate-fade-in">
      <div class="bg-white rounded-xl p-6 shadow-2xl w-full max-w-xs text-center transform scale-100 transition-all border-t-4 border-gray-400">
          <div class="text-4xl mb-3">🛒</div>
          <h3 class="font-bold text-lg mb-2 text-gray-800">El carrito está vacío</h3>
          <p class="text-sm text-gray-500 mb-5 leading-relaxed">
              Agrega algunos productos antes de confirmar el pedido.
          </p>
          
          <button @click="showEmptyCartModal=false" class="w-full bg-gray-800 text-white py-2.5 rounded-lg font-bold text-sm hover:bg-gray-900 shadow-md transition transform active:scale-95">
              Ir a comprar
          </button>
      </div>
    </div>

    <div v-if="isCartOpen" class="fixed inset-0 z-50 flex justify-end">
      <div class="absolute inset-0 bg-black/50" @click="isCartOpen = false"></div>
      <div class="bg-white w-full max-w-md h-full relative z-10 flex flex-col shadow-2xl animate-slide-in">
        <div class="p-4 border-b flex justify-between items-center bg-gray-50"><h2 class="font-bold text-xl">Tu Carrito</h2><button @click="isCartOpen = false" class="text-2xl">×</button></div>
        <div class="flex-1 p-4 overflow-y-auto">
          <div v-if="cart.length===0" class="text-center mt-10 text-gray-400 flex flex-col items-center justify-center h-full"><span class="text-4xl grayscale opacity-50 mb-2">🛒</span>Vacío</div>
          
          <div v-for="(item, index) in cart" :key="index" class="flex items-center gap-3 mb-4 border-b pb-3">
             <img :src="item.image" @click="openImageModal(item.image)" class="w-14 h-14 rounded bg-gray-100 object-cover cursor-pointer hover:opacity-80 transition-opacity shadow-sm">
             <div class="flex-1"><p class="font-bold text-sm">{{ item.name }}</p><p class="text-sky-400 font-bold">{{ formatoMoneda(item.price * item.quantity) }}</p></div>
             
             <div class="flex items-center gap-2 bg-gray-100 rounded-lg p-1">
                <button @click="decrementQty(index)" class="w-6 h-6 flex items-center justify-center bg-white rounded shadow text-gray-400 hover:text-red-500 font-bold">-</button>
                <span class="text-sm font-bold w-4 text-center">{{ item.quantity }}</span>
                <button @click="incrementQty(item)" class="w-6 h-6 flex items-center justify-center bg-white rounded shadow text-gray-400 hover:text-sky-400 font-bold">+</button>
                <button @click="askToRemove(index)" class="ml-2 w-8 h-8 flex items-center justify-center bg-red-50 text-red-500 rounded-lg hover:bg-red-100 transition shadow-sm">
                  🗑
                </button>
              </div>
          </div>
        </div>
        <div class="p-4 border-t bg-gray-50"><button @click="checkout" class="w-full bg-green-500 text-white py-3 rounded-lg font-bold shadow-lg hover:bg-green-400 transition">Confirmar Pedido 📲</button></div>
      </div>
    </div>
  </div>
  <div v-if="showImageModal" class="fixed inset-0 bg-black/90 z-[150] flex items-center justify-center p-4 animate-fade-in" @click="showImageModal = false">
        
        <button @click="showImageModal = false" class="absolute top-6 right-6 text-white bg-white/20 hover:bg-white/40 rounded-full w-10 h-10 flex items-center justify-center text-xl transition shadow-lg z-10 backdrop-blur-sm">
            ✕
        </button>

        <img :src="selectedImage" class="max-w-full max-h-[90vh] object-contain rounded-2xl shadow-2xl animate-bounce-in" @click.stop>
        
    </div>
</template>

<style scoped>
.animate-slide-in { animation: slideIn 0.3s ease-out; } 
@keyframes slideIn { from { transform: translateX(100%); } to { transform: translateX(0); } }
.animate-bounce-in { animation: bounceIn 0.5s; } 
@keyframes bounceIn { 0% { transform: scale(0.3); opacity: 0; } 50% { transform: scale(1.05); opacity: 1; } 100% { transform: scale(1); } }

/* === BARRITA DE SCROLL MODERNA === */
.scroll-elegante {
  -webkit-overflow-scrolling: touch;
  /* ESTO LA EMPUJA MÁS ABAJO, SEPARÁNDOLA DE LOS BOTONES */
  padding-bottom: 12px; 
}

/* 1. Forzar visibilidad y grosor */
.scroll-elegante::-webkit-scrollbar {
  height: 14px; /* Un poquito más alta para que se vea excelente */
  -webkit-appearance: none !important;
  display: block !important; /* Obliga al celular a no esconderla NUNCA */
}

/* 2. Pista (el carril) */
.scroll-elegante::-webkit-scrollbar-track {
  background: #f1f5f9; 
  border-radius: 10px;
  margin: 0 2px;
}

/* 3. Barrita central (el pulgar) */
.scroll-elegante::-webkit-scrollbar-thumb {
  background: #cbd5e1; 
  border-radius: 10px;
  border: 3px solid #f1f5f9; /* Borde transparente falso para afinarla */
}

.scroll-elegante::-webkit-scrollbar-thumb:hover {
  background: #94a3b8; 
}

/* 4. Flecha Izquierda */
.scroll-elegante::-webkit-scrollbar-button:single-button:horizontal:decrement {
  display: block !important;
  width: 24px;
  background-color: #e2e8f0; /* Fondo gris para que parezca un botón real */
  border-radius: 10px 0 0 10px;
  background-image: url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='%2364748b'><path d='M15.41 16.59L10.83 12l4.58-4.59L14 6l-6 6 6 6 1.41-1.41z'/></svg>");
  background-size: 16px;
  background-repeat: no-repeat;
  background-position: center;
}

/* 5. Flecha Derecha */
.scroll-elegante::-webkit-scrollbar-button:single-button:horizontal:increment {
  display: block !important;
  width: 24px;
  background-color: #e2e8f0;
  border-radius: 0 10px 10px 0;
  background-image: url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='%2364748b'><path d='M8.59 16.59L13.17 12 8.59 7.41 10 6l6 6-6 6-1.41-1.41z'/></svg>");
  background-size: 16px;
  background-repeat: no-repeat;
  background-position: center;
}

/* Efecto hover en las flechitas */
.scroll-elegante::-webkit-scrollbar-button:single-button:hover {
  background-color: #cbd5e1;
}
</style>