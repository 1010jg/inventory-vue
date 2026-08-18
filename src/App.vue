<template>
  <main class="warehouse min-h-screen p-5 md:p-10">
    <div class="dot-field" aria-hidden="true"></div>

    <section class="relative mx-auto max-w-6xl">
      <!-- Manifest header -->
      <header class="mb-8 flex flex-col gap-4 rounded-2xl bg-[#14161f] p-8 text-white md:flex-row md:items-end md:justify-between">
        <div>
          <p class="eyebrow text-[#d98e2b]">Warehouse Manifest · No. 004</p>
          <h1 class="display mt-1 text-4xl leading-none">คลังสินค้า</h1>
          <p class="mono mt-2 text-xs tracking-wide text-white/40">STOCK CONTROL LEDGER — UPDATED LIVE</p>
        </div>
        <div class="stat-chip">
          <span class="mono text-2xl font-semibold">{{ products.length }}</span>
          <span class="eyebrow text-white/50">SKUs Tracked</span>
        </div>
      </header>

      <!-- Reorder stamp -->
      <div v-if="lowStock.length" class="stamp mb-7">
        <span class="stamp-badge">⚠ Reorder</span>
        <p class="mono text-sm leading-snug">
          มีสินค้าใกล้หมด <b>{{ lowStock.length }}</b> รายการ — โปรดสั่งเพิ่มก่อนสต็อกขาด
        </p>
      </div>

      <!-- Intake ticket / form -->
      <form @submit.prevent="save" class="intake-ticket">
        <div class="field">
          <label class="field-label">SKU</label>
          <input v-model="form.sku" placeholder="ATM-0042" required />
        </div>
        <div class="field md:col-span-2">
          <label class="field-label">ชื่อสินค้า</label>
          <input v-model="form.name" placeholder="เช่น เมล็ดกาแฟคั่วกลาง 1kg" required />
        </div>
        <div class="field">
          <label class="field-label">ราคา (บาท)</label>
          <input v-model.number="form.price" type="number" min="0" placeholder="0" required />
        </div>
        <div class="field">
          <label class="field-label">คงเหลือ</label>
          <input v-model.number="form.stock" type="number" min="0" placeholder="0" required />
        </div>
        <div class="field">
          <label class="field-label">จุดสั่งซื้อ</label>
          <input v-model.number="form.reorderPoint" type="number" min="0" placeholder="5" />
        </div>

        <div class="flex items-end gap-3 md:col-span-1">
          <button type="submit" class="submit-btn">
            {{ editId ? 'บันทึกการแก้ไข' : '+ เพิ่มสินค้า' }}
          </button>
          <button v-if="editId" type="button" @click="cancelEdit" class="cancel-btn">ยกเลิก</button>
        </div>
      </form>

      <!-- Empty state -->
      <div v-if="!products.length" class="empty-state">
        <p class="eyebrow text-[#5b6072]">Ledger Empty</p>
        <p class="mono mt-1 text-sm text-[#5b6072]">ยังไม่มีสินค้าในคลัง — เริ่มเพิ่มรายการแรกด้านบน</p>
      </div>

      <!-- Product tags -->
      <div class="tag-grid">
        <article
          v-for="x in products"
          :key="x._id"
          class="tag-card"
          :class="{ 'tag-card--low': x.stock <= x.reorderPoint }"
        >
          <span v-if="x.stock <= x.reorderPoint" class="corner-ribbon">Reorder</span>
          <div class="tag-hole"></div>

          <div class="tag-head">
            <p class="mono text-[11px] tracking-wide text-[#5b6072]">{{ x.sku }}</p>
            <h2 class="display text-lg leading-tight">{{ x.name }}</h2>
          </div>

          <div class="tear-line"></div>

          <div class="tag-body">
            <div class="flex items-baseline justify-between">
              <span class="eyebrow text-[#5b6072]">Price</span>
              <span class="mono text-lg font-semibold text-[#14161f]">฿{{ x.price.toLocaleString() }}</span>
            </div>
            <div class="mt-2 flex items-baseline justify-between">
              <span class="eyebrow text-[#5b6072]">Stock</span>
              <span
                class="mono text-lg font-semibold"
                :class="x.stock <= x.reorderPoint ? 'text-[#b23a2e]' : 'text-[#3f7a5c]'"
              >
                {{ x.stock }} <span class="text-xs font-normal text-[#5b6072]">/ {{ x.reorderPoint }} min</span>
              </span>
            </div>

            <p v-if="x.stock <= x.reorderPoint" class="low-note">⚠ สินค้าใกล้หมด โปรดสั่งเพิ่ม</p>

            <div class="mt-4 flex items-center justify-between">
              <div class="barcode" aria-hidden="true">
                <span
                  v-for="(w, i) in barcodeBars(x.sku)"
                  :key="i"
                  :style="{ width: w + 'px' }"
                ></span>
              </div>
              <div class="actions">
                <button @click="edit(x)" class="edit-link">แก้ไข</button>
                <button @click="remove(x._id)" class="delete-link">ลบ</button>
              </div>
            </div>
          </div>
        </article>
      </div>
    </section>
  </main>
</template>

<script setup>
import { computed, onMounted, ref } from 'vue'

const API = 'http://localhost:3001/api/products'
const products = ref([])
const editId = ref(null)
const blank = () => ({ sku: '', name: '', price: 0, stock: 0, reorderPoint: 5 })
const form = ref(blank())

const load = async () => (products.value = await fetch(API).then((r) => r.json()))

const save = async () => {
  await fetch(editId.value ? `${API}/${editId.value}` : API, {
    method: editId.value ? 'PUT' : 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(form.value),
  })
  editId.value = null
  form.value = blank()
  load()
}

const edit = (x) => {
  editId.value = x._id
  form.value = { ...x }
}

const cancelEdit = () => {
  editId.value = null
  form.value = blank()
}

const remove = async (id) => {
  if (confirm('ลบสินค้า?')) {
    await fetch(`${API}/${id}`, { method: 'DELETE' })
    load()
  }
}

onMounted(load)

const lowStock = computed(() => products.value.filter((x) => x.stock <= x.reorderPoint))

// Purely decorative barcode generated deterministically from the SKU string
const barcodeBars = (sku) => {
  const src = (sku || 'SKU').padEnd(18, 'x')
  return Array.from(src).map((ch) => (ch.charCodeAt(0) % 3) + 1)
}
</script>