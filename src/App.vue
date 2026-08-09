<script setup>
import { ref, computed } from 'vue'


const TOTAL_BLOCKS = 32
const BLOCK_SIZE_KB = 2


const createInitialHeap = () => {
  return Array.from({ length: TOTAL_BLOCKS }, (_, i) => ({
    id: i,
    address: `0x${(i * 0x800).toString(16).padStart(4, '0').toUpperCase()}`,
    allocatedTo: null, 
    status: 'FREE' 
  }))
}


const heap = ref(createInitialHeap())
const stackPointers = ref([
  { id: 'ptr-1', name: 'userSession', targetAllocId: 'alloc-101' },
  { id: 'ptr-2', name: 'cacheBuffer', targetAllocId: 'alloc-102' }
])

const allocations = ref([
  { id: 'alloc-101', name: 'UserObject', blocks: [0, 1, 2], sizeKb: 6, refCount: 1 },
  { id: 'alloc-102', name: 'ImageData', blocks: [4, 5, 6, 7], sizeKb: 8, refCount: 1 }
])


heap.value[0].allocatedTo = 'alloc-101'; heap.value[0].status = 'ACTIVE'
heap.value[1].allocatedTo = 'alloc-101'; heap.value[1].status = 'ACTIVE'
heap.value[2].allocatedTo = 'alloc-101'; heap.value[2].status = 'ACTIVE'

heap.value[4].allocatedTo = 'alloc-102'; heap.value[4].status = 'ACTIVE'
heap.value[5].allocatedTo = 'alloc-102'; heap.value[5].status = 'ACTIVE'
heap.value[6].allocatedTo = 'alloc-102'; heap.value[6].status = 'ACTIVE'
heap.value[7].allocatedTo = 'alloc-102'; heap.value[7].status = 'ACTIVE'


const inputVarName = ref('')
const inputSizeBlocks = ref(2)
const isGCRunning = ref(false)


const memoryLogs = ref([
  ' DevMem Engine initialized. 64 KB virtual heap mapped across 32 pages.'
])


const totalAllocatedBlocks = computed(() => {
  return heap.value.filter(b => b.status !== 'FREE').length
})

const memoryUsagePercent = computed(() => {
  return Math.round((totalAllocatedBlocks.value / TOTAL_BLOCKS) * 100)
})

const reclaimableGarbageBytes = computed(() => {
  const garbageCount = heap.value.filter(b => b.status === 'GARBAGE').length
  return garbageCount * BLOCK_SIZE_KB
})


const handleMalloc = () => {
  const varName = inputVarName.value.trim().replace(/\s+/g, '_')
  const requestedBlocks = Number(inputSizeBlocks.value)
  if (!varName || requestedBlocks <= 0) return

 
  let freeStartIndex = -1
  let consecutiveFree = 0

  for (let i = 0; i < TOTAL_BLOCKS; i++) {
    if (heap.value[i].status === 'FREE') {
      if (consecutiveFree === 0) freeStartIndex = i
      consecutiveFree++
      if (consecutiveFree === requestedBlocks) break
    } else {
      consecutiveFree = 0
      freeStartIndex = -1
    }
  }

  if (consecutiveFree < requestedBlocks) {
    memoryLogs.value.unshift(
      ` OUT OF MEMORY (OOM): Failed to allocate ${requestedBlocks * BLOCK_SIZE_KB} KB for '${varName}'. Heap fragmented.`
    )
    return
  }

  const allocId = `alloc-${Date.now().toString().slice(-4)}`
  const assignedBlocks = []

  for (let i = freeStartIndex; i < freeStartIndex + requestedBlocks; i++) {
    heap.value[i].allocatedTo = allocId
    heap.value[i].status = 'ACTIVE'
    assignedBlocks.push(i)
  }

  allocations.value.push({
    id: allocId,
    name: varName,
    blocks: assignedBlocks,
    sizeKb: requestedBlocks * BLOCK_SIZE_KB,
    refCount: 1
  })

  stackPointers.value.push({
    id: `ptr-${Date.now().toString().slice(-4)}`,
    name: varName,
    targetAllocId: allocId
  })

  memoryLogs.value.unshift(
    ` MALLOC SUCCESS: Allocated ${requestedBlocks * BLOCK_SIZE_KB} KB at address ${heap.value[freeStartIndex].address} [Ref: ${varName}]`
  )
  inputVarName.value = ''
}


const nullifyPointer = (ptrId) => {
  const ptrIndex = stackPointers.value.findIndex(p => p.id === ptrId)
  if (ptrIndex === -1) return

  const targetPtr = stackPointers.value[ptrIndex]
  const targetAlloc = allocations.value.find(a => a.id === targetPtr.targetAllocId)

  if (targetAlloc) {
    targetAlloc.refCount--
    memoryLogs.value.unshift(
      ` POINTER SEVERED: Removed reference '${targetPtr.name}'. Object '${targetAlloc.name}' refCount is now ${targetAlloc.refCount}.`
    )

   
    if (targetAlloc.refCount <= 0) {
      targetAlloc.blocks.forEach(blockIdx => {
        heap.value[blockIdx].status = 'GARBAGE'
      })
      memoryLogs.value.unshift(
        ` UNREFERENCED GARBAGE DETECTED: Object '${targetAlloc.name}' is unreachable from Stack Root.`
      )
    }
  }

  stackPointers.value.splice(ptrIndex, 1)
}


const runGarbageCollector = () => {
  if (isGCRunning.value) return
  isGCRunning.value = true
  memoryLogs.value.unshift(' GARBAGE COLLECTOR TRIGGERED: Starting Mark-and-Sweep cycle...')

  setTimeout(() => {
    let reclaimedBlocks = 0

   
    heap.value.forEach(block => {
      if (block.status === 'GARBAGE') {
        block.status = 'FREE'
        block.allocatedTo = null
        reclaimedBlocks++
      }
    })

    
    allocations.value = allocations.value.filter(a => a.refCount > 0)

    isGCRunning.value = false
    memoryLogs.value.unshift(
      ` GC SWEEP COMPLETE: Reclaimed ${reclaimedBlocks * BLOCK_SIZE_KB} KB of leaked memory across ${reclaimedBlocks} heap pages.`
    )
  }, 1000)
}
</script>

<template>
  <div class="studio-container">
    
   
    <header class="hud-header">
      <div>
        <h1 class="brand-title"> DevMem Virtual Allocator & GC Studio</h1>
        <p class="brand-subtitle">
          Vue 3 Composition API interactive memory heap allocator, pointer tracker, and Mark-and-Sweep garbage collector.
        </p>
      </div>

      <div class="header-actions">
        <div class="stat-badge">
          <span>Heap Usage:</span>
          <strong>{{ memoryUsagePercent }}% ({{ totalAllocatedBlocks * BLOCK_SIZE_KB }}/64 KB)</strong>
        </div>
        <button 
          class="btn btn-gc" 
          :disabled="isGCRunning || reclaimableGarbageBytes === 0"
          @click="runGarbageCollector"
        >
          {{ isGCRunning ? ' Sweeping...' : ` Sweep GC (${reclaimableGarbageBytes} KB)` }}
        </button>
      </div>
    </header>

    <!-- MAIN THREE-COLUMN WORKSPACE -->
    <div class="workspace-grid">
      
      <!-- COLUMN 1: STACK POINTER ROOT TOWER -->
      <aside class="panel-card">
        <h3>Stack Memory Root (Pointers)</h3>
        <p class="card-desc">Stack variables holding references to heap allocation IDs:</p>

        <div class="stack-list">
          <div 
            v-for="ptr in stackPointers" 
            :key="ptr.id" 
            class="stack-item"
          >
            <div class="ptr-info">
              <span class="ptr-name">{{ ptr.name }}</span>
              <span class="ptr-arrow">➔ {{ ptr.targetAllocId }}</span>
            </div>
            <button class="btn-sever" @click="nullifyPointer(ptr.id)" title="Nullify Reference">
              Set NULL
            </button>
          </div>

          <div v-if="stackPointers.length === 0" class="empty-state">
            Stack memory clean. No variable references active.
          </div>
        </div>

        <div class="allocation-form">
          <h4>Allocate Memory (malloc)</h4>
          <div class="form-group">
            <input 
              v-model="inputVarName" 
              type="text" 
              placeholder="Variable Name (e.g. userList)" 
              class="input-field" 
            />
            <div class="form-row">
              <select v-model.number="inputSizeBlocks" class="select-field">
                <option :value="1">2 KB (1 Page)</option>
                <option :value="2">4 KB (2 Pages)</option>
                <option :value="4">8 KB (4 Pages)</option>
                <option :value="8">16 KB (8 Pages)</option>
              </select>
              <button class="btn btn-alloc" @click="handleMalloc">malloc()</button>
            </div>
          </div>
        </div>
      </aside>

     
      <main class="panel-card main-heap-card">
        <div class="card-header">
          <h3>64 KB Virtual Heap Memory Pages (32 x 2 KB)</h3>
          <div class="legend">
            <span class="legend-item"><i class="bg-free"></i> Free</span>
            <span class="legend-item"><i class="bg-active"></i> Active</span>
            <span class="legend-item"><i class="bg-garbage"></i> Unreferenced</span>
          </div>
        </div>

        <div class="heap-grid">
          <div 
            v-for="block in heap" 
            :key="block.id" 
            class="heap-block"
            :class="block.status.toLowerCase()"
          >
            <span class="block-addr">{{ block.address }}</span>
            <span class="block-status">{{ block.allocatedTo || 'FREE' }}</span>
          </div>
        </div>
      </main>

    
      <aside class="panel-card">
        <h3>Heap Object Registry</h3>
        <p class="card-desc">Active allocations and reference count counters:</p>

        <div class="registry-list">
          <div 
            v-for="alloc in allocations" 
            :key="alloc.id" 
            class="registry-item"
            :class="{ 'is-unreferenced': alloc.refCount === 0 }"
          >
            <div class="registry-header">
              <strong class="alloc-title">{{ alloc.name }}</strong>
              <span class="alloc-size">{{ alloc.sizeKb }} KB</span>
            </div>
            <div class="registry-meta">
              <span>ID: {{ alloc.id }}</span>
              <span class="ref-badge" :class="alloc.refCount > 0 ? 'ref-ok' : 'ref-dead'">
                Refs: {{ alloc.refCount }}
              </span>
            </div>
          </div>

          <div v-if="allocations.length === 0" class="empty-state">
            Heap registry empty. No allocated objects present.
          </div>
        </div>
      </aside>

    </div>


    <footer class="log-card">
      <h3>Memory Allocator & GC Diagnostic Logs</h3>
      <div class="log-terminal">
        <div 
          v-for="(log, idx) in memoryLogs" 
          :key="idx" 
          class="log-line"
          :class="{ 'is-warn': log.includes('') || log.includes('✂️'), 'is-error': log.includes('❌'), 'is-success': log.includes('🎉') }"
        >
          {{ log }}
        </div>
      </div>
    </footer>

  </div>
</template>

<style>

body {
  margin: 0;
  background-color: #070a13;
  color: #f8fafc;
  font-family: monospace;
}

.studio-container {
  max-width: 1350px;
  margin: 30px auto;
  padding: 0 24px;
}

.hud-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #1e293b;
  padding-bottom: 20px;
  margin-bottom: 25px;
}

.brand-title {
  margin: 0;
  font-size: 24px;
  color: #10b981;
}

.brand-subtitle {
  margin: 4px 0 0 0;
  color: #64748b;
  font-size: 12px;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 15px;
}

.stat-badge {
  background-color: #0f172a;
  border: 1px solid #1e293b;
  padding: 8px 14px;
  border-radius: 8px;
  font-size: 12px;
  color: #10b981;
}

.workspace-grid {
  display: grid;
  grid-template-columns: 320px 1fr 300px;
  gap: 25px;
  margin-bottom: 25px;
}

.panel-card, .log-card {
  background-color: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 14px;
  padding: 20px;
}

.panel-card h3, .log-card h3 {
  margin: 0;
  font-size: 12px;
  color: #64748b;
  text-transform: uppercase;
}

.card-desc {
  font-size: 11px;
  color: #475569;
  margin: 4px 0 15px 0;
}

.stack-list, .registry-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-height: 160px;
}

.stack-item {
  background-color: #070a13;
  border: 1px solid #1e293b;
  border-left: 3px solid #38bdf8;
  padding: 10px;
  border-radius: 6px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.ptr-name {
  font-weight: bold;
  font-size: 12px;
  color: #fff;
  display: block;
}

.ptr-arrow {
  font-size: 10px;
  color: #38bdf8;
}

.btn-sever {
  background-color: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.2);
  color: #ef4444;
  font-size: 10px;
  padding: 4px 8px;
  border-radius: 4px;
  cursor: pointer;
  font-family: monospace;
}

.btn-sever:hover {
  background-color: #ef4444;
  color: #fff;
}

.allocation-form {
  margin-top: 25px;
  border-top: 1px solid #1e293b;
  padding-top: 15px;
}

.allocation-form h4 {
  margin: 0 0 10px 0;
  font-size: 11px;
  color: #cbd5e1;
  text-transform: uppercase;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.form-row {
  display: flex;
  gap: 8px;
}

.input-field, .select-field {
  width: 100%;
  padding: 8px;
  background-color: #070a13;
  border: 1px solid #1e293b;
  border-radius: 6px;
  color: #fff;
  font-size: 12px;
  font-family: monospace;
  box-sizing: border-box;
}

.btn {
  padding: 8px 14px;
  border: none;
  border-radius: 6px;
  font-weight: bold;
  cursor: pointer;
  font-family: monospace;
  font-size: 12px;
}

.btn-alloc { background-color: #10b981; color: #070a13; }
.btn-gc { background-color: #f59e0b; color: #070a13; }
.btn-gc:disabled { opacity: 0.4; cursor: not-allowed; }

.main-heap-card {
  display: flex;
  flex-direction: column;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.legend {
  display: flex;
  gap: 12px;
  font-size: 10px;
  color: #64748b;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 4px;
}

.legend-item i {
  width: 8px;
  height: 8px;
  border-radius: 2px;
  display: inline-block;
}

.bg-free { background-color: #1e293b; }
.bg-active { background-color: #10b981; }
.bg-garbage { background-color: #ef4444; }

.heap-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
  flex-grow: 1;
}

.heap-block {
  background-color: #070a13;
  border: 1px solid #1e293b;
  border-radius: 8px;
  padding: 10px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  min-height: 50px;
  transition: all 0.2s;
}

.heap-block.active {
  border-color: #10b981;
  background-color: rgba(16, 185, 129, 0.08);
}

.heap-block.garbage {
  border-color: #ef4444;
  background-color: rgba(239, 68, 68, 0.08);
}

.block-addr {
  font-size: 9px;
  color: #475569;
}

.block-status {
  font-size: 11px;
  font-weight: bold;
  color: #cbd5e1;
}

.heap-block.active .block-status { color: #10b981; }
.heap-block.garbage .block-status { color: #ef4444; }

.registry-item {
  background-color: #070a13;
  border: 1px solid #1e293b;
  border-radius: 6px;
  padding: 10px;
}

.registry-item.is-unreferenced {
  border-color: #ef4444;
}

.registry-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 4px;
}

.alloc-title {
  font-size: 12px;
  color: #fff;
}

.alloc-size {
  font-size: 10px;
  color: #10b981;
}

.registry-meta {
  display: flex;
  justify-content: space-between;
  font-size: 9px;
  color: #64748b;
}

.ref-badge {
  font-weight: bold;
  padding: 1px 4px;
  border-radius: 3px;
}

.ref-ok { color: #10b981; background-color: rgba(16, 185, 129, 0.1); }
.ref-dead { color: #ef4444; background-color: rgba(239, 68, 68, 0.1); }

.empty-state {
  font-size: 11px;
  color: #475569;
  text-align: center;
  margin-top: 30px;
}

.log-terminal {
  background-color: #070a13;
  border-radius: 8px;
  padding: 12px;
  height: 110px;
  overflow-y: auto;
  margin-top: 10px;
}

.log-line {
  font-size: 11px;
  color: #64748b;
  margin-bottom: 4px;
}

.log-line.is-warn { color: #f59e0b; }
.log-line.is-error { color: #ef4444; }
.log-line.is-success { color: #10b981; font-weight: bold; }
</style>