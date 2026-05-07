<template>
  <main class="page">
    <header class="topbar">
      <div>
        <h1 class="page-title">Inventario</h1>
        <p class="page-sub">Gestao de estoque</p>
      </div>
      <div class="status" :class="{ danger: mensagemErro }">
        <span class="dot"></span>
        {{ mensagemErro || 'Sincronizado' }}
      </div>
    </header>

    <section class="metrics" aria-label="Resumo do inventario">
      <article class="metric">
        <div class="metric-label">Produtos</div>
        <div class="metric-value">{{ totalProdutos }}</div>
      </article>
      <article class="metric">
        <div class="metric-label">Unidades</div>
        <div class="metric-value">{{ totalUnidades }}</div>
      </article>
      <article class="metric">
        <div class="metric-label">Valor total</div>
        <div class="metric-value accent">{{ formatarMoeda(valorTotal) }}</div>
      </article>
    </section>

    <section class="table-card">
      <div class="table-toolbar">
        <div>
          <h2>Produtos em estoque</h2>
          <span>{{ totalProdutos }} {{ totalProdutos === 1 ? 'item registado' : 'itens registados' }}</span>
        </div>
        <button class="btn-add" type="button" @click="abrirModal">
          + Novo produto
        </button>
      </div>

      <div v-if="carregando && !produtos.length" class="empty">
        A carregar produtos...
      </div>
      <div v-else-if="!produtos.length" class="empty">
        Nenhum produto registado. Clique em "Novo produto" para comecar.
      </div>
      <div v-else class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Produto</th>
              <th>Quantidade</th>
              <th>Preco unit.</th>
              <th>Total</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="produto in produtos" :key="produto.id">
              <td><div class="name">{{ produto.nome }}</div></td>
              <td><span class="qty">{{ produto.quantidade }}</span></td>
              <td>{{ formatarMoeda(produto.preco) }}</td>
              <td class="total">{{ formatarMoeda(Number(produto.quantidade) * Number(produto.preco)) }}</td>
              <td>
                <div class="row-actions">
                  <button class="btn-sm" type="button" @click="editar(produto)">Editar</button>
                  <button class="btn-sm del" type="button" @click="remover(produto.id)">Remover</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <div class="overlay" :class="{ open: modalAberto }" @click.self="fecharModal">
      <div class="modal" role="dialog" aria-modal="true" :aria-label="editando ? 'Editar produto' : 'Novo produto'">
        <div class="modal-header">
          <h3>{{ editando ? 'Editar produto' : 'Novo produto' }}</h3>
          <button class="btn-close" type="button" @click="fecharModal" aria-label="Fechar">x</button>
        </div>
        <form @submit.prevent="salvar">
          <div class="modal-body">
            <div class="field">
              <label for="nome">Nome do produto</label>
              <input id="nome" v-model.trim="form.nome" required placeholder="Ex: Teclado USB" />
            </div>
            <div class="row2">
              <div class="field">
                <label for="quantidade">Quantidade</label>
                <input id="quantidade" v-model.number="form.quantidade" required type="number" min="0" placeholder="0" />
              </div>
              <div class="field">
                <label for="preco">Preco (MT)</label>
                <input id="preco" v-model.number="form.preco" required type="number" min="0" step="0.01" placeholder="0.00" />
              </div>
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn-cancel" @click="fecharModal">Cancelar</button>
            <button type="submit" class="btn-save" :disabled="carregando">
              {{ editando ? 'Guardar alteracoes' : 'Guardar produto' }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </main>
</template>

<script setup>
import axios from 'axios'
import { computed, onMounted, ref } from 'vue'

const api = (import.meta.env.VITE_API_URL || 'https://sistema-inventario-backend-hrj1.onrender.com').replace(/\/$/, '')

const produtos = ref([])
const editando = ref(false)
const carregando = ref(false)
const mensagemErro = ref('')
const modalAberto = ref(false)

const form = ref({
  id: null,
  nome: '',
  quantidade: '',
  preco: ''
})

const totalProdutos = computed(() => produtos.value.length)

const totalUnidades = computed(() =>
  produtos.value.reduce((total, p) => total + Number(p.quantidade || 0), 0)
)

const valorTotal = computed(() =>
  produtos.value.reduce(
    (total, p) => total + Number(p.quantidade || 0) * Number(p.preco || 0),
    0
  )
)

const formatarMoeda = (valor) =>
  new Intl.NumberFormat('pt-MZ', {
    style: 'currency',
    currency: 'MZN',
    maximumFractionDigits: 2
  }).format(Number(valor || 0))

const limparFormulario = () => {
  form.value = { id: null, nome: '', quantidade: '', preco: '' }
  editando.value = false
}

const abrirModal = () => {
  limparFormulario()
  modalAberto.value = true
}

const fecharModal = () => {
  modalAberto.value = false
  limparFormulario()
}

const carregar = async () => {
  carregando.value = true
  mensagemErro.value = ''

  try {
    const response = await axios.get(api)
    produtos.value = response.data
  } catch {
    mensagemErro.value = 'Falha ao ligar ao servidor'
  } finally {
    carregando.value = false
  }
}

const salvar = async () => {
  carregando.value = true
  mensagemErro.value = ''

  try {
    if (editando.value) {
      await axios.put(api, form.value)
    } else {
      await axios.post(api, form.value)
    }

    fecharModal()
    await carregar()
  } catch {
    mensagemErro.value = 'Nao foi possivel guardar'
  } finally {
    carregando.value = false
  }
}

const editar = (produto) => {
  form.value = { ...produto }
  editando.value = true
  modalAberto.value = true
}

const remover = async (id) => {
  carregando.value = true
  mensagemErro.value = ''

  try {
    await axios.delete(`${api}?id=${id}`)
    await carregar()
  } catch {
    mensagemErro.value = 'Nao foi possivel eliminar'
  } finally {
    carregando.value = false
  }
}

onMounted(carregar)
</script>

<style>
:root {
  color: #1e1320;
  background: #faf7f9;
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  font-size: 14px;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  margin: 0;
  min-height: 100vh;
  background: #faf7f9;
}

button,
input {
  font: inherit;
}

button {
  border: 0;
  cursor: pointer;
}

.page {
  max-width: 960px;
  margin: 0 auto;
  padding: 28px 24px;
  display: grid;
  gap: 20px;
  position: relative;
  min-height: 100vh;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  flex-wrap: wrap;
}

.page-title {
  font-size: 26px;
  font-weight: 700;
  color: #1e1320;
  letter-spacing: 0;
  line-height: 1.1;
}

.page-sub {
  font-size: 13px;
  color: #9b7a8e;
  margin-top: 3px;
}

.status {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 6px 14px;
  border-radius: 999px;
  border: 1px solid #f0e4eb;
  background: #fff;
  font-size: 12px;
  font-weight: 600;
  color: #9b7a8e;
}

.status.danger {
  color: #993556;
  border-color: #f4c0d1;
  background: #fff5f8;
}

.dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #d4537e;
  box-shadow: 0 0 0 3px rgba(212, 83, 126, 0.15);
}

.danger .dot {
  background: #993556;
  box-shadow: 0 0 0 3px rgba(153, 53, 86, 0.15);
}

.metrics {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.metric {
  background: #fff;
  border: 1px solid #f0e4eb;
  border-radius: 12px;
  padding: 18px 20px;
}

.metric-label {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.6px;
  color: #b89aaa;
  margin-bottom: 8px;
}

.metric-value {
  font-size: 28px;
  font-weight: 700;
  color: #1e1320;
  letter-spacing: 0;
  line-height: 1;
}

.metric-value.accent {
  color: #d4537e;
}

.table-card {
  background: #fff;
  border: 1px solid #f0e4eb;
  border-radius: 12px;
  overflow: hidden;
}

.table-toolbar {
  padding: 16px 20px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  border-bottom: 1px solid #f5eaf1;
  flex-wrap: wrap;
}

.table-toolbar h2 {
  font-size: 15px;
  font-weight: 700;
  color: #1e1320;
  margin-bottom: 2px;
}

.table-toolbar span {
  font-size: 12px;
  color: #b89aaa;
  font-weight: 500;
}

.btn-add {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  height: 36px;
  padding: 0 16px;
  border-radius: 8px;
  background: #d4537e;
  color: #fff;
  font-size: 13px;
  font-weight: 700;
  transition: background 130ms, transform 130ms;
  white-space: nowrap;
}

.btn-add:hover {
  background: #b83d68;
  transform: translateY(-1px);
}

.table-wrap {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  min-width: 520px;
}

thead tr {
  background: #fdf8fb;
}

th {
  padding: 10px 20px;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #b89aaa;
  text-align: left;
}

td {
  padding: 13px 20px;
  font-size: 13px;
  color: #4a3040;
  border-top: 1px solid #faedf3;
  vertical-align: middle;
}

tr:hover td {
  background: #fdf6fa;
}

.name {
  font-weight: 600;
  color: #1e1320;
  font-size: 14px;
}

.qty {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 32px;
  height: 24px;
  padding: 0 8px;
  border-radius: 6px;
  background: #f5eaf3;
  color: #72243e;
  font-size: 12px;
  font-weight: 700;
}

.total {
  font-weight: 700;
  color: #d4537e;
}

.row-actions {
  display: flex;
  gap: 6px;
  justify-content: flex-end;
}

.btn-sm {
  height: 28px;
  padding: 0 10px;
  border: 1px solid #f0e4eb;
  border-radius: 6px;
  background: #fff;
  color: #7a5570;
  font-size: 11px;
  font-weight: 700;
  transition: background 120ms, border-color 120ms, color 120ms;
}

.btn-sm:hover {
  background: #fdf2f7;
}

.btn-sm.del:hover {
  background: #fff0f3;
  color: #993556;
  border-color: #f4c0d1;
}

.empty {
  padding: 48px 20px;
  text-align: center;
  color: #c4a0b5;
  font-size: 13px;
  font-weight: 500;
  line-height: 1.6;
}

.overlay {
  display: none;
  position: fixed;
  inset: 0;
  background: rgba(30, 19, 32, 0.4);
  z-index: 100;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.overlay.open {
  display: flex;
}

.modal {
  background: #fff;
  border-radius: 16px;
  border: 1px solid #f0e4eb;
  width: 100%;
  max-width: 400px;
  overflow: hidden;
  box-shadow: 0 24px 60px rgba(30, 19, 32, 0.18);
}

.modal-header {
  padding: 18px 22px;
  border-bottom: 1px solid #f5eaf1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.modal-header h3 {
  font-size: 16px;
  font-weight: 700;
  color: #1e1320;
}

.btn-close {
  width: 30px;
  height: 30px;
  border: none;
  border-radius: 6px;
  background: #f5eaf3;
  color: #72243e;
  font-size: 20px;
  font-weight: 300;
  line-height: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 120ms;
}

.btn-close:hover {
  background: #f4c0d1;
}

.modal-body {
  padding: 22px;
  display: grid;
  gap: 16px;
}

.field label {
  display: block;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #b89aaa;
  margin-bottom: 6px;
}

.field input {
  width: 100%;
  height: 42px;
  padding: 0 13px;
  border: 1px solid #edd6e5;
  border-radius: 8px;
  background: #fdf8fb;
  color: #1e1320;
  font-size: 14px;
  outline: none;
  transition: border-color 130ms, box-shadow 130ms;
}

.field input:focus {
  border-color: #d4537e;
  background: #fff;
  box-shadow: 0 0 0 3px rgba(212, 83, 126, 0.1);
}

.row2 {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.modal-footer {
  padding: 16px 22px;
  border-top: 1px solid #f5eaf1;
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.btn-cancel {
  height: 38px;
  padding: 0 16px;
  border: 1px solid #edd6e5;
  border-radius: 8px;
  background: #fff;
  color: #7a5570;
  font-size: 13px;
  font-weight: 600;
  transition: background 120ms;
}

.btn-cancel:hover {
  background: #fdf2f7;
}

.btn-save {
  height: 38px;
  padding: 0 20px;
  border: none;
  border-radius: 8px;
  background: #d4537e;
  color: #fff;
  font-size: 13px;
  font-weight: 700;
  transition: background 130ms, transform 130ms;
}

.btn-save:hover {
  background: #b83d68;
  transform: translateY(-1px);
}

.btn-save:disabled {
  opacity: 0.6;
  cursor: wait;
  transform: none;
}

@media (max-width: 720px) {
  .page {
    padding: 20px 16px;
  }

  .metrics {
    grid-template-columns: 1fr;
  }

  .metric-value {
    font-size: 24px;
  }

  .topbar {
    flex-direction: column;
    align-items: flex-start;
  }

  .status {
    width: 100%;
    justify-content: center;
  }
}

@media (max-width: 480px) {
  .row2 {
    grid-template-columns: 1fr;
  }

  .table-toolbar {
    flex-direction: column;
    align-items: flex-start;
  }

  .btn-add {
    width: 100%;
    justify-content: center;
  }
}
</style>
