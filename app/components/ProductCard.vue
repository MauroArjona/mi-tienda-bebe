<script setup>
import { computed } from 'vue'

const props = defineProps(['product', 'user'])
const emit = defineEmits(['add-to-cart', 'edit', 'delete'])

const formatoMoneda = (valor) => new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS', minimumFractionDigits: 0 }).format(valor)

// Calculamos porcentaje solo si hay precio viejo
const offPct = computed(() => {
  if (props.product.old_price && props.product.old_price > props.product.price) {
    return Math.round((1 - (props.product.price / props.product.old_price)) * 100)
  }
  return 0
})
</script>

<template>
  <div class="bg-white p-3 rounded-2xl shadow-sm border border-gray-100 relative group hover:shadow-xl transition-all duration-300 flex flex-col justify-between h-full">
    
    <div v-if="user" class="absolute top-2 right-2 flex gap-1 z-20">
      <button @click="$emit('edit', product)" class="bg-blue-500 text-white w-7 h-7 rounded-full shadow flex items-center justify-center hover:bg-blue-600 text-xs">✎</button>
      <button @click="$emit('delete', product.id)" class="bg-red-500 text-white w-7 h-7 rounded-full shadow flex items-center justify-center hover:bg-red-600 text-xs">🗑</button>
    </div>

    <div class="aspect-square bg-gray-100 rounded-xl mb-3 overflow-hidden relative w-full">
      <img :src="product.image || 'https://via.placeholder.com/150'" class="w-full h-full object-cover group-hover:scale-110 transition duration-500">
      
      <div v-if="!product.stock || product.stock <= 0" class="absolute inset-0 bg-white/60 flex items-center justify-center z-10">
        <span class="bg-red-500 text-white px-3 py-1 rounded-full text-xs font-bold shadow-lg transform -rotate-12">AGOTADO</span>
      </div>

      <div v-if="product.is_promo && offPct > 0" class="absolute top-2 left-2 bg-blue-500 text-white rounded-full w-11 h-11 flex flex-col items-center justify-center shadow-lg z-10 animate-bounce-in">
        <span class="text-xs font-black leading-none mt-0.5">{{ offPct }}%</span>
        <span class="text-[8px] font-bold uppercase leading-none mb-0.5">OFF</span>
      </div>
      <div v-else-if="product.is_promo" class="absolute top-0 left-0 bg-orange-500 text-white text-[10px] font-black px-2 py-1 rounded-br-lg shadow-md z-10">
        🔥 OFERTA
      </div>
    </div>

    <div class="flex-1 flex flex-col w-full">
        
        <div class="flex justify-between items-start gap-2 mb-2 w-full">
            <h3 class="font-bold text-gray-800 leading-tight text-sm uppercase break-words line-clamp-2 text-left">{{ product.name }}</h3>
            
            <div class="flex flex-row items-center gap-1 flex-shrink-0">
                <span class="text-[10px] font-bold text-sky-600 bg-sky-50 px-2 py-0.5 rounded border border-sky-100 tracking-wide">{{ product.category }}</span>
                <span v-if="product.talle" class="text-[10px] font-bold text-gray-500 bg-gray-100 px-2 py-0.5 rounded border border-gray-200">{{ product.talle }}</span>
            </div>
        </div>
        
        <div class="mt-auto pt-2 border-t border-dashed border-gray-100 flex items-end justify-between w-full">
            
            <div class="flex flex-col items-start leading-none">
                <span v-if="product.old_price" class="text-[11px] text-gray-400 line-through decoration-red-300 decoration-1 mb-1 block h-3 text-left">
                    {{ formatoMoneda(product.old_price) }}
                </span>
                <span v-else class="block h-4"></span>

                <span class="text-lg font-extrabold text-gray-900 text-left">{{ formatoMoneda(product.price) }}</span>
            </div>

            <div v-if="user" class="text-[10px] font-bold text-gray-400 bg-gray-50 px-2 py-1 rounded border border-gray-100 whitespace-nowrap">
              Stock: {{ product.stock || 0 }}
            </div>
        </div>
    </div>

    <button 
        @click="$emit('add-to-cart', product)" 
        :disabled="!product.stock || product.stock <= 0"
        :class="['mt-3 w-full py-2 rounded-lg text-sm font-bold transition flex items-center justify-center shadow-sm', 
        (!product.stock || product.stock <= 0) ? 'bg-gray-100 text-gray-400 cursor-not-allowed' : 'bg-sky-50 text-sky-600 hover:bg-sky-400 hover:text-white border border-sky-100']">
        {{ (!product.stock || product.stock <= 0) ? 'Sin Stock' : 'AGREGAR' }}
    </button>
  </div>
</template>

<style scoped>
.animate-bounce-in { animation: bounceIn 0.5s; } 
@keyframes bounceIn { 0% { transform: scale(0); } 80% { transform: scale(1.1); } 100% { transform: scale(1); } }
</style>