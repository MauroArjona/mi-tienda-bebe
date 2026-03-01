<script setup>
import { ref, watch, computed } from 'vue'
const supabase = useSupabaseClient()
const props = defineProps(['user', 'products']) 
const emit = defineEmits(['refresh', 'close'])

const showPanel = ref(false)
const activeTab = ref('productos')
const sales = ref([])
const showProductForm = ref(false)
const uploading = ref(false)

// VARIABLES NUEVAS PARA REPORTES
const reportView = ref('diario') // 'diario' o 'mensual'

// VARIABLES PARA VENTA MANUAL (CAJA)
const showManualSaleForm = ref(false)
const manualCart = ref([])
const selectedProductId = ref('')
const selectedQty = ref(1)

const categoriesList = ['Todos', 'Pañales', 'Toallitas', 'Cuidado Capilar','Oleos','Jabónes', 'Cremas','Guantes','Algodones','Apositos','Accesorios']
const sizesList = ['Todos', 'PR', 'RN', 'P', 'J','M', 'G', 'XG', 'XXG', 'XXXG'] 
const marcasList = ['Pampers', 'Huggies', 'Estrella', 'Babysec']  

const form = ref({ id: null, name: '', marca: '', price: '', old_price: null, image: '', category: 'Pañales', talle: '', stock: 0, is_promo: false })
 
const fetchSales = async () => {
  const { data } = await supabase.from('ventas').select('*').order('created_at', { ascending: false })
  if (data) sales.value = data
}

watch(showPanel, (newVal) => { if(newVal) fetchSales() })

// --- REPORTES ---
const reporteDiario = computed(() => {
  const reporte = {}
  sales.value.forEach(venta => {
    if (venta.estado === 'Cancelado') return
    const fecha = new Date(venta.created_at).toLocaleDateString('es-AR')
    if (!reporte[fecha]) reporte[fecha] = { total: 0, cantidad: 0 }
    reporte[fecha].total += venta.total
    reporte[fecha].cantidad += 1
  })
  return reporte
})

// NUEVO: REPORTE MENSUAL
const reporteMensual = computed(() => {
  const reporte = {}
  sales.value.forEach(venta => {
    if (venta.estado === 'Cancelado') return
    // Obtenemos "Enero 2026", "Diciembre 2025", etc.
    const fecha = new Date(venta.created_at)
    const mesAnio = fecha.toLocaleString('es-AR', { month: 'long', year: 'numeric' })
    const key = mesAnio.charAt(0).toUpperCase() + mesAnio.slice(1) // Capitalizar primera letra

    if (!reporte[key]) reporte[key] = { total: 0, cantidad: 0 }
    reporte[key].total += venta.total
    reporte[key].cantidad += 1
  })
  return reporte
})

const totalHistorico = computed(() => sales.value.filter(v => v.estado !== 'Cancelado').reduce((acc, curr) => acc + curr.total, 0))

const cambiarEstado = async (venta, nuevoEstado) => {
  if (nuevoEstado === 'Cancelado' && venta.estado !== 'Cancelado') {
    if (confirm('¿Devolver stock?')) {
      for (const item of venta.items) {
        const { data: prod } = await supabase.from('productos').select('stock').eq('id', item.id).single()
        if (prod) await supabase.from('productos').update({ stock: prod.stock + item.quantity }).eq('id', item.id)
      }
      await supabase.from('ventas').update({ estado: 'Cancelado' }).eq('id', venta.id)
    }
  } else {
    await supabase.from('ventas').update({ estado: nuevoEstado }).eq('id', venta.id)
  }
  await fetchSales()
  emit('refresh') 
}

const openNew = () => { form.value = { id: null, name: '', marca: '', price: '', old_price: null, image: '', category: 'Pañales', talle: '', stock: 0, is_promo: false }; showProductForm.value = true }
const openEdit = (p) => { form.value = { ...p }; showProductForm.value = true }

const uploadImage = (e) => {
  const file = e.target.files[0]
  if (!file) return
  uploading.value = true
  const reader = new FileReader()
  reader.readAsDataURL(file)
  reader.onload = (event) => {
    const img = new Image()
    img.src = event.target.result
    img.onload = () => {
      const canvas = document.createElement('canvas')
      const ctx = canvas.getContext('2d')
      const MAX_WIDTH = 800
      let width = img.width
      let height = img.height
      if (width > MAX_WIDTH) { height *= MAX_WIDTH / width; width = MAX_WIDTH }
      canvas.width = width; canvas.height = height
      ctx.drawImage(img, 0, 0, width, height)
      canvas.toBlob(async (blob) => {
        const fileName = `${Date.now()}.jpg`
        const { error } = await supabase.storage.from('fotos_productos').upload(fileName, blob, { contentType: 'image/jpeg', upsert: true })
        if (error) { alert('Error: ' + error.message); uploading.value = false; return }
        const { data } = supabase.storage.from('fotos_productos').getPublicUrl(fileName)
        form.value.image = data.publicUrl
        uploading.value = false
      }, 'image/jpeg', 0.7)
    }
  }
}

const saveProduct = async () => {
  const p = { ...form.value }
  if (p.category !== 'Pañales') p.talle = null
  if (!p.old_price) p.old_price = null
  if (!p.id) {
    delete p.id 
    const { error } = await supabase.from('productos').insert(p)
    if (error) return alert('Error: ' + error.message)
  } else {
    const { error } = await supabase.from('productos').update(p).eq('id', p.id)
    if (error) return alert('Error: ' + error.message)
  }
  showProductForm.value = false
  emit('refresh')
}

const deleteProduct = async (id) => { if (confirm('¿Borrar?')) { await supabase.from('productos').delete().eq('id', id); emit('refresh') } }

// --- LOGICA VENTA MANUAL ---
const addToManualCart = () => {
    if (!selectedProductId.value) return
    const prod = props.products.find(p => p.id === selectedProductId.value)
    if (!prod) return
    
    // Verificar si ya está en el carrito manual
    const existing = manualCart.value.find(i => i.id === prod.id)
    if (existing) {
        if (existing.quantity + selectedQty.value > prod.stock) return alert('No hay suficiente stock')
        existing.quantity += selectedQty.value
    } else {
        if (selectedQty.value > prod.stock) return alert('No hay suficiente stock')
        manualCart.value.push({ ...prod, quantity: selectedQty.value })
    }
    selectedProductId.value = ''
    selectedQty.value = 1
}

const removeFromManualCart = (index) => {
    manualCart.value.splice(index, 1)
}

const saveManualSale = async () => {
    if (manualCart.value.length === 0) return
    const total = manualCart.value.reduce((acc, item) => acc + (item.price * item.quantity), 0)

    const { error } = await supabase.from('ventas').insert({
        items: manualCart.value,
        total: total,
        estado: 'Entregado' 
    })
    
    if (error) return alert('Error al guardar venta: ' + error.message)

    for (const item of manualCart.value) {
        const nuevoStock = item.stock - item.quantity
        await supabase.from('productos').update({ stock: nuevoStock }).eq('id', item.id)
    }

    alert('¡Venta registrada con éxito!')
    manualCart.value = []
    showManualSaleForm.value = false
    await fetchSales()
    emit('refresh') 
}

const manualTotal = computed(() => manualCart.value.reduce((acc, item) => acc + (item.price * item.quantity), 0))
// ---------------------------

const formatoMoneda = (v) => new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS' }).format(v)
const fechaFormato = (f) => new Date(f).toLocaleString('es-AR')

defineExpose({ openEdit, deleteProduct, openNew })
</script>

<template>
  <div>
    <button v-if="user" @click="showPanel = true" class="fixed bottom-6 right-6 bg-gray-900/90 text-white p-4 rounded-full shadow-2xl hover:scale-110 transition z-40">⚙️</button>

    <div v-if="showPanel" class="fixed inset-0 z-50 bg-gray-100 flex flex-col animate-slide-up">
      <div class="bg-gray-900 text-white p-4 flex justify-between shadow-md items-center">
         <h2 class="font-bold flex gap-2 items-center">⚙️ Panel Admin</h2>
         <div class="flex gap-2 text-sm bg-gray-800 p-1 rounded-lg">
           <button @click="activeTab='productos'" :class="['px-3 py-1 rounded transition', activeTab==='productos'?'bg-gray-600 text-white':'text-gray-400 hover:text-white']">📦 Productos</button>
           <button @click="activeTab='ventas'" :class="['px-3 py-1 rounded transition', activeTab==='ventas'?'bg-gray-600 text-white':'text-gray-400 hover:text-white']">💰 Ventas</button>
           <button @click="activeTab='reportes'" :class="['px-3 py-1 rounded transition', activeTab==='reportes'?'bg-teal-600 text-white':'text-gray-400 hover:text-white']">📊 Reportes</button>
         </div>
         <button @click="showPanel = false" class="text-xl px-2">✕</button>
      </div>

      <div class="flex-1 overflow-y-auto p-6 max-w-5xl mx-auto w-full">
         <div v-if="activeTab === 'productos'">
            <div class="flex justify-between items-center mb-4">
               <h3 class="font-bold text-xl">Inventario</h3>
               <button @click="openNew" class="bg-green-600 text-white px-4 py-2 rounded font-bold shadow hover:bg-green-700">+ Nuevo Producto</button>
            </div>
            <div class="bg-white rounded-xl shadow overflow-hidden">
               <table class="w-full text-sm text-left">
                  <thead class="bg-gray-50 text-gray-500 uppercase text-xs"><tr><th class="p-3">Producto</th><th class="p-3">Precio</th><th class="p-3">Stock</th><th class="p-3">Acciones</th></tr></thead>
                  <tbody class="divide-y">
                     <tr v-for="p in products" :key="p.id" class="hover:bg-gray-50">
                        <td class="p-3 font-medium flex items-center gap-2">
                           <span v-if="p.is_promo" class="text-xs">🔥</span> {{ p.name }}
                        </td>
                        <td class="p-3 text-teal-600 font-bold">
                           {{ formatoMoneda(p.price) }}
                           <span v-if="p.old_price" class="text-xs text-gray-400 line-through ml-1">{{ p.old_price }}</span>
                        </td>
                        <td class="p-3"><span :class="['px-2 py-1 rounded text-xs font-bold', p.stock > 5 ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700']">{{ p.stock }} u.</span></td>
                        <td class="p-3 text-blue-600 cursor-pointer hover:underline" @click="openEdit(p)">Editar</td>
                     </tr>
                  </tbody>
               </table>
            </div>
         </div>

         <div v-if="activeTab === 'ventas'">
            <div class="flex justify-between items-center mb-4">
                <h3 class="font-bold text-xl">Últimos Pedidos</h3>
                <button @click="showManualSaleForm = true" class="bg-sky-500 text-white px-4 py-2 rounded font-bold shadow hover:bg-sky-600">
                    + Venta Manual (Local)
                </button>
            </div>
            
            <div v-if="sales.length === 0" class="text-center py-10 text-gray-400">No hay ventas registradas aún.</div>
            <div v-for="v in sales" :key="v.id" class="bg-white p-4 mb-4 rounded-xl shadow border-l-4" :class="v.estado==='Pendiente'?'border-yellow-400':(v.estado==='Cancelado'?'border-red-500':'border-green-500')">
               <div class="flex justify-between mb-2">
                  <span class="font-bold text-gray-700">Pedido #{{ v.id }} <span class="text-xs font-normal text-gray-400 ml-2">{{ fechaFormato(v.created_at) }}</span></span>
                  <span class="font-bold uppercase text-xs px-2 py-1 rounded" :class="v.estado==='Pendiente'?'bg-yellow-100 text-yellow-700':(v.estado==='Cancelado'?'bg-red-100 text-red-700':'bg-green-100 text-green-700')">{{ v.estado }}</span>
               </div>
               <div class="text-sm bg-gray-50 p-3 rounded mb-3 border border-gray-100">
                  <div v-for="i in v.items" :key="i.name" class="flex justify-between py-1 border-b border-gray-100 last:border-0">
                     <span>{{ i.quantity }}x {{ i.name }} {{ i.talle ? `(${i.talle})` : '' }}</span>
                     <span class="font-medium text-gray-600">${{ i.price * i.quantity }}</span>
                  </div>
                  <div class="font-black mt-2 text-right text-lg text-teal-700">{{ formatoMoneda(v.total) }}</div>
               </div>
               <div v-if="v.estado === 'Pendiente'" class="flex justify-end gap-2 text-xs font-bold text-white">
                  <button @click="cambiarEstado(v, 'Cancelado')" class="bg-red-400 px-3 py-2 rounded hover:bg-red-500 transition">Cancelar</button>
                  <button @click="cambiarEstado(v, 'Entregado')" class="bg-green-500 px-3 py-2 rounded hover:bg-green-600 transition">Confirmar</button>
               </div>
            </div>
         </div>

         <div v-if="activeTab === 'reportes'">
            <div class="flex justify-between items-center mb-4">
                 <h3 class="font-bold text-xl">Reportes</h3>
                 <div class="bg-white border rounded-lg p-1 flex">
                     <button @click="reportView='diario'" :class="['px-3 py-1 text-xs font-bold rounded transition', reportView==='diario' ? 'bg-teal-500 text-white' : 'text-gray-500']">Diario</button>
                     <button @click="reportView='mensual'" :class="['px-3 py-1 text-xs font-bold rounded transition', reportView==='mensual' ? 'bg-purple-500 text-white' : 'text-gray-500']">Mensual</button>
                 </div>
            </div>

            <div class="bg-gradient-to-r from-teal-500 to-green-500 text-white p-6 rounded-2xl shadow-lg mb-6 flex justify-between items-center">
               <div><p class="text-teal-100 font-bold uppercase text-xs mb-1">Total Histórico (Todos los tiempos)</p><p class="text-4xl font-black">{{ formatoMoneda(totalHistorico) }}</p></div>
               <div class="text-4xl opacity-30">💰</div>
            </div>

            <div v-if="reportView === 'diario'" class="bg-white rounded-xl shadow overflow-hidden animate-fade-in">
               <div class="bg-teal-50 p-3 border-b border-teal-100 font-bold text-teal-700 text-sm">📅 Detalle por Día</div>
               <table class="w-full text-sm text-left">
                  <thead class="bg-gray-50 text-gray-500 uppercase text-xs"><tr><th class="p-4">Fecha</th><th class="p-4 text-center">Pedidos</th><th class="p-4 text-right">Total</th></tr></thead>
                  <tbody class="divide-y">
                     <tr v-for="(datos, fecha) in reporteDiario" :key="fecha" class="hover:bg-gray-50">
                        <td class="p-4 font-bold text-gray-700">{{ fecha }}</td>
                        <td class="p-4 text-center"><span class="bg-blue-100 text-blue-800 px-2 py-1 rounded-full text-xs font-bold">{{ datos.cantidad }} ventas</span></td>
                        <td class="p-4 text-right font-black text-teal-600 text-lg">{{ formatoMoneda(datos.total) }}</td>
                     </tr>
                  </tbody>
               </table>
            </div>

            <div v-if="reportView === 'mensual'" class="bg-white rounded-xl shadow overflow-hidden animate-fade-in">
               <div class="bg-purple-50 p-3 border-b border-purple-100 font-bold text-purple-700 text-sm">📅 Resumen por Mes</div>
               <table class="w-full text-sm text-left">
                  <thead class="bg-gray-50 text-gray-500 uppercase text-xs"><tr><th class="p-4">Mes</th><th class="p-4 text-center">Pedidos</th><th class="p-4 text-right">Total Mes</th></tr></thead>
                  <tbody class="divide-y">
                     <tr v-for="(datos, mes) in reporteMensual" :key="mes" class="hover:bg-gray-50">
                        <td class="p-4 font-bold text-gray-700 capitalize">{{ mes }}</td>
                        <td class="p-4 text-center"><span class="bg-purple-100 text-purple-800 px-2 py-1 rounded-full text-xs font-bold">{{ datos.cantidad }} ventas</span></td>
                        <td class="p-4 text-right font-black text-purple-600 text-lg">{{ formatoMoneda(datos.total) }}</td>
                     </tr>
                  </tbody>
               </table>
            </div>
         </div>
      </div>
    </div>

    <div v-if="showProductForm" class="fixed inset-0 bg-black/80 z-[60] flex items-center justify-center p-4">
       <div class="bg-white w-full max-w-sm rounded-2xl p-6 overflow-y-auto max-h-[90vh]">
          <h2 class="font-bold mb-4 text-lg">{{ form.id ? 'Editar' : 'Nuevo' }} Producto</h2>
          <label class="text-xs font-bold text-gray-500 uppercase mb-1 block">Nombre</label>
          <input v-model="form.name" class="w-full border-2 border-gray-100 p-2 rounded-lg mb-3 focus:border-teal-500 outline-none transition">
            <div v-if="form.category === 'Pañales'" class="mb-3 animate-fade-in">
               <label class="text-xs font-bold text-sky-600 uppercase mb-1 block">Marca</label>
               <select v-model="form.marca" class="w-full border-2 border-sky-100 p-2 rounded-lg bg-sky-50 text-sky-800 focus:border-sky-500 outline-none">
                  <option value="">- Seleccionar Marca -</option>
                  <option v-for="m in marcasList" :key="m" :value="m">{{ m }}</option>
               </select>
            </div>

          <div class="flex gap-2 mb-3">
             <div class="w-1/2">
                <label class="text-xs font-bold text-gray-500 uppercase mb-1 block">Precio Venta</label>
                <input v-model="form.price" type="number" class="w-full border-2 border-gray-100 p-2 rounded-lg focus:border-teal-500 outline-none font-bold">
             </div>
             <div class="w-1/2">
                <label class="text-xs font-bold text-gray-400 uppercase mb-1 block">Antes (Tachado)</label>
                <input v-model="form.old_price" type="number" placeholder="Opcional" class="w-full border-2 border-gray-100 p-2 rounded-lg focus:border-teal-500 outline-none text-gray-500">
             </div>
          </div>
          <label class="text-xs font-bold text-gray-500 uppercase mb-1 block">Stock</label>
          <input v-model="form.stock" type="number" class="w-full border-2 border-gray-100 p-2 rounded-lg focus:border-teal-500 outline-none mb-3">
          <div class="flex items-center gap-3 mb-4 bg-orange-50 p-3 rounded-xl border border-orange-100 cursor-pointer select-none" @click="form.is_promo = !form.is_promo">
             <div :class="['w-12 h-6 rounded-full flex items-center p-1 transition-all duration-300', form.is_promo ? 'bg-orange-500 justify-end' : 'bg-gray-300 justify-start']">
                <div class="w-4 h-4 bg-white rounded-full shadow-md"></div>
             </div>
             <div><p class="font-bold text-sm text-gray-700">🔥 ¡Es una Oferta!</p><p class="text-[10px] text-gray-500">Aparecerá primera en la lista.</p></div>
          </div>
          <label class="text-xs font-bold text-gray-500 uppercase mb-1 block">Categoría</label>
          <select v-model="form.category" class="w-full border-2 border-gray-100 p-2 rounded-lg mb-3 bg-white focus:border-teal-500 outline-none">
             <option v-for="c in categoriesList" :key="c">{{ c }}</option>
          </select>
          <div v-if="form.category === 'Pañales'" class="mb-3 animate-fade-in">
             <label class="text-xs font-bold text-teal-600 uppercase mb-1 block">Talle</label>
             <select v-model="form.talle" class="w-full border-2 border-teal-100 p-2 rounded-lg bg-teal-50 text-teal-800 focus:border-teal-500 outline-none">
                <option value="">-</option><option v-for="t in sizesList" :key="t">{{ t }}</option>
             </select>
          </div>
          <label class="text-xs font-bold text-gray-500 uppercase mb-1 block">Foto del Producto</label>
          <label class="block w-full border-2 border-dashed border-gray-300 rounded-xl p-4 text-center cursor-pointer hover:bg-gray-50 hover:border-teal-400 transition mb-3 relative group">
             <input type="file" accept="image/*" @change="uploadImage" class="hidden">
             <div v-if="!uploading" class="text-gray-400 group-hover:text-teal-600 transition"><p class="text-2xl mb-1">📸</p><p class="text-xs font-bold">Subir foto</p></div>
             <div v-else class="text-teal-600 animate-pulse font-bold text-sm">Subiendo...</div>
          </label>
          <div v-if="form.image" class="mb-4 flex justify-center"><img :src="form.image" class="h-24 rounded-lg shadow-sm border border-gray-200 object-cover"></div>
          <button @click="saveProduct" :disabled="uploading" class="w-full bg-green-600 hover:bg-green-700 text-white py-3 rounded-xl font-bold shadow-lg transition transform active:scale-95">Guardar Producto</button>
          <button @click="showProductForm=false" class="w-full mt-3 text-gray-400 hover:text-gray-600 font-medium py-2">Cancelar</button>
       </div>
    </div>

    <div v-if="showManualSaleForm" class="fixed inset-0 bg-black/80 z-[60] flex items-center justify-center p-4">
        <div class="bg-white w-full max-w-sm rounded-2xl p-6 overflow-y-auto max-h-[90vh]">
            <h2 class="font-bold mb-4 text-lg">💰 Nueva Venta (Local)</h2>
            <div class="flex gap-2 mb-2">
                <select v-model="selectedProductId" class="w-3/4 border-2 border-gray-100 p-2 rounded-lg bg-white outline-none text-sm">
                    <option value="" disabled selected>Seleccionar producto...</option>
                    <option v-for="p in products" :key="p.id" :value="p.id" :disabled="p.stock <= 0">
                        {{ p.name }} ({{ p.stock }} u.)
                    </option>
                </select>
                <input v-model="selectedQty" type="number" min="1" class="w-1/4 border-2 border-gray-100 p-2 rounded-lg outline-none text-center font-bold">
            </div>
            <button @click="addToManualCart" class="w-full bg-sky-100 text-sky-600 font-bold py-2 rounded-lg mb-4 text-sm hover:bg-sky-200">+ Agregar a la lista</button>

            <div v-if="manualCart.length > 0" class="bg-gray-50 rounded-lg p-2 mb-4 max-h-40 overflow-y-auto">
                <div v-for="(item, idx) in manualCart" :key="idx" class="flex justify-between items-center text-sm border-b border-gray-200 last:border-0 py-2">
                    <div>
                        <p class="font-bold">{{ item.name }}</p>
                        <p class="text-xs text-gray-500">{{ item.quantity }} x {{ formatoMoneda(item.price) }}</p>
                    </div>
                    <div class="flex items-center gap-2">
                        <span class="font-bold text-gray-700">{{ formatoMoneda(item.price * item.quantity) }}</span>
                        <button @click="removeFromManualCart(idx)" class="text-red-500 font-bold px-1">✕</button>
                    </div>
                </div>
            </div>
            <div v-else class="text-center text-gray-400 text-sm mb-4 italic">Lista vacía</div>

            <div class="flex justify-between items-center mb-4 text-lg font-black text-teal-600 border-t pt-2">
                <span>Total:</span>
                <span>{{ formatoMoneda(manualTotal) }}</span>
            </div>

            <button @click="saveManualSale" :disabled="manualCart.length === 0" :class="['w-full py-3 rounded-xl font-bold shadow-lg transition', manualCart.length > 0 ? 'bg-green-600 text-white hover:bg-green-700' : 'bg-gray-300 text-gray-500 cursor-not-allowed']">
                ✅ Cobrar y Guardar
            </button>
            <button @click="showManualSaleForm=false" class="w-full mt-3 text-gray-400 hover:text-gray-600 font-medium py-2">Cancelar</button>
        </div>
    </div>

  </div>
</template>

<style scoped>
.animate-slide-up { animation: slideUp 0.3s ease-out; } 
@keyframes slideUp { from { transform: translateY(100%); } to { transform: translateY(0); } }
.animate-fade-in { animation: fadeIn 0.3s ease-out; } 
@keyframes fadeIn { from { opacity: 0; transform: translateY(-5px); } to { opacity: 1; transform: translateY(0); } }
</style>