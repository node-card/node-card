<template>
    <div class="card-redeem-page">
        <el-main class="redeem-main">
            <el-row justify="center">
                <el-col :xs="24" :sm="23" :md="22" :lg="20">
                    <el-card class="hero-card" shadow="never">
                        <div class="hero-title">重要公告</div>
                        <div class="hero-desc">{{ state.noticeText }}</div>
                    </el-card>

                    <el-card class="mt16 redeem-card" shadow="never">
                        <div class="mode-header">
                            <div class="section-head mode-head">兑换模式</div>
                            <div class="browser-tabs" role="tablist" aria-label="兑换模式">
                                <button
                                    type="button"
                                    class="browser-tab"
                                    :class="{ 'is-active': state.mode === 'single' }"
                                    @click="state.mode = 'single'"
                                >
                                    单个兑换
                                </button>
                                <button
                                    type="button"
                                    class="browser-tab"
                                    :class="{ 'is-active': state.mode === 'batch' }"
                                    @click="state.mode = 'batch'"
                                >
                                    批量兑换
                                </button>
                            </div>
                        </div>

                        <div class="card-head-block" :class="{ 'is-disabled': state.cardHeadSelectionLocked }">
                            <div class="card-head-header">
                                <div>
                                    <div class="card-head-title">选择卡头</div>
                                    <div class="card-head-subtitle">如果不选择，默认按下单时选择的卡头进行兑换</div>
                                </div>
                            </div>

                            <div v-if="state.cardHeadOptions.length > 0" class="card-head-tabs">
                                <button
                                    v-for="item in state.cardHeadOptions"
                                    :key="item.platformId"
                                    type="button"
                                    class="card-head-tab"
                                    :class="{
                                        'is-active': state.selectedPlatformId === item.platformId,
                                        'is-disabled': item.disabled,
                                    }"
                                    @click="onSelectCardHead(item)"
                                >
                                    <div class="card-head-tab-top">
                                        <span>卡头 {{ item.cardHeadNum }} · {{ item.region }}</span>
                                        <span class="card-head-badge" :class="item.disabled ? 'is-offline' : 'is-ok'">
                                            {{ item.disabled ? '维护中' : '可用' }}
                                        </span>
                                    </div>
                                    <div class="card-head-track">
                                        <div
                                            class="card-head-fill"
                                            :class="{
                                                'is-zero': !item.disabled && item.capacityRate <= 0,
                                                'is-offline': item.disabled,
                                            }"
                                            :style="{ width: getCardHeadRateWidth(item) }"
                                        ></div>
                                    </div>
                                    <div class="card-head-rate">
                                        {{ item.disabled ? '维护中' : formatCardHeadRate(item.capacityRate) }}
                                    </div>
                                </button>
                            </div>
                            <div v-else class="card-head-empty">暂无卡头状态数据</div>

                            <div class="card-head-footer">
                                <el-button
                                    class="card-head-refresh-btn"
                                    :loading="state.cardHeadLoading"
                                    :disabled="state.cardHeadSelectionLocked"
                                    @click="loadCardHeadOptions"
                                >
                                    刷新卡头状态
                                </el-button>
                                <span v-if="state.cardHeadSelectionLocked" class="card-head-lock-text">批量兑换进行中，卡头已锁定</span>
                            </div>
                        </div>

                        <div v-show="state.mode === 'single'" class="mt16">
                            <div class="section-head panel-head">单个兑换</div>

                            <el-form class="single-issue-form" label-width="110px">
                                <el-form-item label="输入卡密">
                                    <el-input
                                        v-model="state.singleForm.secret"
                                        clearable
                                        placeholder="请输入卡密"
                                        @keyup.enter="submitSingleIssue"
                                    />
                                </el-form-item>

                                <el-form-item label="选择商户">
                                    <el-select
                                        v-model="state.singleForm.merchantId"
                                        class="w100"
                                        clearable
                                        :loading="state.merchantOptionsLoading"
                                        placeholder="请选择商户，0刀卡无需填"
                                        @change="onSingleMerchantChange"
                                    >
                                        <el-option
                                            v-for="item in state.merchantOptions"
                                            :key="item.id"
                                            :label="item.name"
                                            :value="item.id"
                                        />
                                    </el-select>
                                </el-form-item>

                                <div class="merchant-link-row">
                                    <a href="https://zcnzlu6e9jne.feishu.cn/docx/DbOedWSwXoUIIXxlL7XcH8wXnRe" target="_blank" rel="noopener noreferrer">
                                        0刀卡无需填，选择后，该卡只能用于该商户，请慎重！不懂的点这里查看教程
                                    </a>
                                </div>

                                <el-form-item>
                                    <el-button type="primary" :loading="state.singleIssuing" @click="submitSingleIssue">立即兑换</el-button>
                                    <el-button class="single-check-btn" type="success" :loading="state.singleStatusChecking" @click="checkSingleCardStatus">
                                        只查询不兑换
                                    </el-button>
                                    <el-button
                                        class="single-record-btn"
                                        type="warning"
                                        :disabled="state.singleEventsLoading || state.singleEventsCooldown > 0"
                                        @click="fetchAndScrollSingleEvents"
                                    >
                                        获取消费记录{{ state.singleEventsCooldown > 0 ? ' ' + state.singleEventsCooldown : '' }}
                                    </el-button>
                                    <el-button @click="resetSingleForm">重置</el-button>
                                </el-form-item>
                            </el-form>

                            <div ref="singleEventAnchorRef"></div>

                            <el-card v-if="state.singleResult" class="mt16" shadow="never">
                                <div class="info-grid">
                                    <el-card class="info-panel" shadow="never">
                                        <template #header>卡片信息（点击可自动复制）</template>
                                        <div v-for="item in singleCardRows" :key="item.label" class="copy-row" @click="copyInfoValue(item.value)">
                                            <span class="copy-label">{{ item.label }}</span>
                                            <span class="copy-value">{{ item.value }}</span>
                                        </div>
                                    </el-card>

                                    <el-card class="info-panel" shadow="never">
                                        <template #header>地址信息（点击可自动复制）</template>
                                        <div v-for="item in singleAddressRows" :key="item.label" class="copy-row" @click="copyInfoValue(item.value)">
                                            <span class="copy-label">{{ item.label }}</span>
                                            <span class="copy-value">{{ item.value }}</span>
                                        </div>
                                    </el-card>
                                </div>

                                <div class="event-box mt16">
                                    <div class="event-toolbar">
                                        <el-button
                                            class="event-load-btn"
                                            type="primary"
                                            :loading="state.singleEventsLoading"
                                            :disabled="state.singleEventsLoading || state.singleEventsCooldown > 0"
                                            @click="loadSingleEvents"
                                        >
                                            获取消费记录{{ state.singleEventsCooldown > 0 ? ' ' + state.singleEventsCooldown : '' }}
                                        </el-button>
                                    </div>
                                    <el-table :data="state.singleEvents" border empty-text="暂无事件数据">
                                        <el-table-column prop="event_id" label="ID" width="90" />
                                        <el-table-column prop="merchant_name" label="商户" min-width="180" />
                                        <el-table-column prop="content" label="内容" min-width="220" />
                                        <el-table-column prop="result" label="失败原因" min-width="220" />
                                        <el-table-column prop="amount" label="金额" width="140" />
                                        <el-table-column prop="status" label="状态" width="120" />
                                    </el-table>
                                </div>
                            </el-card>
                        </div>

                        <div v-show="state.mode === 'batch'" class="batch-mode mt16">
                            <div class="section-head panel-head">批量兑换</div>

                            <div class="batch-toolbar">
                                <el-button type="primary" :disabled="state.batchRunning" @click="state.batchAddDialogVisible = true">添加卡密</el-button>
                                <el-button type="success" :disabled="!canStartBatch" :loading="state.batchRunning" @click="startBatchExchange">
                                    开始兑换
                                </el-button>
                                <el-button
                                    class="batch-check-btn"
                                    type="success"
                                    :disabled="state.batchRunning || state.batchDraftItems.length === 0"
                                    :loading="state.batchStatusChecking"
                                    @click="checkBatchCardStatuses"
                                >
                                    只查询不兑换
                                </el-button>
                                <el-button :disabled="!canCopyAllSuccess" @click="openCopySuccessDialog">复制成功项</el-button>
                                <el-button type="danger" plain :disabled="!canRetryFailed" @click="retryFailedRows">失败重试</el-button>
                            </div>

                            <div class="batch-stats">
                                <span>待兑换池：{{ state.batchDraftItems.length }} 条</span>
                                <span>总卡密数量：{{ state.batchRows.length }}</span>
                                <span>兑换成功：{{ batchSuccessCount }}</span>
                                <span>失败：{{ batchFailedCount }}</span>
                                <span>已到期：{{ batchExpiredCount }}</span>
                                <span>已耗时：{{ formatElapsed(state.batchElapsedSeconds) }}</span>
                            </div>

                            <el-table :data="pagedBatchRows" border row-key="id" empty-text="暂无批量兑换数据">
                                <el-table-column label="序号" width="90">
                                    <template #default="{ $index }">
                                        {{ getBatchRowSerial($index) }}
                                    </template>
                                </el-table-column>
                                <el-table-column prop="secret" label="卡密" min-width="220" />
                                <el-table-column label="兑换状态" width="150">
                                    <template #default="{ row }">
                                        <span class="status-cell">
                                            <span class="status-dot" :class="`status-${row.status}`"></span>
                                            {{ getBatchStatusText(row.status) }}
                                        </span>
                                    </template>
                                </el-table-column>
                                <el-table-column label="操作" min-width="780">
                                    <template #default="{ row }">
                                        <div v-if="row.status === 'success'" class="batch-action-wrap">
                                            <div class="batch-action-group">
                                                <el-button class="table-op-btn" size="small" @click="copyBatchField(row, 'cardNumber', '卡号')">复制卡号</el-button>
                                                <el-button class="table-op-btn" size="small" @click="copyBatchField(row, 'cvv', 'CVV')">复制CVV</el-button>
                                                <el-button class="table-op-btn" size="small" @click="copyBatchField(row, 'expiry', '有效期')">复制有效期</el-button>
                                                <el-button class="table-op-btn" size="small" @click="copyBatchField(row, 'fullAddress', '地址')">复制地址</el-button>
                                                <el-button class="table-op-btn" type="warning" size="small" @click="onBatchQuickCopy(row)">一键复制</el-button>
                                            </div>
                                            <el-button class="table-detail-btn" type="primary" size="small" @click="openBatchDetail(row)">查看详情</el-button>
                                        </div>
                                        <div v-else-if="row.status === 'failed'" class="batch-action-wrap">
                                            <div class="batch-fail-text">卡密：{{ row.secret }} 兑换失败</div>
                                            <el-button class="table-detail-btn" type="primary" size="small" @click="openBatchDetail(row)">查看详情</el-button>
                                        </div>
                                        <div v-else class="batch-action-wrap">
                                            <div class="batch-action-placeholder">-</div>
                                            <el-button class="table-detail-btn" type="primary" size="small" disabled>查看详情</el-button>
                                        </div>
                                    </template>
                                </el-table-column>
                            </el-table>

                            <div class="batch-pagination">
                                <el-pagination
                                    layout="total, prev, pager, next"
                                    :total="state.batchRows.length"
                                    :page-size="state.batchPageSize"
                                    :current-page="state.batchCurrentPage"
                                    @current-change="onBatchPageChange"
                                />
                            </div>

                            <el-card v-if="state.batchStatusRows.length > 0" class="batch-status-card" shadow="never">
                                <template #header>批量查询结果</template>
                                <el-table :data="state.batchStatusRows" border row-key="index" empty-text="暂无查询结果">
                                    <el-table-column prop="index" label="序号" width="90" />
                                    <el-table-column prop="secret" label="卡密" min-width="260" />
                                    <el-table-column prop="statusText" label="兑换状态" width="160" />
                                    <el-table-column label="操作" min-width="220">
                                        <template #default="{ row }">
                                            <span>{{ row.operationText || '-' }}</span>
                                        </template>
                                    </el-table-column>
                                </el-table>
                            </el-card>
                        </div>
                    </el-card>
                </el-col>
            </el-row>
        </el-main>

        <el-dialog v-model="state.batchAddDialogVisible" title="添加卡密" width="620px">
            <el-form label-width="110px">
                <el-form-item label="输入卡密">
                    <el-input v-model="state.batchAddForm.secretText" type="textarea" :rows="8" placeholder="请输入卡密，一行一个" resize="none" />
                </el-form-item>
                <el-form-item label="选择商户">
                    <el-select
                        v-model="state.batchAddForm.merchantId"
                        class="w100"
                        clearable
                        :loading="state.merchantOptionsLoading"
                        placeholder="请选择商户，0刀卡无需填"
                        @change="onBatchAddMerchantChange"
                    >
                        <el-option v-for="item in state.merchantOptions" :key="item.id" :label="item.name" :value="item.id" />
                    </el-select>
                </el-form-item>
            </el-form>
            <template #footer>
                <el-button @click="state.batchAddDialogVisible = false">取消</el-button>
                <el-button type="primary" @click="confirmBatchAdd">确定</el-button>
            </template>
        </el-dialog>

        <el-dialog v-model="state.copyDialogVisible" title="选择复制格式" width="560px">
            <el-form label-width="110px">
                <el-form-item label="复制格式">
                    <el-select v-model="state.copyFormat" class="w100">
                        <el-option v-for="item in COPY_FORMAT_OPTIONS" :key="item.value" :label="item.label" :value="item.value" />
                    </el-select>
                </el-form-item>
            </el-form>
            <template #footer>
                <el-button @click="state.copyDialogVisible = false">取消</el-button>
                <el-button type="primary" @click="copySuccessItems">确定</el-button>
            </template>
        </el-dialog>

        <el-dialog v-model="state.batchQuickCopyDialogVisible" title="选择一键复制格式" width="620px">
            <el-form label-width="120px">
                <el-form-item label="复制格式">
                    <el-select v-model="state.batchQuickCopyFormat" class="w100">
                        <el-option v-for="item in COPY_FORMAT_OPTIONS" :key="item.value" :label="item.label" :value="item.value" />
                    </el-select>
                </el-form-item>
                <el-form-item label="应用到后续复制">
                    <el-checkbox v-model="state.batchQuickCopyApplyNext">勾选后记住格式，直到刷新页面</el-checkbox>
                </el-form-item>
            </el-form>
            <template #footer>
                <el-button @click="closeBatchQuickCopyDialog">取消</el-button>
                <el-button type="primary" @click="confirmBatchQuickCopy">确定</el-button>
            </template>
        </el-dialog>

        <el-dialog v-model="state.batchDetailVisible" title="卡密详情" width="1280px" class="batch-detail-dialog">
            <template v-if="state.batchDetailRow">
                <template v-if="state.batchDetailRow.status === 'success'">
                    <div class="info-grid">
                        <el-card class="info-panel" shadow="never">
                            <template #header>卡片信息（点击可自动复制）</template>
                            <div v-for="item in batchDetailCardRows" :key="item.label" class="copy-row" @click="copyInfoValue(item.value)">
                                <span class="copy-label">{{ item.label }}</span>
                                <span class="copy-value">{{ item.value }}</span>
                            </div>
                        </el-card>

                        <el-card class="info-panel" shadow="never">
                            <template #header>地址信息（点击可自动复制）</template>
                            <div v-for="item in batchDetailAddressRows" :key="item.label" class="copy-row" @click="copyInfoValue(item.value)">
                                <span class="copy-label">{{ item.label }}</span>
                                <span class="copy-value">{{ item.value }}</span>
                            </div>
                        </el-card>
                    </div>

                    <div class="event-box mt16">
                        <div class="event-toolbar">
                            <el-button
                                class="event-load-btn"
                                type="primary"
                                :loading="state.batchDetailLoading"
                                :disabled="state.batchDetailLoading || state.batchDetailEventsCooldown > 0"
                                @click="loadBatchDetailEvents"
                            >
                                获取消费记录{{ state.batchDetailEventsCooldown > 0 ? ' ' + state.batchDetailEventsCooldown : '' }}
                            </el-button>
                        </div>
                        <el-table class="batch-detail-events-table" :data="state.batchDetailEvents" border size="small" :loading="state.batchDetailLoading" empty-text="暂无事件数据">
                            <el-table-column prop="event_id" label="ID" width="80" />
                            <el-table-column prop="merchant_name" label="商户" min-width="170" />
                            <el-table-column prop="content" label="内容" min-width="220" />
                            <el-table-column prop="result" label="失败原因" min-width="210" />
                            <el-table-column prop="amount" label="金额" width="120" />
                            <el-table-column prop="status" label="状态" width="110" />
                        </el-table>
                    </div>
                </template>
                <template v-else>
                    <el-card class="info-panel failure-detail-panel" shadow="never">
                        <div class="copy-row">
                            <span class="copy-label">卡密</span>
                            <span class="copy-value">{{ state.batchDetailRow.secret || '-' }}</span>
                        </div>
                        <div class="copy-row">
                            <span class="copy-label">错误信息</span>
                            <span class="copy-value">{{ state.batchDetailRow.message || '兑换失败' }}</span>
                        </div>
                    </el-card>
                </template>
            </template>
        </el-dialog>
    </div>
</template>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import { ElMessage, ElNotification } from 'element-plus'
import { API_BASE } from './config'

const COPY_FORMAT_OPTIONS = [
    { label: '卡号 cvv 有效期（中间用空格）', value: 'card_cvv_exp_space' },
    { label: '卡号----cvv----有效期----完整地址（中间用----）', value: 'card_cvv_exp_addr_dash' },
    { label: '卡号 有效期 cvv（中间用空格）', value: 'card_exp_cvv_space' },
    { label: '卡号----有效期----cvv----完整地址（中间用----）', value: 'card_exp_cvv_addr_dash' },
]

const state = reactive({
    mode: 'single',
    noticeText: '0刀卡无需填，选择后，该卡只能用于该商户，请慎重！不懂的点这里查看教程。',
    cardHeadLoading: false,
    cardHeadSelectionLocked: false,
    cardHeadOptions: [],
    selectedPlatformId: '',
    merchantOptionsLoading: false,
    merchantOptions: [],
    singleIssuing: false,
    singleStatusChecking: false,
    singleEventsLoading: false,
    singleEventsCooldown: 0,
    singleForm: {
        secret: '',
        merchantId: '',
    },
    singleMerchantName: '',
    singleResult: null,
    singleEvents: [],
    batchRunning: false,
    batchRunCompleted: false,
    batchRows: [],
    batchStatusChecking: false,
    batchStatusRows: [],
    batchDraftItems: [],
    batchCurrentPage: 1,
    batchPageSize: 20,
    batchElapsedSeconds: 0,
    batchStartedAt: 0,
    batchAddDialogVisible: false,
    batchAddForm: {
        secretText: '',
        merchantId: '',
        merchantName: '',
    },
    copyDialogVisible: false,
    copyFormat: 'card_cvv_exp_space',
    batchQuickCopyDialogVisible: false,
    batchQuickCopyFormat: 'card_cvv_exp_space',
    batchQuickCopyApplyNext: false,
    batchQuickCopyTargetRowId: '',
    batchQuickCopySavedFormat: '',
    batchDetailVisible: false,
    batchDetailLoading: false,
    batchDetailEventsCooldown: 0,
    batchDetailRow: null,
    batchDetailEvents: [],
})

let batchRowSeed = 1
let batchTimer = null
let singleEventsCooldownTimer = null
let batchDetailEventsCooldownTimer = null
const singleEventAnchorRef = ref(null)

const batchSuccessCount = computed(() => state.batchRows.filter((row) => row.status === 'success').length)
const batchFailedCount = computed(() => state.batchRows.filter((row) => row.status === 'failed').length)
const batchExpiredCount = computed(() => state.batchRows.filter((row) => row.status === 'expired').length)

const pagedBatchRows = computed(() => {
    const start = (state.batchCurrentPage - 1) * state.batchPageSize
    return state.batchRows.slice(start, start + state.batchPageSize)
})

const canStartBatch = computed(() => !state.batchRunning && state.batchDraftItems.length > 0)
const canCopyAllSuccess = computed(() => !state.batchRunning && state.batchRows.some((row) => row.status === 'success'))
const canRetryFailed = computed(
    () => !state.batchRunning && state.batchRunCompleted && state.batchRows.some((row) => row.status === 'failed')
)

const singleCardRows = computed(() => {
    const item = state.singleResult
    if (!item) return []
    return [
        { label: '卡号', value: item.cardNumber || '-' },
        { label: 'CVV', value: item.cvv || '-' },
        { label: '有效期', value: item.expiry || '-' },
        { label: '限制商户', value: item.merchantName || '-' },
        { label: '金额', value: item.balanceText || '-' },
        { label: '剩余小时', value: item.remainingHoursText || '-' },
        { label: '兑换时间', value: formatDateTime(item.exchangeTime) },
        { label: '过期时间', value: formatDateTime(item.expireTime) },
    ]
})

const singleAddressRows = computed(() => {
    const item = state.singleResult
    if (!item) return []
    return [
        { label: '完整地址', value: item.fullAddress || '-' },
        { label: '街道', value: item.streetAddress || '-' },
        { label: '城市', value: item.city || '-' },
        { label: '州/省', value: item.fullState || '-' },
        { label: '邮编', value: item.postalCode || '-' },
        { label: '国家', value: item.country || '-' },
    ]
})

const batchDetailCardRows = computed(() => {
    const item = state.batchDetailRow
    if (!item) return []
    return [
        { label: '卡密', value: item.secret || '-' },
        { label: '状态', value: getBatchStatusText(item.status) },
        { label: '卡号', value: item.cardNumber || '-' },
        { label: 'CVV', value: item.cvv || '-' },
        { label: '有效期', value: item.expiry || '-' },
        { label: '限制商户', value: item.merchantName || '-' },
        { label: '过期时间', value: formatDateTime(item.expireTime) },
    ]
})

const batchDetailAddressRows = computed(() => {
    const item = state.batchDetailRow
    if (!item) return []
    return [
        { label: '完整地址', value: item.fullAddress || '-' },
        { label: '街道', value: item.streetAddress || '-' },
        { label: '城市', value: item.city || '-' },
        { label: '州/省', value: item.fullState || '-' },
        { label: '邮编', value: item.postalCode || '-' },
        { label: '国家', value: item.country || '-' },
    ]
})

function makeBatchRowId() {
    const id = batchRowSeed
    batchRowSeed += 1
    return 'batch-row-' + id
}

async function request(method, path, options = {}) {
    const params = options.params || null
    const data = options.data || null
    const silentError = options.silentError === true

    let url = API_BASE + path
    if (params) {
        const query = new URLSearchParams()
        Object.keys(params).forEach((key) => {
            const val = params[key]
            if (val === undefined || val === null) return
            query.append(key, String(val))
        })
        const qs = query.toString()
        if (qs !== '') {
            url += (url.indexOf('?') >= 0 ? '&' : '?') + qs
        }
    }

    const fetchOptions = {
        method,
        headers: {
            Accept: 'application/json;charset=utf-8',
        },
    }
    if (method !== 'GET') {
        fetchOptions.headers['Content-Type'] = 'application/json;charset=utf-8'
        fetchOptions.body = JSON.stringify(data || {})
    }

    let response
    try {
        response = await fetch(url, fetchOptions)
    } catch (_error) {
        const msg = '网络请求失败'
        if (!silentError) {
            ElNotification({ type: 'error', message: msg })
        }
        const e = new Error(msg)
        e.msg = msg
        throw e
    }

    let payload = null
    const text = await response.text()
    if (text.trim() !== '') {
        try {
            payload = JSON.parse(text)
        } catch (_e) {
            payload = null
        }
    }

    if (!response.ok) {
        const msg = payload && payload.msg ? String(payload.msg) : '请求失败，HTTP ' + response.status
        if (!silentError) {
            ElNotification({ type: 'error', message: msg })
        }
        const e = new Error(msg)
        e.msg = msg
        throw e
    }

    if (!payload || typeof payload !== 'object') {
        const msg = '响应格式错误'
        if (!silentError) {
            ElNotification({ type: 'error', message: msg })
        }
        const e = new Error(msg)
        e.msg = msg
        throw e
    }

    if (Number(payload.code) !== 1) {
        const msg = String(payload.msg || '请求失败')
        if (!silentError) {
            ElNotification({ type: 'error', message: msg })
        }
        const e = new Error(msg)
        e.msg = msg
        throw e
    }

    return payload
}

function issueCard(payload, requestOption = {}) {
    return request('POST', '/api/open/card/redeem', { data: payload, silentError: requestOption.silentError === true })
}

function getCardInfos(cardKey) {
    return request('POST', '/api/open/card/transactions', {
        data: {
            card_key: String(cardKey || '').trim(),
        },
    })
}

function getRedeemNotice() {
    return request('GET', '/api/open/config/notice')
}

function getMerchantList() {
    return request('POST', '/api/open/merchant/list')
}

function getPlatformCapacity() {
    return request('GET', '/api/open/platform/capacity')
}

function getCardStatus(cardKey) {
    return request('GET', '/api/open/card/status', {
        params: {
            card_key: String(cardKey || '').trim(),
        },
    })
}

function normalizeIssueResult(raw) {
    const value = raw || {}
    const balance = Number(value.available_amount ?? value.balance ?? value.amount ?? 0)
    const remainingHours = Number(value.remaining_hours ?? value.available_hours ?? 0)
    const expiryText = String(value.exp || value.expiry || value.expire_at || '').trim()
    const expiryMonth = String(value.expiry_month || value.exp_month || '').trim()
    const expiryYear = String(value.expiry_year || value.exp_year || '').trim()
    const normalizedExpiry = expiryText !== '' ? expiryText : expiryMonth !== '' && expiryYear !== '' ? expiryMonth + '/' + expiryYear.slice(-2) : ''
    return {
        cardNumber: String(value.card_number || value.card_no || ''),
        cvv: String(value.cvv || ''),
        expiry: normalizedExpiry,
        remainingHoursText: Number.isFinite(remainingHours) ? String(remainingHours) : '0',
        balanceText: Number.isFinite(balance) ? String(balance) : '0',
        exchangeTime: Number(value.redeem_time || value.exchange_time || value.used_time || 0),
        expireTime: Number(value.expire_time || value.expired_at || value.expire_ts || 0),
        merchantName: String(value.limit_merchant || value.merchant_name || value.merchantName || ''),
        fullAddress: String(value.full_billing_address || value.full_address || value.address || ''),
        streetAddress: String(value.street_address || value.address_line1 || ''),
        city: String(value.city || value.town || ''),
        fullState: String(value.full_state || value.state || ''),
        postalCode: String(value.postal_code || value.zip || ''),
        country: String(value.country || value.country_code || ''),
    }
}

function normalizeTransactionRows(raw) {
    const rows = Array.isArray(raw)
        ? raw
        : Array.isArray(raw && raw.transactions)
          ? raw.transactions
          : Array.isArray(raw && raw.list)
            ? raw.list
            : Array.isArray(raw && raw.rows)
              ? raw.rows
              : []
    return rows.map((item) => {
        const row = item || {}
        const eventTimeRaw = row.date || row.event_time || row.created_at || row.time || row.transaction_time || ''
        const eventTimeDate = String(eventTimeRaw || '').trim() === '' ? null : new Date(eventTimeRaw)
        const eventTime =
            eventTimeDate && !Number.isNaN(eventTimeDate.getTime())
                ? eventTimeDate.getFullYear() +
                  '-' +
                  String(eventTimeDate.getMonth() + 1).padStart(2, '0') +
                  '-' +
                  String(eventTimeDate.getDate()).padStart(2, '0') +
                  ' ' +
                  String(eventTimeDate.getHours()).padStart(2, '0') +
                  ':' +
                  String(eventTimeDate.getMinutes()).padStart(2, '0') +
                  ':' +
                  String(eventTimeDate.getSeconds()).padStart(2, '0')
                : String(eventTimeRaw || '-')

        const currency = String(row.currency || row.currency_code || '').trim()
        const amountValue = row.transactionAmount ?? row.amount ?? row.billing_amount ?? row.money
        const isNumericAmount = typeof amountValue === 'number' || /^-?\d+(\.\d+)?$/.test(String(amountValue || '').trim())
        const amountText =
            amountValue === undefined || amountValue === null || String(amountValue).trim() === ''
                ? '-'
                : currency !== '' && isNumericAmount
                  ? String(amountValue) + ' ' + currency
                  : String(amountValue)
        const status = String(row.status || '').trim()
        const failureReasonRaw = String(row.failureReason || row.failure_reason || row.result || '').trim()
        const failureReason = failureReasonRaw !== '' ? failureReasonRaw : status !== '' ? status : '-'
        const content = String(row.content || row.description || row.memo || '-').trim() || '-'
        const statusText = status !== '' ? status : '-'

        return {
            event_id: String(row.event_id || row.id || row.transaction_id || '-'),
            event_time: eventTime,
            merchant_name: String(row.merchant || (row.merchantDetails && row.merchantDetails.name) || row.merchant_name || row.merchantName || '-'),
            content,
            amount: amountText,
            result: failureReason,
            status: statusText,
        }
    })
}

function extractApiMessage(error) {
    const msg = String((error && (error.msg || error.message)) || '').trim()
    return msg !== '' ? msg : '兑换失败'
}

function isExpiredMessage(msg) {
    const text = String(msg || '').toLowerCase()
    return (
        text.indexOf('到期') >= 0 ||
        text.indexOf('expired') >= 0 ||
        text.indexOf('cancelled') >= 0 ||
        text.indexOf('canceled') >= 0 ||
        text.indexOf('已取消') >= 0
    )
}

function isAlreadyIssuedMessage(msg) {
    const text = String(msg || '').toLowerCase()
    return text.indexOf('already in use') >= 0 || text.indexOf('已兑换') >= 0 || text.indexOf('已经兑换') >= 0
}

function getBatchStatusText(status) {
    if (status === 'success') return '兑换成功'
    if (status === 'failed') return '兑换失败'
    if (status === 'expired') return '已经到期'
    if (status === 'processing') return '兑换中'
    return '等待兑换'
}

function getBatchRowSerial(indexOnPage) {
    return (state.batchCurrentPage - 1) * state.batchPageSize + Number(indexOnPage) + 1
}

function formatDateTime(timestamp) {
    if (!timestamp) return '-'
    const d = new Date(timestamp * 1000)
    const y = d.getFullYear()
    const m = String(d.getMonth() + 1).padStart(2, '0')
    const day = String(d.getDate()).padStart(2, '0')
    const h = String(d.getHours()).padStart(2, '0')
    const i = String(d.getMinutes()).padStart(2, '0')
    const s = String(d.getSeconds()).padStart(2, '0')
    return y + '-' + m + '-' + day + ' ' + h + ':' + i + ':' + s
}

function getCardStatusText(status) {
    const statusMap = {
        unused: '卡密未使用',
        used: '卡密已使用',
        redeeming: '卡密兑换中',
        none: '卡密不存在',
    }
    const key = String(status || '').trim().toLowerCase()
    return statusMap[key] || ('未知状态：' + (key || '-'))
}

function lockCardHeadSelection(shouldLock) {
    state.cardHeadSelectionLocked = shouldLock === true
}

function startSingleEventsCooldown(seconds = 10) {
    if (singleEventsCooldownTimer) {
        clearInterval(singleEventsCooldownTimer)
        singleEventsCooldownTimer = null
    }
    state.singleEventsCooldown = seconds
    singleEventsCooldownTimer = setInterval(() => {
        if (state.singleEventsCooldown <= 1) {
            state.singleEventsCooldown = 0
            clearInterval(singleEventsCooldownTimer)
            singleEventsCooldownTimer = null
            return
        }
        state.singleEventsCooldown -= 1
    }, 1000)
}

function startBatchDetailEventsCooldown(seconds = 10) {
    if (batchDetailEventsCooldownTimer) {
        clearInterval(batchDetailEventsCooldownTimer)
        batchDetailEventsCooldownTimer = null
    }
    state.batchDetailEventsCooldown = seconds
    batchDetailEventsCooldownTimer = setInterval(() => {
        if (state.batchDetailEventsCooldown <= 1) {
            state.batchDetailEventsCooldown = 0
            clearInterval(batchDetailEventsCooldownTimer)
            batchDetailEventsCooldownTimer = null
            return
        }
        state.batchDetailEventsCooldown -= 1
    }, 1000)
}

function formatElapsed(seconds) {
    const m = Math.floor(seconds / 60)
    const s = seconds % 60
    return String(m).padStart(2, '0') + ':' + String(s).padStart(2, '0')
}

function normalizeMerchantOptions(raw) {
    const rows = Array.isArray(raw) ? raw : []
    return rows
        .map((item) => ({
            id: String((item && (item.id ?? item.merchant_dict_id ?? item.merchantId)) || '').trim(),
            name: String((item && (item.name || item.merchant_name || item.title)) || '').trim(),
        }))
        .filter((item) => item.id !== '' && item.name !== '')
}

function normalizeCardHeadOptions(raw) {
    const rows = Array.isArray(raw) ? raw : []
    return rows
        .map((item) => {
            const platformId = String((item && item.platform_id) ?? '').trim()
            const capacityRate = Number(item && item.capacity_rate)
            return {
                platformId,
                capacityRate: Number.isFinite(capacityRate) ? capacityRate : 0,
                cardHeadNum: String((item && item.card_head_num) || '').trim(),
                region: String((item && item.region) || '').trim(),
                disabled: capacityRate === -1,
            }
        })
        .filter((item) => item.platformId !== '')
}

function formatCardHeadRate(rate) {
    if (!Number.isFinite(rate)) return '0%'
    if (rate <= 0) return '0%'
    return rate + '%'
}

function getCardHeadRateWidth(item) {
    if (!item || item.disabled) return '100%'
    const rate = Number(item.capacityRate)
    if (!Number.isFinite(rate) || rate <= 0) return '0%'
    return Math.min(100, Math.max(0, rate)) + '%'
}

function onSelectCardHead(item) {
    if (state.cardHeadSelectionLocked) return
    if (!item || item.disabled) return
    state.selectedPlatformId = item.platformId
}

async function loadCardHeadOptions() {
    state.cardHeadLoading = true
    try {
        const res = await getPlatformCapacity()
        const data = res && res.data ? res.data : null
        const list = Array.isArray(data && data.list) ? data.list : []
        state.cardHeadOptions = normalizeCardHeadOptions(list)
        if (state.selectedPlatformId !== '') {
            const exists = state.cardHeadOptions.some((item) => item.platformId === state.selectedPlatformId && !item.disabled)
            if (!exists) {
                state.selectedPlatformId = ''
            }
        }
    } catch (_e) {
        state.cardHeadOptions = []
        state.selectedPlatformId = ''
    } finally {
        state.cardHeadLoading = false
    }
}

async function loadMerchantOptions() {
    state.merchantOptionsLoading = true
    try {
        const res = await getMerchantList()
        const data = res && res.data ? res.data : null
        const list = Array.isArray(data) ? data : Array.isArray(data && data.list) ? data.list : []
        state.merchantOptions = normalizeMerchantOptions(list)
        if (state.singleForm.merchantId !== '' && !state.merchantOptions.some((item) => item.id === state.singleForm.merchantId)) {
            state.singleForm.merchantId = ''
            state.singleMerchantName = ''
        }
        if (state.batchAddForm.merchantId !== '' && !state.merchantOptions.some((item) => item.id === state.batchAddForm.merchantId)) {
            state.batchAddForm.merchantId = ''
            state.batchAddForm.merchantName = ''
        }
    } catch (_e) {
        state.merchantOptions = []
    } finally {
        state.merchantOptionsLoading = false
    }
}

function onSingleMerchantChange(merchantId) {
    if (!merchantId) {
        state.singleMerchantName = ''
        return
    }
    const row = state.merchantOptions.find((item) => item.id === merchantId)
    state.singleMerchantName = row ? row.name : ''
}

function buildIssuePayload(secret, merchantId, platformIdOverride = '') {
    const payload = {
        card_key: String(secret || '').trim(),
    }
    const id = String(merchantId || '').trim()
    if (id !== '') {
        payload.merchant_dict_id = id
    }
    const platformId = String(platformIdOverride || state.selectedPlatformId || '').trim()
    if (platformId !== '') {
        payload.platform_id = Number(platformId)
    }
    return payload
}

async function submitSingleIssue() {
    if (String(state.singleForm.secret || '').trim() === '') {
        ElMessage.warning('请输入卡密')
        return
    }

    const payload = buildIssuePayload(state.singleForm.secret, state.singleForm.merchantId)
    state.singleIssuing = true
    try {
        const res = await issueCard(payload)
        state.singleResult = normalizeIssueResult(res.data)
        state.singleEvents = []
        ElNotification({
            type: 'success',
            message: res.msg || '兑换成功',
        })
    } finally {
        state.singleIssuing = false
    }
}

async function checkSingleCardStatus() {
    const cardKey = String(state.singleForm.secret || '').trim()
    if (cardKey === '') {
        ElMessage.warning('请先输入卡密')
        return
    }

    state.singleStatusChecking = true
    try {
        const res = await getCardStatus(cardKey)
        const status = String((res.data && res.data.status) || '').trim().toLowerCase()
        const message = getCardStatusText(status)
        const usedTime = Number((res.data && res.data.used_time) || 0)
        const usedTimeText = usedTime > 0 ? formatDateTime(usedTime) : ''
        if (status === 'none') {
            ElMessage.error(message)
            return
        }
        if (status === 'redeeming') {
            ElMessage.warning(message)
            return
        }
        if (status === 'used' && usedTimeText !== '' && usedTimeText !== '-') {
            ElMessage.success('卡密已使用，使用时间：' + usedTimeText)
            return
        }
        ElMessage.success(message)
    } finally {
        state.singleStatusChecking = false
    }
}

async function checkBatchCardStatuses() {
    if (state.batchDraftItems.length === 0) {
        ElMessage.warning('请先添加卡密')
        return
    }

    state.batchStatusChecking = true
    state.batchStatusRows = []
    try {
        for (let index = 0; index < state.batchDraftItems.length; index++) {
            const item = state.batchDraftItems[index]
            const secret = String(item.secret || '').trim()
            if (secret === '') {
                continue
            }

            try {
                const res = await getCardStatus(secret)
                const status = String((res.data && res.data.status) || '').trim().toLowerCase()
                const usedTime = Number((res.data && res.data.used_time) || 0)
                const usedTimeText = usedTime > 0 ? formatDateTime(usedTime) : '-'
                state.batchStatusRows.push({
                    index: index + 1,
                    secret,
                    statusText: getCardStatusText(status),
                    operationText: status === 'used' && usedTime > 0 ? '使用时间：' + usedTimeText : '-',
                })
            } catch (error) {
                state.batchStatusRows.push({
                    index: index + 1,
                    secret,
                    statusText: '查询失败',
                    operationText: extractApiMessage(error),
                })
            }
        }
    } finally {
        state.batchStatusChecking = false
    }
}

function resetSingleForm() {
    state.singleForm.secret = ''
    state.singleForm.merchantId = ''
    state.singleMerchantName = ''
    state.singleResult = null
    state.singleEvents = []
}

async function loadSingleEvents() {
    const secret = String(state.singleForm.secret || '').trim()
    if (secret === '') {
        ElMessage.warning('请先输入卡密')
        return
    }
    if (state.singleEventsCooldown > 0) {
        return
    }

    startSingleEventsCooldown(10)
    state.singleEventsLoading = true
    try {
        const res = await getCardInfos(secret)
        state.singleEvents = normalizeTransactionRows(res.data)
    } finally {
        state.singleEventsLoading = false
    }
}

async function fetchAndScrollSingleEvents() {
    const secret = String(state.singleForm.secret || '').trim()
    if (secret === '') {
        ElMessage.warning('请先输入卡密')
        return
    }

    await nextTick()
    if (singleEventAnchorRef.value && typeof singleEventAnchorRef.value.scrollIntoView === 'function') {
        singleEventAnchorRef.value.scrollIntoView({ behavior: 'smooth', block: 'start' })
    }
    await loadSingleEvents()
}

function onBatchAddMerchantChange(merchantId) {
    if (!merchantId) {
        state.batchAddForm.merchantName = ''
        return
    }
    const row = state.merchantOptions.find((item) => item.id === merchantId)
    state.batchAddForm.merchantName = row ? row.name : ''
}

function confirmBatchAdd() {
    const secrets = String(state.batchAddForm.secretText || '')
        .split(/\r?\n/)
        .map((row) => row.trim())
        .filter((row) => row !== '')

    if (secrets.length === 0) {
        ElMessage.warning('请先输入卡密')
        return
    }

    const merchantId = String(state.batchAddForm.merchantId || '').trim()
    const merchantName = String(state.batchAddForm.merchantName || '').trim()
    if (merchantId !== '' && merchantName === '') {
        ElMessage.warning('请选择有效商户')
        return
    }

    secrets.forEach((secret) => {
        state.batchDraftItems.push({
            secret,
            merchantId,
            merchantName,
        })
    })

    state.batchAddForm.secretText = ''
    state.batchAddForm.merchantId = ''
    state.batchAddForm.merchantName = ''
    state.batchAddDialogVisible = false
    ElMessage.success('已添加 ' + secrets.length + ' 条卡密')
}

function startBatchTimer() {
    stopBatchTimer()
    state.batchStartedAt = Date.now()
    state.batchElapsedSeconds = 0
    batchTimer = setInterval(() => {
        state.batchElapsedSeconds = Math.floor((Date.now() - state.batchStartedAt) / 1000)
    }, 1000)
}

function stopBatchTimer() {
    if (batchTimer) {
        clearInterval(batchTimer)
        batchTimer = null
    }
    if (state.batchStartedAt > 0) {
        state.batchElapsedSeconds = Math.floor((Date.now() - state.batchStartedAt) / 1000)
    }
}

function applyIssueResultToRow(row, result, message) {
    row.cardNumber = result.cardNumber
    row.cvv = result.cvv
    row.expiry = result.expiry
    row.expireTime = result.expireTime
    row.merchantName = result.merchantName || row.merchantName
    row.fullAddress = result.fullAddress
    row.streetAddress = result.streetAddress
    row.city = result.city
    row.fullState = result.fullState
    row.postalCode = result.postalCode
    row.country = result.country
    row.message = message

    const now = Math.floor(Date.now() / 1000)
    const expiredByTime = result.expireTime > 0 && result.expireTime <= now
    row.status = expiredByTime || isExpiredMessage(message) ? 'expired' : 'success'
}

async function waitNextBatchIssue() {
    await new Promise((resolve) => {
        window.setTimeout(resolve, 5000)
    })
}

async function issueOneBatchRow(row) {
    const payload = buildIssuePayload(row.secret, row.merchantId, row.platformId || '')
    try {
        const res = await issueCard(payload, { silentError: true })
        const message = String(res.msg || '兑换成功')
        const result = normalizeIssueResult(res.data)
        applyIssueResultToRow(row, result, message)
        return {
            shouldDelayNext: row.status === 'success' && !isAlreadyIssuedMessage(message),
        }
    } catch (error) {
        const msg = extractApiMessage(error)
        row.status = isExpiredMessage(msg) ? 'expired' : 'failed'
        row.message = msg
        return {
            shouldDelayNext: false,
        }
    }
}

async function startBatchExchange() {
    if (state.batchDraftItems.length === 0) {
        ElMessage.warning('请先添加卡密')
        return
    }

    const selectedPlatformId = String(state.selectedPlatformId || '').trim()
    if (selectedPlatformId !== '') {
        lockCardHeadSelection(true)
    }

    state.batchRows = state.batchDraftItems.map((item) => ({
        id: makeBatchRowId(),
        secret: item.secret,
        merchantId: item.merchantId,
        merchantName: item.merchantName,
        platformId: selectedPlatformId,
        status: 'pending',
        expireTime: 0,
        cardNumber: '',
        cvv: '',
        expiry: '',
        fullAddress: '',
        streetAddress: '',
        city: '',
        fullState: '',
        postalCode: '',
        country: '',
        message: '',
    }))

    state.batchCurrentPage = 1
    state.batchRunCompleted = false
    state.batchRunning = true
    startBatchTimer()

    try {
        for (let index = 0; index < state.batchRows.length; index++) {
            const row = state.batchRows[index]
            row.status = 'processing'
            const issueMeta = await issueOneBatchRow(row)
            if (issueMeta.shouldDelayNext && index < state.batchRows.length - 1) {
                await waitNextBatchIssue()
            }
        }
        state.batchDraftItems = []
    } finally {
        lockCardHeadSelection(false)
        state.batchRunning = false
        state.batchRunCompleted = true
        stopBatchTimer()
    }
}

async function retryFailedRows() {
    const failedRows = state.batchRows.filter((row) => row.status === 'failed')
    if (failedRows.length === 0) {
        ElMessage.warning('当前没有可重试的失败卡密')
        return
    }

    const retryPlatformId = String((failedRows[0] && failedRows[0].platformId) || state.selectedPlatformId || '').trim()
    if (retryPlatformId !== '') {
        lockCardHeadSelection(true)
        failedRows.forEach((row) => {
            row.platformId = retryPlatformId
        })
    }

    state.batchRunCompleted = false
    state.batchRunning = true
    startBatchTimer()
    try {
        for (let index = 0; index < failedRows.length; index++) {
            const row = failedRows[index]
            row.status = 'processing'
            row.message = ''
            const issueMeta = await issueOneBatchRow(row)
            if (issueMeta.shouldDelayNext && index < failedRows.length - 1) {
                await waitNextBatchIssue()
            }
        }
    } finally {
        lockCardHeadSelection(false)
        state.batchRunning = false
        state.batchRunCompleted = true
        stopBatchTimer()
    }
}

function onBatchPageChange(page) {
    state.batchCurrentPage = page
}

async function copyToClipboard(text) {
    if (navigator.clipboard && navigator.clipboard.writeText) {
        await navigator.clipboard.writeText(text)
        return
    }

    const textarea = document.createElement('textarea')
    textarea.value = text
    textarea.style.position = 'fixed'
    textarea.style.opacity = '0'
    document.body.appendChild(textarea)
    textarea.focus()
    textarea.select()
    const ok = document.execCommand('copy')
    document.body.removeChild(textarea)
    if (!ok) {
        throw new Error('copy_failed')
    }
}

async function copyInfoValue(value) {
    const text = String(value || '').trim()
    if (text === '' || text === '-') {
        ElMessage.warning('暂无可复制内容')
        return
    }
    try {
        await copyToClipboard(text)
        ElMessage.success('已复制' + text)
    } catch (_e) {
        ElMessage.error('复制失败')
    }
}

function openCopySuccessDialog() {
    const rows = state.batchRows.filter((row) => row.status === 'success')
    if (rows.length === 0) {
        ElMessage.warning('暂无可复制的兑换成功卡密')
        return
    }
    state.copyDialogVisible = true
}

function normalizeCopyField(value) {
    const text = String(value || '').trim()
    if (text === '') return '-'
    return text.replace(/[\r\n]+/g, ' ')
}

function buildCopyLine(row, format) {
    const cardNumber = normalizeCopyField(row.cardNumber)
    const cvv = normalizeCopyField(row.cvv)
    const expiry = normalizeCopyField(row.expiry)
    const fullAddress = normalizeCopyField(row.fullAddress)

    if (format === 'card_cvv_exp_addr_dash') {
        return cardNumber + '----' + cvv + '----' + expiry + '----' + fullAddress
    }
    if (format === 'card_exp_cvv_space') {
        return cardNumber + ' ' + expiry + ' ' + cvv
    }
    if (format === 'card_exp_cvv_addr_dash') {
        return cardNumber + '----' + expiry + '----' + cvv + '----' + fullAddress
    }
    return cardNumber + ' ' + cvv + ' ' + expiry
}

async function copyBatchField(row, fieldName, fieldLabel) {
    if (row.status !== 'success') {
        ElMessage.warning('当前仅支持复制兑换成功行')
        return
    }
    const text = normalizeCopyField(row[fieldName])
    if (text === '-') {
        ElMessage.warning(fieldLabel + '为空，无法复制')
        return
    }
    try {
        await copyToClipboard(text)
        ElMessage.success('已复制' + fieldLabel)
    } catch (_e) {
        ElMessage.error('复制失败')
    }
}

function closeBatchQuickCopyDialog() {
    state.batchQuickCopyDialogVisible = false
    state.batchQuickCopyApplyNext = false
    state.batchQuickCopyTargetRowId = ''
}

async function copyBatchRowByFormat(row, format) {
    const text = buildCopyLine(row, format)
    await copyToClipboard(text)
}

function onBatchQuickCopy(row) {
    if (row.status !== 'success') {
        ElMessage.warning('当前仅支持一键复制兑换成功行')
        return
    }

    if (state.batchQuickCopySavedFormat !== '') {
        copyBatchRowByFormat(row, state.batchQuickCopySavedFormat)
            .then(() => {
                ElMessage.success('已按已保存格式复制')
            })
            .catch(() => {
                ElMessage.error('复制失败')
            })
        return
    }

    state.batchQuickCopyTargetRowId = row.id
    state.batchQuickCopyFormat = state.copyFormat
    state.batchQuickCopyApplyNext = false
    state.batchQuickCopyDialogVisible = true
}

async function confirmBatchQuickCopy() {
    const row = state.batchRows.find((item) => item.id === state.batchQuickCopyTargetRowId)
    if (!row || row.status !== 'success') {
        closeBatchQuickCopyDialog()
        ElMessage.warning('当前行不可复制')
        return
    }

    try {
        await copyBatchRowByFormat(row, state.batchQuickCopyFormat)
        if (state.batchQuickCopyApplyNext) {
            state.batchQuickCopySavedFormat = state.batchQuickCopyFormat
        }
        ElMessage.success('已复制该行数据')
        closeBatchQuickCopyDialog()
    } catch (_e) {
        ElMessage.error('复制失败')
    }
}

async function copySuccessItems() {
    const rows = state.batchRows.filter((row) => row.status === 'success')
    if (rows.length === 0) {
        ElMessage.warning('暂无可复制的兑换成功卡密')
        state.copyDialogVisible = false
        return
    }

    const text = rows
        .map((row) => buildCopyLine(row, state.copyFormat))
        .join('\n')

    try {
        await copyToClipboard(text)
        state.copyDialogVisible = false
        ElMessage.success('已复制 ' + rows.length + ' 条兑换成功的数据！')
    } catch (_e) {
        ElMessage.error('复制失败')
    }
}

async function loadBatchDetailEvents() {
    if (!state.batchDetailRow) return
    if (state.batchDetailEventsCooldown > 0) return
    startBatchDetailEventsCooldown(10)
    state.batchDetailLoading = true
    try {
        const res = await getCardInfos(state.batchDetailRow.secret)
        state.batchDetailEvents = normalizeTransactionRows(res.data)
    } finally {
        state.batchDetailLoading = false
    }
}

async function openBatchDetail(row) {
    state.batchDetailRow = { ...row }
    state.batchDetailVisible = true
    state.batchDetailEvents = []
    if (row.status === 'success') {
        await loadBatchDetailEvents()
    }
}

async function loadRedeemNotice() {
    try {
        const res = await getRedeemNotice()
        const noticeContent = String((res.data && res.data.notice_content) || '').trim()
        if (noticeContent !== '') {
            state.noticeText = noticeContent
        }
    } catch (_e) {
        // 公告读取失败时保留默认文案
    }
}

onMounted(() => {
    loadRedeemNotice()
    loadCardHeadOptions()
    loadMerchantOptions()
})

onBeforeUnmount(() => {
    stopBatchTimer()
    if (singleEventsCooldownTimer) {
        clearInterval(singleEventsCooldownTimer)
        singleEventsCooldownTimer = null
    }
    if (batchDetailEventsCooldownTimer) {
        clearInterval(batchDetailEventsCooldownTimer)
        batchDetailEventsCooldownTimer = null
    }
})
</script>
