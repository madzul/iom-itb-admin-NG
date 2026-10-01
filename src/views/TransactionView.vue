<template>
  <div class="min-h-screen">
    <Breadcrumb breadcrumb="transaksi-merchandise" />

    <div
      v-if="isImageModalOpen"
      class="fixed inset-0 z-50 flex items-center justify-center p-6 bg-black/60"
      @click="closeImageModal"
    >
      <img :src="selectedImage" alt="Bukti Bayar" class="max-w-full max-h-full rounded-md shadow-lg" />
    </div>

    <div class="mt-8 space-y-5">
      <section class="relative overflow-hidden rounded-2xl bg-[#003793] p-4 text-white shadow-sm sm:p-6">
        <div class="absolute -right-10 -top-12 h-40 w-40 rounded-full bg-white opacity-10"></div>
        <div class="absolute bottom-0 right-20 h-24 w-24 rounded-full bg-blue-300 opacity-10"></div>
        <div class="relative flex flex-col gap-4 lg:flex-row lg:items-end lg:justify-between">
          <div>
            <h1 class="text-2xl font-bold md:text-4xl">Transaksi Merchandise</h1>
            <p class="mt-2 max-w-2xl text-sm leading-relaxed text-blue-100">
              Kelola pembayaran, bukti transfer, dan status pengiriman pesanan merchandise.
            </p>
          </div>
          <div class="rounded-2xl bg-white/10 px-4 py-3 text-sm text-blue-50">
            <p class="text-sm font-medium text-blue-100">Status pesanan</p>
            <p class="mt-1 font-semibold">Ubah dari dropdown, lalu klik Simpan.</p>
          </div>
        </div>
      </section>

      <section class="grid grid-cols-1 gap-3 md:grid-cols-4">
        <div class="rounded-2xl border-2 border-slate-200 bg-white p-4 shadow-sm">
          <p class="text-sm font-semibold text-slate-500">Total Transaksi</p>
          <p class="mt-2 text-2xl font-bold text-blue-900">{{ pagination?.totalEntries || computedData.length }}</p>
        </div>
        <div class="rounded-2xl border-2 border-slate-200 bg-white p-4 shadow-sm">
          <p class="text-sm font-semibold text-slate-500">Lunas</p>
          <p class="mt-2 text-2xl font-bold text-green-700">{{ paidCount }}</p>
        </div>
        <div class="rounded-2xl border-2 border-slate-200 bg-white p-4 shadow-sm">
          <p class="text-sm font-semibold text-slate-500">Perlu Diproses</p>
          <p class="mt-2 text-2xl font-bold text-amber-700">{{ needProcessCount }}</p>
        </div>
        <div class="rounded-2xl border-2 border-slate-200 bg-white p-4 shadow-sm">
          <p class="text-sm font-semibold text-slate-500">Halaman</p>
          <p class="mt-2 text-2xl font-bold text-blue-900">{{ pagination?.currentPage || 1 }} / {{ pagination?.totalPages || 1 }}</p>
        </div>
      </section>

      <section class="grid gap-3 sm:grid-cols-2 lg:grid-cols-5">
          <div>
            <label class="block mb-1.5 text-sm font-semibold text-slate-900">Per halaman</label>
            <AppSelect
              v-model="limit"
              :options="pageLimitOptions"
              @change="refreshFromFirstPage"
            />
          </div>

          <div>
            <label class="block mb-1.5 text-sm font-semibold text-slate-900">Metode Bayar</label>
            <AppSelect
              v-model="paymentMethod"
              :options="paymentMethodOptions"
              @change="refreshFromFirstPage"
            />
          </div>

          <div>
            <label class="block mb-1.5 text-sm font-semibold text-slate-900">Status Bayar</label>
            <AppSelect
              v-model="paymentStatus"
              :options="paymentStatusOptions"
              @change="refreshFromFirstPage"
            />
          </div>

          <div>
            <label class="block mb-1.5 text-sm font-semibold text-slate-900">Status Pesanan</label>
            <AppSelect
              v-model="orderStatus"
              :options="orderStatusFilterOptions"
              @change="refreshFromFirstPage"
            />
          </div>

          <div>
            <label class="block mb-1.5 text-sm font-semibold text-slate-900">Cari</label>
            <div class="relative">
              <span class="absolute inset-y-0 left-0 flex items-center pl-2.5">
                <svg viewBox="0 0 24 24" class="w-4 h-4 text-slate-400 fill-current">
                  <path d="M10 4a6 6 0 100 12 6 6 0 000-12zm-8 6a8 8 0 1114.32 4.906l5.387 5.387a1 1 0 01-1.414 1.414l-5.387-5.387A8 8 0 012 10z" />
                </svg>
              </span>
              <input
                v-model="search"
                @input="onSearchInput"
                placeholder="Kode, nama, email..."
                class="block w-full rounded-lg border border-slate-200 bg-white py-2 pl-8 pr-3 text-sm text-slate-700 placeholder-slate-400 transition-all focus:border-blue-400 focus:outline-none focus:ring-2 focus:ring-blue-500/30"
              />
            </div>
          </div>
      </section>

      <section class="flex flex-col gap-3 rounded-2xl border-2 border-slate-200 bg-white p-4 shadow-sm sm:flex-row sm:items-center sm:justify-between">
        <div>
          <p class="text-sm font-semibold text-slate-900">Export Data Transaksi</p>
          <p class="mt-0.5 text-xs text-slate-500">
            Mengunduh seluruh transaksi sesuai filter &amp; pencarian aktif (semua halaman, bukan hanya halaman ini).
            <span v-if="exporting" class="font-semibold text-blue-700">{{ exportProgressText }}</span>
          </p>
        </div>
        <div class="flex gap-2">
          <button
            type="button"
            :disabled="Boolean(exporting)"
            class="inline-flex items-center gap-1.5 rounded-full border border-green-600 bg-white px-4 py-2 text-sm font-semibold text-green-700 hover:bg-green-50 disabled:cursor-not-allowed disabled:opacity-50"
            @click="exportTransactions('csv')"
          >
            <svg viewBox="0 0 24 24" class="h-4 w-4 fill-current" aria-hidden="true">
              <path d="M12 3a1 1 0 011 1v9.586l3.293-3.293a1 1 0 111.414 1.414l-5 5a1 1 0 01-1.414 0l-5-5a1 1 0 111.414-1.414L11 13.586V4a1 1 0 011-1zM4 19a1 1 0 011-1h14a1 1 0 110 2H5a1 1 0 01-1-1z" />
            </svg>
            {{ exporting === 'csv' ? 'Menyiapkan...' : 'Export CSV' }}
          </button>
          <button
            type="button"
            :disabled="Boolean(exporting)"
            class="inline-flex items-center gap-1.5 rounded-full bg-green-600 px-4 py-2 text-sm font-semibold text-white hover:bg-green-500 disabled:cursor-not-allowed disabled:opacity-50"
            @click="exportTransactions('xlsx')"
          >
            <svg viewBox="0 0 24 24" class="h-4 w-4 fill-current" aria-hidden="true">
              <path d="M12 3a1 1 0 011 1v9.586l3.293-3.293a1 1 0 111.414 1.414l-5 5a1 1 0 01-1.414 0l-5-5a1 1 0 111.414-1.414L11 13.586V4a1 1 0 011-1zM4 19a1 1 0 011-1h14a1 1 0 110 2H5a1 1 0 01-1-1z" />
            </svg>
            {{ exporting === 'xlsx' ? 'Menyiapkan...' : 'Export Excel' }}
          </button>
        </div>
      </section>

      <div class="overflow-hidden rounded-2xl border-2 border-slate-200 bg-white shadow-sm">
        <div class="overflow-x-auto">
          <table class="min-w-full text-sm">
            <thead>
              <tr class="bg-blue-900">
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">No</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Pesanan</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Pembeli</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Alamat</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Catatan</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-right text-blue-100">Total</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Pembayaran</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Status Pesanan</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Tanggal</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-left text-blue-100">Bukti</th>
                <th class="px-4 py-3.5 text-sm font-semibold text-right text-blue-100">Aksi</th>
              </tr>
            </thead>
            <tbody class="bg-white divide-y divide-slate-100">
              <tr v-if="isLoading">
                <td v-for="c in 11" :key="c" class="px-4 py-4">
                  <div class="h-4 w-full max-w-[120px] animate-pulse rounded bg-slate-100"></div>
                </td>
              </tr>
              <tr v-else-if="computedData.length === 0">
                <td colspan="11" class="px-4 py-12 text-sm text-center text-slate-400 italic">Belum ada transaksi merchandise.</td>
              </tr>
              <tr
                v-else
                v-for="(u, index) in computedData"
                :key="u.id"
                class="transition-colors hover:bg-blue-50/40"
              >
                <td class="px-4 py-4 text-slate-500 align-middle">{{ startNumber + index }}</td>
                <td class="px-4 py-4 align-middle">
                  <p class="font-semibold text-slate-900 whitespace-nowrap">{{ u.code || '-' }}</p>
                  <p class="mt-0.5 text-xs text-slate-500 whitespace-nowrap">
                    {{ merchandiseName(u) }} x {{ u.qty || 0 }}
                  </p>
                </td>
                <td class="px-4 py-4 align-middle">
                  <p class="font-medium text-slate-900 whitespace-nowrap">{{ u.username || '-' }}</p>
                  <p class="text-xs text-slate-500 whitespace-nowrap">{{ u.email || '-' }}</p>
                  <p class="text-xs text-slate-500 whitespace-nowrap">{{ u.noTelp || '-' }}</p>
                </td>
                <td class="px-4 py-4 align-middle">
                  <p class="max-w-[240px] text-slate-600 line-clamp-2" :title="u.address || '-'">
                    {{ u.address || '-' }}
                  </p>
                </td>
                <td class="px-4 py-4 align-middle">
                  <p class="max-w-[200px] text-slate-600 line-clamp-2" :title="u.notes || '-'">
                    {{ u.notes || '-' }}
                  </p>
                </td>
                <td class="px-4 py-4 text-right align-middle whitespace-nowrap">
                  <p class="font-semibold text-slate-900">{{ formatNominal(u.grossAmount) }}</p>
                  <p class="text-xs text-slate-500">{{ u.qty || 0 }} item</p>
                </td>
                <td class="px-4 py-4 align-middle">
                  <div class="space-y-1">
                    <span :class="methodBadgeClass(u.paymentMethod)" class="inline-block px-2 py-0.5 text-xs font-medium rounded-full">
                      {{ paymentMethodLabel(u.paymentMethod) }}
                    </span>
                    <br />
                    <span :class="paymentStatusBadgeClass(u.paymentStatus)" class="inline-block px-2 py-0.5 text-xs font-medium rounded-full capitalize">
                      {{ paymentStatusLabel(u.paymentStatus) }}
                    </span>
                    <br v-if="canConfirmPayment(u)" />
                    <button
                      v-if="canConfirmPayment(u)"
                      type="button"
                      :disabled="confirmingPayment[u.id]"
                      class="rounded-full bg-green-50 px-2 py-0.5 text-xs font-semibold text-green-700 hover:bg-green-100 disabled:cursor-not-allowed disabled:opacity-50"
                      @click.prevent="confirmPayment(u)"
                    >
                      {{ confirmingPayment[u.id] ? 'Memproses...' : 'Konfirmasi Pembayaran' }}
                    </button>
                  </div>
                </td>
                <td class="px-4 py-4 align-middle">
                  <div class="min-w-[170px] space-y-2">
                    <AppSelect
                      v-model="statusDraft[u.id]"
                      :disabled="savingStatus[u.id]"
                      :options="orderStatusSelectOptions"
                    />
                    <span
                      class="inline-block px-2 py-0.5 text-xs font-medium rounded-full"
                      :class="orderStatusBadgeClass(statusDraft[u.id] || u.status)"
                    >
                      {{ formatStatus(statusDraft[u.id] || u.status) }}
                    </span>
                  </div>
                </td>
                <td class="px-4 py-4 text-slate-600 align-middle whitespace-nowrap">
                  <p>{{ formatDate(u.createdAt) }}</p>
                  <p class="text-xs text-slate-400">Update: {{ formatDate(u.updatedAt) }}</p>
                </td>
                <td class="px-4 py-4 align-middle">
                  <button
                    v-if="isManualProof(u.payment)"
                    class="rounded-full bg-blue-50 px-3 py-1.5 text-sm font-semibold text-blue-700 hover:bg-blue-100"
                    @click="openImageModal(u.payment || '')"
                  >
                    Lihat
                  </button>
                  <span
                    v-else
                    class="inline-flex h-8 min-w-[54px] items-center justify-center rounded-full bg-slate-100 px-3 text-xs font-semibold text-slate-500"
                  >
                    {{ u.paymentMethod === 'midtrans' ? 'MID' : '-' }}
                  </span>
                </td>
                <td class="px-4 py-4 text-right align-middle whitespace-nowrap">
                  <button
                    type="button"
                    :disabled="savingStatus[u.id] || !hasStatusChanged(u)"
                    class="mr-2 rounded-full bg-indigo-600 px-3 py-1.5 text-xs font-semibold text-white hover:bg-indigo-500 disabled:cursor-not-allowed disabled:bg-slate-300"
                    @click.prevent="saveStatus(u)"
                  >
                    {{ savingStatus[u.id] ? 'Saving...' : 'Simpan' }}
                  </button>
                  <button
                    v-if="u.publicToken"
                    type="button"
                    class="mr-2 rounded-full bg-blue-50 px-3 py-1.5 text-sm font-semibold text-blue-700 hover:bg-blue-100"
                    @click.prevent="copyTrackingLink(u)"
                  >
                    Salin Link
                  </button>
                  <button
                    type="button"
                    class="rounded-full bg-red-50 px-3 py-1.5 text-xs font-semibold text-red-600 hover:bg-red-100"
                    @click.prevent="deleteItem(u.id)"
                  >
                    Hapus
                  </button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="flex flex-col items-center justify-between gap-2 border-t border-slate-100 px-6 py-4 sm:flex-row">
          <span class="text-xs text-slate-500">
            Menampilkan {{ pagination?.start || 0 }}-{{ pagination?.end || 0 }} dari {{ pagination?.totalEntries || 0 }} entri
          </span>
          <div class="inline-flex">
            <button
              class="rounded-l-lg border border-slate-200 bg-white px-3.5 py-1.5 text-sm font-medium text-slate-600 transition-all hover:bg-slate-50 disabled:cursor-not-allowed disabled:opacity-40"
              :disabled="page <= 1"
              @click="() => { page = (pagination?.currentPage || 1) - 1; getData(); }"
            >
              Sebelumnya
            </button>
            <button
              class="rounded-r-lg border-y border-r border-slate-200 bg-white px-3.5 py-1.5 text-sm font-medium text-slate-600 transition-all hover:bg-slate-50 disabled:cursor-not-allowed disabled:opacity-40"
              :disabled="page >= (pagination?.totalPages || 1)"
              @click="() => { page = (pagination?.currentPage || 1) + 1; getData(); }"
            >
              Berikutnya
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { GET_TRANSACTIONS, DELETE_TRANSACTION, PUT_TRANSACTION, CONFIRM_TRANSACTION_PAYMENT, FETCH_ALL_TRANSACTIONS } from '@/store/transaction.module';
import { exportRows } from '@/utils/exportData';
import type { ExportColumn } from '@/utils/exportData';
import Breadcrumb from '@/components/AppBreadcrumb.vue';
import AppSelect from '@/components/input/AppSelect.vue';
import { useStore } from 'vuex';
import Swal from 'sweetalert2';

type Merchandise = {
  id?: number;
  name?: string;
  price?: number | string | null;
};

type Transaction = {
  id: number;
  code?: string;
  publicToken?: string;
  username?: string;
  email?: string;
  noTelp?: string;
  address?: string;
  notes?: string;
  merchandiseId?: number;
  merchandises?: Merchandise;
  qty?: number;
  payment?: string | null;
  paymentMethod?: string;
  paymentStatus?: string;
  paymentType?: string | null;
  midtransOrderId?: string | null;
  midtransTransactionId?: string | null;
  grossAmount?: number | string | null;
  status: string;
  createdAt?: string;
  updatedAt?: string;
};

const store = useStore();

const isLoading = ref(true);
const page = ref(1);
const limit = ref(10);
const search = ref('');
const paymentMethod = ref('');
const paymentStatus = ref('');
const orderStatus = ref('');
const isImageModalOpen = ref(false);
const selectedImage = ref('');
const statusDraft = ref<Record<number, string>>({});
const savingStatus = ref<Record<number, boolean>>({});
const confirmingPayment = ref<Record<number, boolean>>({});
const orderStatusOptions = ['waiting', 'on process', 'on delivery', 'arrived', 'done', 'canceled', 'denied'];
const notifyStatuses = new Set(['on process', 'on delivery', 'arrived', 'done', 'canceled', 'denied']);
const orderStatusLabels: Record<string, string> = {
  waiting: 'Menunggu',
  'on process': 'Diproses',
  'on delivery': 'Dikirim',
  arrived: 'Tiba',
  done: 'Selesai',
  canceled: 'Dibatalkan',
  denied: 'Ditolak',
};
const paymentStatusLabels: Record<string, string> = {
  pending: 'Menunggu',
  settlement: 'Lunas',
  expired: 'Kedaluwarsa',
  failed: 'Gagal',
  refunded: 'Dikembalikan',
};
const pageLimitOptions = [
  { value: 5, label: '5' },
  { value: 10, label: '10' },
  { value: 20, label: '20' },
  { value: 50, label: '50' },
];
const paymentMethodOptions = [
  { value: '', label: 'Semua' },
  { value: 'manual', label: 'Manual' },
  { value: 'midtrans', label: 'Midtrans' },
];
const paymentStatusOptions = [
  { value: '', label: 'Semua' },
  { value: 'pending', label: 'Menunggu' },
  { value: 'settlement', label: 'Lunas' },
  { value: 'expired', label: 'Kedaluwarsa' },
  { value: 'failed', label: 'Gagal' },
  { value: 'refunded', label: 'Dikembalikan' },
];
const orderStatusSelectOptions = orderStatusOptions.map((status) => ({
  value: status,
  label: orderStatusLabels[status],
}));
const orderStatusFilterOptions = [
  { value: '', label: 'Semua' },
  ...orderStatusSelectOptions,
];
const publicAppBaseUrl = (process.env.VUE_APP_PUBLIC_APP_URL || 'https://iom-app.kirisame.jp.net').replace(/\/+$/, '');
let searchTimer: ReturnType<typeof setTimeout> | null = null;

const computedData = computed<Transaction[]>(() => {
  const transactions = store.getters.transactions;
  return Array.isArray(transactions) ? transactions : [];
});

const pagination = computed(() => store.getters.transactionPagination || {});
const startNumber = computed(() => pagination.value?.start || 1);
const paidCount = computed(() => computedData.value.filter((item) => item.paymentStatus === 'settlement').length);
const needProcessCount = computed(() => computedData.value.filter((item) => ['waiting', 'on process'].includes(item.status)).length);

const getData = async () => {
  isLoading.value = true;

  try {
    const data = await store.dispatch(GET_TRANSACTIONS, {
      data: {
        page: page.value,
        limit: limit.value,
        search: search.value || undefined,
        paymentMethod: paymentMethod.value || undefined,
        paymentStatus: paymentStatus.value || undefined,
        status: orderStatus.value || undefined,
      },
    });

    (data || []).forEach((transaction: Transaction) => {
      statusDraft.value[transaction.id] = transaction.status;
    });

    return data;
  } finally {
    isLoading.value = false;
  }
};

const refreshFromFirstPage = () => {
  page.value = 1;
  getData();
};

const onSearchInput = () => {
  if (searchTimer) clearTimeout(searchTimer);
  searchTimer = setTimeout(refreshFromFirstPage, 350);
};

onMounted(getData);

const openImageModal = (imageUrl: string) => {
  selectedImage.value = imageUrl;
  isImageModalOpen.value = true;
};

const closeImageModal = () => {
  isImageModalOpen.value = false;
};

const isManualProof = (payment?: string | null) => Boolean(payment) && !String(payment).startsWith('midtrans:');

const canConfirmPayment = (item: Transaction) => item.paymentMethod === 'manual' && item.paymentStatus === 'pending';

const confirmPayment = async (item: Transaction) => {
  const confirmation = await Swal.fire({
    title: 'Konfirmasi pembayaran?',
    text: 'Pastikan Anda sudah memeriksa bukti transfer sebelum konfirmasi. Pembeli akan menerima notifikasi Email dan WhatsApp.',
    icon: 'question',
    showCancelButton: true,
    confirmButtonColor: '#16a34a',
    cancelButtonColor: '#6b7280',
    confirmButtonText: 'Ya, sudah saya periksa',
    cancelButtonText: 'Batal',
  });

  if (!confirmation.isConfirmed) return;

  confirmingPayment.value[item.id] = true;

  try {
    await store.dispatch(CONFIRM_TRANSACTION_PAYMENT, { id: item.id });
    await getData();
    await Swal.fire({
      title: 'Pembayaran dikonfirmasi',
      text: 'Status pembayaran diperbarui menjadi Lunas.',
      icon: 'success',
      confirmButtonColor: '#4f46e5',
    });
  } catch (error) {
    await Swal.fire({
      title: 'Gagal konfirmasi pembayaran',
      text: getErrorMessage(error),
      icon: 'error',
      confirmButtonColor: '#4f46e5',
    });
  } finally {
    confirmingPayment.value[item.id] = false;
  }
};

const merchandiseName = (transaction: Transaction) => {
  return transaction.merchandises?.name || `Merchandise #${transaction.merchandiseId || '-'}`;
};

const formatStatus = (status?: string) => {
  if (!status) return '-';
  return orderStatusLabels[status] || status.replace(/\b\w/g, (char) => char.toUpperCase());
};

const formatNominal = (amount?: number | string | null) => {
  if (amount == null || amount === '') return '-';
  const value = Number(amount);
  if (Number.isNaN(value)) return String(amount);
  return `Rp ${value.toLocaleString('id-ID')}`;
};

const paymentMethodLabel = (method?: string) => {
  if (method === 'midtrans') return 'Midtrans';
  if (method === 'manual') return 'Manual';
  return '-';
};

const paymentStatusLabel = (status?: string) => paymentStatusLabels[status || 'pending'] || 'Menunggu';

const methodBadgeClass = (method?: string) =>
  method === 'midtrans' ? 'bg-blue-100 text-blue-700' : 'bg-slate-200 text-slate-700';

const paymentStatusBadgeClass = (status?: string) => {
  switch (status) {
    case 'settlement':
      return 'bg-green-100 text-green-700';
    case 'expired':
      return 'bg-gray-200 text-gray-700';
    case 'failed':
      return 'bg-red-100 text-red-700';
    case 'refunded':
      return 'bg-purple-100 text-purple-700';
    case 'pending':
    default:
      return 'bg-yellow-100 text-yellow-700';
  }
};

const orderStatusBadgeClass = (status?: string) => {
  switch (status) {
    case 'done':
    case 'arrived':
      return 'bg-green-100 text-green-700';
    case 'on process':
    case 'on delivery':
      return 'bg-blue-100 text-blue-700';
    case 'canceled':
    case 'denied':
      return 'bg-red-100 text-red-700';
    case 'waiting':
    default:
      return 'bg-slate-100 text-slate-700';
  }
};

const formatDate = (dateString?: string) => {
  if (!dateString) return '-';

  const date = new Date(dateString);
  if (Number.isNaN(date.getTime())) return '-';

  const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'Mei', 'Jun', 'Jul', 'Agu', 'Sep', 'Okt', 'Nov', 'Des'];
  const day = String(date.getDate()).padStart(2, '0');
  const month = monthNames[date.getMonth()];
  const year = date.getFullYear();
  const hour = String(date.getHours()).padStart(2, '0');
  const minute = String(date.getMinutes()).padStart(2, '0');

  return `${day} ${month} ${year}, ${hour}:${minute}`;
};

const hasStatusChanged = (item: Transaction) => {
  return Boolean(item?.id) && Boolean(statusDraft.value[item.id]) && statusDraft.value[item.id] !== item.status;
};

const trackingUrl = (item: Transaction) => {
  if (!item.publicToken) return '';
  return `${publicAppBaseUrl}/order-status?token=${encodeURIComponent(item.publicToken)}`;
};

const getErrorMessage = (error: unknown) => {
  const apiError = error as { response?: { data?: { message?: string } }; message?: string };
  return apiError?.response?.data?.message || apiError?.message || 'Terjadi kesalahan.';
};

const writeClipboard = async (text: string) => {
  if (navigator.clipboard?.writeText) {
    await navigator.clipboard.writeText(text);
    return;
  }

  const textArea = document.createElement('textarea');
  textArea.value = text;
  textArea.style.position = 'fixed';
  textArea.style.opacity = '0';
  document.body.appendChild(textArea);
  textArea.focus();
  textArea.select();
  document.execCommand('copy');
  document.body.removeChild(textArea);
};

const copyTrackingLink = async (item: Transaction) => {
  const url = trackingUrl(item);
  if (!url) {
    await Swal.fire({
      title: 'Link belum tersedia',
      text: 'Transaksi lama perlu token publik sebelum link status bisa disalin.',
      icon: 'warning',
      confirmButtonColor: '#4f46e5',
    });
    return;
  }

  try {
    await writeClipboard(url);
    await Swal.fire({
      title: 'Link tersalin',
      text: 'Link status pesanan siap dikirim ke pembeli.',
      icon: 'success',
      timer: 1600,
      showConfirmButton: false,
    });
  } catch (error) {
    await Swal.fire({
      title: 'Gagal menyalin link',
      text: url,
      icon: 'info',
      confirmButtonColor: '#4f46e5',
    });
  }
};

const saveStatus = async (item: Transaction) => {
  const nextStatus = statusDraft.value[item.id];

  if (!nextStatus || nextStatus === item.status) return;

  const confirmation = await Swal.fire({
    title: 'Update status pesanan?',
    text: notifyStatuses.has(nextStatus)
      ? 'Pembeli akan menerima notifikasi Email dan WhatsApp jika kontak tersedia.'
      : 'Status pesanan akan diperbarui.',
    icon: 'question',
    showCancelButton: true,
    confirmButtonColor: '#4f46e5',
    cancelButtonColor: '#6b7280',
    confirmButtonText: 'Ya, update',
    cancelButtonText: 'Batal',
  });

  if (!confirmation.isConfirmed) {
    statusDraft.value[item.id] = item.status;
    return;
  }

  savingStatus.value[item.id] = true;

  try {
    await store.dispatch(PUT_TRANSACTION, {
      id: item.id,
      data: {
        status: nextStatus,
      },
    });
    await getData();
    await Swal.fire({
      title: 'Berhasil',
      text: 'Status pesanan berhasil diperbarui.',
      icon: 'success',
      confirmButtonColor: '#4f46e5',
    });
  } catch (error) {
    statusDraft.value[item.id] = item.status;
    await Swal.fire({
      title: 'Gagal update status',
      text: getErrorMessage(error),
      icon: 'error',
      confirmButtonColor: '#4f46e5',
    });
  } finally {
    savingStatus.value[item.id] = false;
  }
};

// ─── Export CSV / Excel ───────────────────────────────────────────────────
const exporting = ref<'' | 'csv' | 'xlsx'>('');
const exportLoaded = ref(0);
const exportTotal = ref(0);
const exportProgressText = computed(() =>
  exportTotal.value ? `Mengambil data ${exportLoaded.value}/${exportTotal.value}...` : 'Mengambil data...',
);

const toNumber = (value?: number | string | null) => {
  if (value == null || value === '') return null;
  const num = Number(value);
  return Number.isNaN(num) ? null : num;
};

const pad2 = (n: number) => String(n).padStart(2, '0');

const formatDateTimeExport = (dateString?: string) => {
  if (!dateString) return '';
  const date = new Date(dateString);
  if (Number.isNaN(date.getTime())) return '';
  return `${date.getFullYear()}-${pad2(date.getMonth() + 1)}-${pad2(date.getDate())} ${pad2(date.getHours())}:${pad2(date.getMinutes())}`;
};

const exportColumns: ExportColumn<Transaction>[] = [
  { header: 'No', value: (_row, i) => i + 1, width: 6, format: 'integer' },
  { header: 'Kode Pesanan', value: (row) => row.code || '', width: 22 },
  { header: 'Tanggal Pesan', value: (row) => formatDateTimeExport(row.createdAt), width: 18 },
  { header: 'Nama Pembeli', value: (row) => row.username || '', width: 24 },
  { header: 'Email', value: (row) => row.email || '', width: 28 },
  { header: 'No. Telp', value: (row) => (row.noTelp ? String(row.noTelp) : ''), width: 16 },
  { header: 'Alamat', value: (row) => row.address || '', width: 40 },
  { header: 'Catatan', value: (row) => row.notes || '', width: 30 },
  { header: 'Merchandise', value: (row) => merchandiseName(row), width: 28 },
  { header: 'Harga Satuan (Rp)', value: (row) => toNumber(row.merchandises?.price), width: 16, format: 'currency' },
  { header: 'Qty', value: (row) => toNumber(row.qty), width: 8, format: 'integer' },
  { header: 'Total (Rp)', value: (row) => toNumber(row.grossAmount), width: 16, format: 'currency' },
  { header: 'Metode Bayar', value: (row) => paymentMethodLabel(row.paymentMethod), width: 14 },
  { header: 'Tipe Pembayaran', value: (row) => row.paymentType || '', width: 16 },
  { header: 'Status Bayar', value: (row) => paymentStatusLabel(row.paymentStatus), width: 14 },
  { header: 'Status Pesanan', value: (row) => formatStatus(row.status), width: 14 },
  { header: 'Midtrans Order ID', value: (row) => row.midtransOrderId || '', width: 24 },
  { header: 'Bukti Transfer', value: (row) => (isManualProof(row.payment) ? String(row.payment) : ''), width: 40 },
  { header: 'Link Status Pesanan', value: (row) => trackingUrl(row), width: 40 },
  { header: 'Terakhir Diperbarui', value: (row) => formatDateTimeExport(row.updatedAt), width: 18 },
];

const buildExportFilename = () => {
  const now = new Date();
  const stamp = `${now.getFullYear()}${pad2(now.getMonth() + 1)}${pad2(now.getDate())}-${pad2(now.getHours())}${pad2(now.getMinutes())}`;
  const filters = [paymentMethod.value, paymentStatus.value, orderStatus.value]
    .filter(Boolean)
    .map((part) => part.replace(/\s+/g, '-'));
  return ['transaksi-merchandise', ...filters, stamp].join('_');
};

const exportTransactions = async (format: 'csv' | 'xlsx') => {
  if (exporting.value) return;

  exporting.value = format;
  exportLoaded.value = 0;
  exportTotal.value = 0;

  try {
    const rows: Transaction[] = await store.dispatch(FETCH_ALL_TRANSACTIONS, {
      data: {
        search: search.value || undefined,
        paymentMethod: paymentMethod.value || undefined,
        paymentStatus: paymentStatus.value || undefined,
        status: orderStatus.value || undefined,
      },
      onProgress: (loaded: number, total: number) => {
        exportLoaded.value = loaded;
        exportTotal.value = total;
      },
    });

    if (!rows.length) {
      await Swal.fire({
        title: 'Tidak ada data',
        text: 'Tidak ada transaksi yang cocok dengan filter saat ini.',
        icon: 'info',
        confirmButtonColor: '#4f46e5',
      });
      return;
    }

    exportRows(format, rows, exportColumns, buildExportFilename(), 'Transaksi Merchandise');
  } catch (error) {
    await Swal.fire({
      title: 'Gagal export data',
      text: getErrorMessage(error),
      icon: 'error',
      confirmButtonColor: '#4f46e5',
    });
  } finally {
    exporting.value = '';
  }
};

const deleteItem = async (id: number) => {
  const confirmation = await Swal.fire({
    title: 'Hapus transaksi?',
    text: 'Data transaksi yang dihapus tidak bisa dikembalikan.',
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#3085d6',
    cancelButtonColor: '#d33',
    confirmButtonText: 'Ya, hapus',
    cancelButtonText: 'Batal',
  });

  if (!confirmation.isConfirmed) return;

  try {
    await store.dispatch(DELETE_TRANSACTION, { id });
    await getData();
    await Swal.fire({
      title: 'Terhapus',
      text: 'Transaksi berhasil dihapus.',
      icon: 'success',
      confirmButtonColor: '#4CAF50',
      confirmButtonText: 'OK',
    });
  } catch (error) {
    await Swal.fire({
      title: 'Gagal hapus transaksi',
      text: getErrorMessage(error),
      icon: 'error',
      confirmButtonColor: '#4f46e5',
    });
  }
};
</script>
