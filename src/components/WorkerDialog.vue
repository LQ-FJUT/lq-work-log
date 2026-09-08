<script setup lang="ts">
import { computed, nextTick, ref, watch } from 'vue'
import { imageFileToAvatarDataUrl, isSingleVisibleEmoji } from '../avatar'
import { fenToCurrency, fenToInput, parseYuanToFen } from '../money'
import type { Site, Worker } from '../types'
import type {
  BatchWorkerItem,
  BatchWorkerResult,
  BatchWorkerSubmission,
  WorkerFormPayload,
} from '../worker-form'
import UiIcon from './UiIcon.vue'

type EntryMode = 'single' | 'batch'
type BatchRow = { id: number; name: string; rate: string }

const props = defineProps<{
  open: boolean
  mode: 'add' | 'edit'
  worker: Worker | null
  sites: readonly Site[]
  existingNames: readonly string[]
  submitting: boolean
  serverError: string
  batchResult: BatchWorkerResult | null
  rateTimeline?: readonly { month: string; rateFen: number }[]
}>()

const emit = defineEmits<{
  close: []
  archive: []
  'submit-single': [payload: WorkerFormPayload]
  'submit-batch': [submission: BatchWorkerSubmission]
}>()

const entryMode = ref<EntryMode>('single')
const name = ref('')
const rate = ref('')
const note = ref('')
const avatarDataUrl = ref<string | null>(null)
const avatarEmoji = ref<string | null>(null)
const defaultSiteId = ref<string | null>(null)
const localError = ref('')
const batchRows = ref<BatchRow[]>([])
let rowSequence = 0
let requestSequence = 0

const visibleError = computed(() => localError.value || props.serverError)
const duplicateNames = computed(() => {
  const existing = new Set(props.existingNames.map((item) => item.trim().toLocaleLowerCase('zh-CN')))
  const seen = new Set<string>()
  const duplicates = new Set<string>()
  for (const row of batchRows.value) {
    const candidate = row.name.trim()
    if (!candidate) continue
    const key = candidate.toLocaleLowerCase('zh-CN')
    if (existing.has(key) || seen.has(key)) duplicates.add(candidate)
    seen.add(key)
  }
  return [...duplicates]
})

function newRow(item?: BatchWorkerItem): BatchRow {
  return {
    id: ++rowSequence,
    name: item?.name ?? '',
    rate: item ? fenToInput(item.defaultDailyRateFen) : '',
  }
}

function reset(): void {
  const worker = props.mode === 'edit' ? props.worker : null
  entryMode.value = 'single'
  name.value = worker?.name ?? ''
  rate.value = fenToInput(worker?.defaultDailyRateFen ?? 0)
  note.value = worker?.note ?? ''
  avatarDataUrl.value = worker?.avatarDataUrl ?? null
  avatarEmoji.value = worker?.avatarEmoji ?? null
  defaultSiteId.value = worker?.defaultSiteId ?? null
  batchRows.value = [newRow()]
  localError.value = ''
}

watch(() => props.open, (open) => {
  if (open) reset()
}, { immediate: true })

watch(() => props.batchResult, (result) => {
  if (!result) return
  batchRows.value = result.remaining.length ? result.remaining.map((item) => newRow(item)) : [newRow()]
  localError.value = `已成功添加 ${result.completed} 人；剩余 ${result.remaining.length} 行尚未添加。${result.error ? ` ${result.error}` : ''}`
})

function setEntryMode(value: EntryMode): void {
  entryMode.value = value
  localError.value = ''
}

function appendRow(afterIndex = batchRows.value.length - 1): void {
  const row = newRow()
  batchRows.value.splice(afterIndex + 1, 0, row)
  void nextTick(() => document.querySelector<HTMLInputElement>(`[data-batch-worker-id="${row.id}"]`)?.focus())
}

function removeRow(index: number): void {
  if (batchRows.value.length === 1) batchRows.value = [newRow()]
  else batchRows.value.splice(index, 1)
}

async function useAvatarFile(file: File): Promise<void> {
  localError.value = ''
  try {
    avatarDataUrl.value = await imageFileToAvatarDataUrl(file)
    avatarEmoji.value = null
  } catch (error) {
    localError.value = error instanceof Error ? error.message : '头像处理失败。'
  }
}

async function onAvatarChange(event: Event): Promise<void> {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (file) await useAvatarFile(file)
  input.value = ''
}

function onAvatarPaste(event: ClipboardEvent): void {
  const file = [...(event.clipboardData?.files ?? [])].find((candidate) => candidate.type.startsWith('image/'))
  if (!file) return
  event.preventDefault()
  void useAvatarFile(file)
}

function updateEmoji(value: string): void {
  avatarEmoji.value = value || null
  if (value) avatarDataUrl.value = null
  localError.value = ''
}

function submit(): void {
  localError.value = ''
  if (props.mode === 'add' && entryMode.value === 'batch') {
    const nonEmpty = batchRows.value.filter((row) => row.name.trim() || row.rate.trim())
    if (!nonEmpty.length) {
      localError.value = '请至少填写一名工人。'
      return
    }
    const parsed = nonEmpty.map((row) => ({ row, name: row.name.trim(), rate: parseYuanToFen(row.rate) }))
    const invalid = parsed.find((item) => !item.name || item.name.length > 40 || item.rate === null)
    if (invalid) {
      localError.value = `第 ${batchRows.value.indexOf(invalid.row) + 1} 行需要填写姓名和正确的日薪。`
      return
    }
    emit('submit-batch', {
      requestId: ++requestSequence,
      items: parsed.map((item) => ({
        name: item.name,
        defaultDailyRateFen: item.rate!,
        defaultSiteId: defaultSiteId.value,
      })),
    })
    return
  }

  const parsedRate = parseYuanToFen(rate.value)
  const normalizedName = name.value.trim()
  if (!normalizedName || parsedRate === null) {
    localError.value = !normalizedName ? '请输入工人姓名。' : '请输入正确的日薪金额。'
    return
  }
  if (avatarEmoji.value && !isSingleVisibleEmoji(avatarEmoji.value)) {
    localError.value = 'Emoji 头像只能填写一个可见 emoji。'
    return
  }
  emit('submit-single', {
    name: normalizedName,
    avatarDataUrl: avatarDataUrl.value,
    avatarEmoji: avatarEmoji.value,
    defaultDailyRateFen: parsedRate,
    note: note.value.trim(),
    defaultSiteId: defaultSiteId.value,
  })
}
</script>

<template>
  <Transition name="modal">
    <div v-if="open" class="modal-backdrop" @mousedown.self="!submitting && emit('close')">
      <section class="modal" :class="{ 'modal-wide': entryMode === 'batch' }" role="dialog" aria-modal="true" aria-labelledby="worker-dialog-title">
        <div class="modal-header">
          <div><span class="eyebrow">人员资料</span><h2 id="worker-dialog-title">{{ mode === 'add' ? '添加工人' : '编辑工人' }}</h2></div>
          <button class="icon-button" type="button" aria-label="关闭" :disabled="submitting" @click="emit('close')"><UiIcon name="close" /></button>
        </div>
        <form class="modal-form" @submit.prevent="submit">
          <div v-if="mode === 'add'" class="worker-entry-tabs" role="tablist" aria-label="添加方式">
            <button type="button" role="tab" :aria-selected="entryMode === 'single'" :class="{ active: entryMode === 'single' }" @click="setEntryMode('single')">单个添加</button>
            <button type="button" role="tab" :aria-selected="entryMode === 'batch'" :class="{ active: entryMode === 'batch' }" @click="setEntryMode('batch')">批量添加</button>
          </div>

          <template v-if="entryMode === 'single' || mode === 'edit'">
            <label class="form-field"><span>姓名</span><input v-model="name" maxlength="40" autofocus></label>
            <div class="avatar-picker" tabindex="0" aria-label="头像选择区，可按 Ctrl+V 粘贴图片" @paste="onAvatarPaste">
              <div class="avatar-preview"><img v-if="avatarDataUrl" :src="avatarDataUrl" alt="头像预览"><span v-else-if="avatarEmoji" class="avatar-emoji">{{ avatarEmoji }}</span><span v-else>{{ name.slice(0, 1) || '人' }}</span></div>
              <div class="avatar-picker-actions">
                <label class="button button-secondary">选择图片<input type="file" accept="image/*" hidden @change="onAvatarChange"></label>
                <small>也可先点此区域，再按 Ctrl+V 粘贴图片</small>
                <label class="form-field compact-field"><span>或使用一个 Emoji</span><input :value="avatarEmoji ?? ''" maxlength="16" placeholder="例如 👷" @input="updateEmoji(($event.target as HTMLInputElement).value)"></label>
                <button v-if="avatarDataUrl || avatarEmoji" class="link-button remove-avatar-button" type="button" @click="avatarDataUrl = null; avatarEmoji = null">移除头像</button>
              </div>
            </div>
            <label class="form-field"><span>默认日薪</span><div class="money-input"><span>¥</span><input v-model="rate" inputmode="decimal"></div><small>仅作为尚无历史月记录时的起始基线；月度日薪从生效月起沿用，直到再次修改。</small></label>
            <section v-if="mode === 'edit'" class="rate-timeline" aria-labelledby="rate-timeline-title">
              <div><span class="eyebrow">历史口径</span><h3 id="rate-timeline-title">月度日薪变化</h3></div>
              <ol v-if="rateTimeline?.length">
                <li v-for="item in rateTimeline" :key="item.month"><time :datetime="item.month">{{ Number(item.month.slice(0, 4)) }}年{{ Number(item.month.slice(5, 7)) }}月起</time><strong>{{ fenToCurrency(item.rateFen) }} / 日</strong></li>
              </ol>
              <p v-else class="empty-list">还没有单独设置过月度日薪。</p>
              <small>旧数据只保存生效月份，因此这里按月度记录推导，不能还原具体修改日期。</small>
            </section>
            <label class="form-field"><span>默认工地</span><select v-model="defaultSiteId"><option :value="null">未分配</option><option v-for="site in sites" :key="site.id" :value="site.id">{{ site.name }}</option></select></label>
            <label class="form-field"><span>人员备注</span><textarea v-model="note" maxlength="500"></textarea></label>
          </template>

          <template v-else>
            <p class="form-hint">每行填写姓名和日薪；在日薪框按回车会追加并聚焦下一行。</p>
            <div class="batch-worker-table">
              <div class="batch-worker-head"><span>姓名</span><span>日薪（元）</span><span></span></div>
              <div v-for="(row, index) in batchRows" :key="row.id" class="batch-worker-row">
                <input v-model="row.name" :data-batch-worker-id="row.id" maxlength="40" :aria-label="`第 ${index + 1} 行姓名`" placeholder="姓名">
                <div class="money-input"><span>¥</span><input v-model="row.rate" inputmode="decimal" :aria-label="`第 ${index + 1} 行日薪`" placeholder="0.00" @keydown.enter.prevent="appendRow(index)"></div>
                <button class="icon-button" type="button" :aria-label="`删除第 ${index + 1} 行`" @click="removeRow(index)"><UiIcon name="trash" /></button>
              </div>
            </div>
            <button class="button button-secondary" type="button" @click="appendRow()"><UiIcon name="plus" /> 再加一行</button>
            <label class="form-field"><span>这批工人的默认工地</span><select v-model="defaultSiteId"><option :value="null">未分配</option><option v-for="site in sites" :key="site.id" :value="site.id">{{ site.name }}</option></select></label>
            <p v-if="duplicateNames.length" class="warning-text">重名提醒：{{ duplicateNames.join('、') }}。仍可继续添加。</p>
          </template>

          <p v-if="visibleError" class="form-error" role="alert">{{ visibleError }}</p>
          <div class="modal-actions">
            <button v-if="mode === 'edit'" class="button button-danger archive-worker-action" type="button" :disabled="submitting" @click="emit('archive')"><UiIcon name="archive" /> 归档工人</button>
            <span class="modal-actions-spacer"></span>
            <button class="button button-ghost" type="button" :disabled="submitting" @click="emit('close')">取消</button>
            <button class="button button-primary" type="submit" :disabled="submitting">{{ submitting ? '正在保存…' : entryMode === 'batch' ? '一次添加全部' : '保存' }}</button>
          </div>
        </form>
      </section>
    </div>
  </Transition>
</template>
