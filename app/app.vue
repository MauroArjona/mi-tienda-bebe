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
const showOnlyOffers = ref(false) 

const categoriesList = ['Todos', 'Pañales', 'Shampoo', 'Toallitas', 'Jabón', 'Cremas', 'Accesorios']
const sizesList = ['Todos', 'RN', 'P', 'M', 'G', 'XG', 'XXG', 'XXXG'] 

// --- CARGAR DATOS ---
const fetchProducts = async () => {
  const { data } = await supabase.from('productos').select('*').order('created_at', { ascending: false })
  if (data) products.value = data
}
await fetchProducts()

// --- FILTRADO ---
const filteredProducts = computed(() => {
  let temp = [...products.value] 

  if (selectedCategory.value !== 'Todos') temp = temp.filter(p => p.category?.toLowerCase() === selectedCategory.value.toLowerCase())
  if (selectedSize.value !== 'Todos') temp = temp.filter(p => p.talle?.toUpperCase() === selectedSize.value.toUpperCase())
  if (searchQuery.value) temp = temp.filter(p => p.name.toLowerCase().includes(searchQuery.value.toLowerCase()))
  
  if (showOnlyOffers.value) {
    temp = temp.filter(p => p.is_promo === true)
  }

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
  cart.value.forEach(i => text += `*${i.quantity}x ${i.name}* - $${i.price*i.quantity}%0A`)
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
    <header class="bg-sky-400 text-white p-4 sticky top-0 z-40 shadow-lg flex justify-between items-center">
      <h1 class="text-xl font-bold flex items-center gap-2"> Pañalera Gianluca</h1>
      <div class="flex gap-2">
         <button v-if="user" @click="adminPanelRef.openNew()" class="bg-white text-sky-400 px-3 py-1 rounded text-sm font-bold shadow hover:bg-gray-100"> Agregar</button>
         <button v-if="user" @click="logout" class="text-xs bg-sky-800 px-2 rounded">Salir</button>
         <button @click="isCartOpen = !isCartOpen" class="relative p-2"><span class="text-2xl">🛒</span><span v-if="totalItemsCount" class="absolute top-0 right-0 bg-red-500 text-white text-xs font-bold rounded-full w-5 h-5 flex items-center justify-center">{{ totalItemsCount }}</span></button>
      </div>
    </header>

    <div class="max-w-[1400px] mx-auto flex gap-4 p-4 items-start mt-4">
      
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
        
        <div class="bg-white p-4 rounded-2xl shadow-sm border border-gray-100 mb-6 sticky top-20 z-30">
           <div class="relative mb-4">
             <span class="absolute left-3 top-3 text-gray-400">🔍</span>
             <input v-model="searchQuery" placeholder="Buscar..." class="w-full pl-10 pr-4 py-3 rounded-xl border bg-gray-50 focus:bg-white outline-none focus:border-sky-500 transition">
           </div>
           
           <div class="mb-4 overflow-x-auto pb-2 no-scrollbar">
             <div class="flex gap-2">
               <button v-for="c in categoriesList" :key="c" @click="selectedCategory=c" :class="['px-3 py-1 rounded-full text-xs font-bold transition whitespace-nowrap', selectedCategory===c?'bg-sky-400 text-white shadow':'bg-gray-100 text-gray-400 hover:bg-gray-200']">{{ c }}</button>
             </div>
           </div>

           <div class="flex flex-col sm:flex-row sm:items-center justify-between border-t border-gray-100 pt-3 gap-3">
                <div v-if="['Todos','Pañales'].includes(selectedCategory)" class="flex items-center gap-2 overflow-x-auto no-scrollbar w-full sm:w-auto pb-1 sm:pb-0">
                  <span class="text-xs font-bold text-gray-400 uppercase whitespace-nowrap">Talle:</span>
                  <button v-for="s in sizesList" :key="s" @click="selectedSize=s" :class="['px-2 py-1 rounded text-[10px] border transition min-w-[28px] text-center flex-shrink-0', selectedSize===s?'bg-gray-800 text-white border-gray-800':'bg-white text-gray-500 border-gray-200 hover:bg-gray-50']">{{ s }}</button>
                </div>
                <div v-else class="hidden sm:block flex-1"></div>
                
                <button @click="showOnlyOffers = !showOnlyOffers" :class="['flex items-center justify-between sm:justify-start gap-2 px-3 py-1.5 rounded-lg text-xs font-bold transition border w-full sm:w-auto', showOnlyOffers ? 'bg-orange-100 text-orange-400 border-orange-200' : 'bg-white text-gray-500 border-gray-200 hover:bg-gray-50']">
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
             <img :src="item.image" class="w-14 h-14 rounded bg-gray-100 object-cover">
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
</template>

<style scoped>
.animate-slide-in { animation: slideIn 0.3s ease-out; } @keyframes slideIn { from { transform: translateX(100%); } to { transform: translateX(0); } }
.animate-bounce-in { animation: bounceIn 0.5s; } @keyframes bounceIn { 0% { transform: scale(0.3); opacity: 0; } 50% { transform: scale(1.05); opacity: 1; } 100% { transform: scale(1); } }
.no-scrollbar::-webkit-scrollbar { display: none; }
</style>