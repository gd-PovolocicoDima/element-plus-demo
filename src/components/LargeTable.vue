<template>
  <div style="padding: 1rem;">
    <h2>Element Plus Table V2 — 3000 строк с чекбоксами, фильтрацией, сортировкой, нумерацией и пагинацией</h2>

    <el-table-v2
      ref="tableRef"
      v-model:sort-state="sortState"
      :columns="columns"
      :data="pagedData"
      row-key="id"
      :estimated-row-height="44"
      :width="900"
      :height="800"
      @column-sort="onSort"
    >
    </el-table-v2>

    <div style="margin-top: 1rem; text-align: right;">
      <el-pagination
        background
        :page-size="rowsPerPage"
        :current-page="currentPage"
        :total="visibleData.length"
        layout="sizes, prev, pager, next, ->, total"
        :page-sizes="[20, 50, 80, 100]"
        @current-change="onPageChange"
        @size-change="onSizeChange"
      />
    </div>

    <div style="margin-top: 1rem;">
      <strong>Выбранные ID:</strong>
      <span v-if="selectedRows.length === 0">— ни одна строка не выбрана —</span>
      <ul v-else style="padding-left: 1rem;">
        <li v-for="row in selectedRows" :key="row.id">{{ row.id }}</li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, h } from 'vue'
import { ElCheckbox, ElInput, ElTag } from 'element-plus'
import { TableV2SortOrder } from 'element-plus'

const brands = [
  'Toyota','Ford','Chevrolet','Honda','Nissan',
  'BMW','Mercedes','Volkswagen','Hyundai','Kia'
]
const colors = [
  'Black','White','Silver','Blue','Red',
  'Green','Yellow','Gray','Orange','Brown'
]

function randomVin(len = 5) {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
  let s = ''
  for (let i = 0; i < len; i++) {
    s += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return s
}

function randomYear(min = 1990, max = 2022) {
  return Math.floor(Math.random() * (max - min + 1)) + min
}

function generateCar(id) {
  return {
    id,
    vin: randomVin(),
    brand: brands[Math.floor(Math.random() * brands.length)],
    color: colors[Math.floor(Math.random() * colors.length)],
    year: randomYear()
  }
}

const originalData = ref([])
const selectedRows = ref([])
const selectedSet = ref(new Set())

const currentPage = ref(1)
const rowsPerPage = ref(80)

const sortState = ref({})

const vinFilter = ref('')
const brandFilter = ref('')
const colorFilter = ref('')
const yearFilter = ref('')

onMounted(() => {
  const arr = []
  for (let i = 1; i <= 3000; i++) {
    arr.push(generateCar(i))
  }
  originalData.value = arr
})

const visibleData = computed(() => {
  let data = originalData.value.slice()

  if (vinFilter.value.trim()) {
    const sub = vinFilter.value.trim().toLowerCase()
    data = data.filter(r => r.vin.toLowerCase().includes(sub))
  }
  if (brandFilter.value.trim()) {
    const sub = brandFilter.value.trim().toLowerCase()
    data = data.filter(r => r.brand.toLowerCase().includes(sub))
  }
  if (colorFilter.value.trim()) {
    const sub = colorFilter.value.trim().toLowerCase()
    data = data.filter(r => r.color.toLowerCase().includes(sub))
  }
  if (yearFilter.value.trim()) {
    const sub = yearFilter.value.trim()
    data = data.filter(r => String(r.year).startsWith(sub))
  }

  const entry = Object.entries(sortState.value).find(([, o]) => o === 'asc' || o === 'desc')
  if (entry) {
    const [key, order] = entry
    data.sort((a, b) => {
      const av = a[key]
      const bv = b[key]
      if (av < bv) return order === TableV2SortOrder.ASC ? -1 : 1
      if (av > bv) return order === TableV2SortOrder.ASC ? 1 : -1
      return 0
    })
  }

  return data
})

const pagedData = computed(() => {
  const start = (currentPage.value - 1) * rowsPerPage.value
  return visibleData.value.slice(start, start + rowsPerPage.value)
})

function isRowChecked(id) {
  return selectedSet.value.has(id)
}
function toggleSelectAll(val) {
  if (val) {
    const all = visibleData.value
    selectedRows.value = all.slice()
    selectedSet.value = new Set(all.map(r => r.id))
  } else {
    selectedRows.value = []
    selectedSet.value = new Set()
  }
}
function toggleRowSelection(rowData, val) {
  if (!rowData) return
  if (val) {
    if (!selectedSet.value.has(rowData.id)) {
      selectedRows.value.push(rowData)
      selectedSet.value.add(rowData.id)
    }
  } else {
    selectedSet.value.delete(rowData.id)
    selectedRows.value = selectedRows.value.filter(r => r.id !== rowData.id)
  }
}

function onSort({ key, order }) {
  if (order) {
    sortState.value = { [key]: order }
  } else {
    sortState.value = {}
  }
  currentPage.value = 1
}
function onPageChange(page) {
  currentPage.value = page
}
function onSizeChange(size) {
  rowsPerPage.value = size
  currentPage.value = 1
}

function createHeaderRenderer(label, dataKey, filterValue, onFilterChange) {
  return () => {
    const currentSort = sortState.value[dataKey]
    const isAsc = currentSort === TableV2SortOrder.ASC
    const isDesc = currentSort === TableV2SortOrder.DESC
    const nextOrder = isAsc ? TableV2SortOrder.DESC : TableV2SortOrder.ASC

    const headerLabel = h(
      'div',
      {
        style: 'display:flex; align-items:center; justify-content:center; gap:4px; cursor:pointer; user-select:none; font-weight:bold; font-size:12px;',
        onClick: () => {
          onSort({ key: dataKey, order: nextOrder })
        }
      },
      [
        h('span', {}, () => label),
        isAsc
          ? h('span', { style: 'font-size:10px;' }, () => '▲')
          : isDesc
          ? h('span', { style: 'font-size:10px;' }, () => '▼')
          : null
      ]
    )

    const filterInput = h(ElInput, {
      modelValue: filterValue.value,
      placeholder: `Фильтр ${label}...`,
      size: 'small',
      clearable: true,
      style: 'width: 100%; margin-top:4px;',
      onInput: val => {
        onFilterChange(val)
        currentPage.value = 1
      }
    })

    return [headerLabel, filterInput]
  }
}

const columns = [
  {
    key: 'col-index',
    dataKey: 'id',
    title: '№',
    width: 60,
    sortable: false,
    headerCellRenderer: () =>
      h('div', { style: 'font-weight:bold; font-size:12px; text-align:center;' }, () => '#'),
    cellRenderer: ({ rowIndex }) => {
      const globalIdx = (currentPage.value - 1) * rowsPerPage.value + rowIndex + 1
      return h('span', { style: 'font-size:12px;' }, () => String(globalIdx))
    }
  },
  {
    key: 'selection-id',
    dataKey: 'id',
    title: '',
    width: 100,
    fixed: 'left',
    sortable: false,
    headerCellRenderer: () => {
      const total = visibleData.value.length
      const cnt = selectedRows.value.length
      const allChecked = total > 0 && cnt === total
      const someChecked = cnt > 0 && cnt < total

      return h(
        'div',
        { style: 'display:flex; align-items:center; justify-content:center; gap:4px;' },
        [
          h(ElCheckbox, {
            modelValue: allChecked,
            indeterminate: someChecked,
            'onUpdate:modelValue': v => toggleSelectAll(v)
          }),
          h('span', { style: 'font-weight:bold; font-size:12px;' }, () => 'ID')
        ]
      )
    },
    cellRenderer: ({ rowData }) => {
      if (!rowData) return h('div')
      const checked = isRowChecked(rowData.id)
      return h(
        'div',
        { style: 'display:flex; align-items:center; justify-content:center; gap:4px;' },
        [
          h(ElCheckbox, {
            modelValue: checked,
            'onUpdate:modelValue': v => toggleRowSelection(rowData, v)
          }),
          h('span', { style: 'font-size:12px;' }, () => String(rowData.id))
        ]
      )
    }
  },
  {
    key: 'col-vin',
    dataKey: 'vin',
    title: 'VIN',
    width: 180,
    sortable: true,
    headerCellRenderer: createHeaderRenderer('VIN', 'vin', vinFilter, v => (vinFilter.value = v))
  },
  {
    key: 'col-brand',
    dataKey: 'brand',
    title: 'Brand',
    width: 160,
    sortable: true,
    headerCellRenderer: createHeaderRenderer('Brand', 'brand', brandFilter, v => (brandFilter.value = v))
  },
  {
    key: 'col-color',
    dataKey: 'color',
    title: 'Color',
    width: 140,
    sortable: true,
    headerCellRenderer: createHeaderRenderer('Color', 'color', colorFilter, v => (colorFilter.value = v)),
    cellRenderer: ({ rowData }) => {
      if (!rowData) return h('div')
      return h(
        ElTag,
        {
          size: 'small',
          style: {
            backgroundColor: rowData.color.toLowerCase(),
            color: '#fff',
            padding: '2px 8px',
            fontSize: '12px'
          }
        },
        () => rowData.color
      )
    }
  },
  {
    key: 'col-year',
    dataKey: 'year',
    title: 'Year',
    width: 100,
    sortable: true,
    headerCellRenderer: createHeaderRenderer('Year', 'year', yearFilter, v => (yearFilter.value = v))
  }
]
</script>

<style scoped>
h2 {
  margin-bottom: 1rem;
}
.el-table-v2__header-cell:hover {
  background-color: #f5f7fa;
}
</style>
