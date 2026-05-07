<script setup lang="ts">
import { ref, computed, nextTick, watch } from 'vue'
import { useData } from 'vitepress'
import toolsData from '../../data/mcp-tools.json'
import {
  Accordion,
  AccordionItem,
  AccordionTrigger,
  AccordionContent,
} from './ui/accordion'

// ── Type definitions ──────────────────────────────────────────────────────────

interface SchemaProperty {
  type: string | string[]
  description?: string
  enum?: string[]
  default?: unknown
  format?: string
  minimum?: number
  maximum?: number
}

interface ToolSchema {
  properties?: Record<string, SchemaProperty>
  required?: string[]
}

interface Tool {
  name: string
  title: string
  description: string
  inputSchema?: ToolSchema
  annotations?: Record<string, unknown>
}

interface ScopeDef {
  id: string
  name: string
  description: string
  tools: string[]
}

interface LocaleTool {
  title?: string
  description?: string
  properties?: Record<string, string>
}

interface LocaleData {
  scopes?: Record<string, { name: string; description: string }>
  tools?: Record<string, LocaleTool>
}

// ── Data ─────────────────────────────────────────────────────────────────────

const allTools: Tool[] = (toolsData as any).tools
const scopeDefs: ScopeDef[] = (toolsData as any).scopes ?? []
const locales: Record<string, LocaleData> = (toolsData as any).locales ?? {}

// tool name → scope def (for quick lookup)
const toolScopeMap = new Map<string, ScopeDef>()
for (const scope of scopeDefs) {
  for (const toolName of scope.tools) {
    toolScopeMap.set(toolName, scope)
  }
}

// Stable color index per scope (based on position in array)
const scopeColors = ['brand', 'green', 'yellow', 'red', 'indigo', 'purple']
const scopeColorMap = new Map<string, string>()
scopeDefs.forEach((s, i) => {
  scopeColorMap.set(s.name, scopeColors[i % scopeColors.length])
})

// ── Locale helpers ────────────────────────────────────────────────────────────

const { lang } = useData()

const i18n = computed(() => {
  switch (lang.value) {
    case 'zh-CN':
      return {
        search: '搜索工具名称或描述...',
        parameters: '参数',
        required: '必填',
        noParams: '无参数',
        noMatch: (q: string) => `没有匹配 "${q}" 的工具`,
      }
    case 'zh-HK':
      return {
        search: '搜尋工具名稱或描述...',
        parameters: '參數',
        required: '必填',
        noParams: '無參數',
        noMatch: (q: string) => `沒有匹配 "${q}" 的工具`,
      }
    default:
      return {
        search: 'Search tools by name or description...',
        parameters: 'Parameters',
        required: 'Required',
        noParams: 'No parameters',
        noMatch: (q: string) => `No tools match "${q}"`,
      }
  }
})

function getLocaleTool(toolName: string): LocaleTool | undefined {
  return locales[lang.value]?.tools?.[toolName]
}

function getTitle(tool: Tool): string {
  return getLocaleTool(tool.name)?.title ?? tool.title ?? tool.name
}

function getDescription(tool: Tool): string {
  return getLocaleTool(tool.name)?.description ?? tool.description
}

function getScopeLabel(tool: Tool): { label: string; color: string } | null {
  const scope = toolScopeMap.get(tool.name)
  if (!scope) return null
  const localeScope = locales[lang.value]?.scopes?.[scope.name]
  const label = localeScope?.name ?? scope.name
  const color = scopeColorMap.get(scope.name) ?? 'brand'
  return { label, color }
}

// ── Params ────────────────────────────────────────────────────────────────────

interface ParamRow {
  name: string
  type: string
  required: boolean
  description?: string
  enum?: string[]
  default?: unknown
}

function formatType(type: string | string[]): string {
  return Array.isArray(type) ? type.join(' | ') : type
}

function getParams(tool: Tool): ParamRow[] {
  const schema = tool.inputSchema
  if (!schema?.properties) return []
  const required = new Set(schema.required ?? [])
  const localeProps = getLocaleTool(tool.name)?.properties ?? {}
  return Object.entries(schema.properties).map(([name, def]) => ({
    name,
    type: formatType(def.type),
    required: required.has(name),
    description: localeProps[name] ?? def.description,
    enum: def.enum,
    default: def.default,
  }))
}

// ── Search & accordion ────────────────────────────────────────────────────────

const query = ref('')
const openValue = ref<string>('')
const listRef = ref<HTMLElement | null>(null)

watch(openValue, async (v) => {
  if (!v || !listRef.value) return
  await nextTick()
  await new Promise((resolve) => setTimeout(resolve, 220))
  const container = listRef.value
  if (!container) return
  const openItem = container.querySelector<HTMLElement>(
    '.mcp-accordion-item[data-state="open"]'
  )
  if (!openItem) return
  const itemTop =
    openItem.getBoundingClientRect().top -
    container.getBoundingClientRect().top +
    container.scrollTop
  const itemBottom = itemTop + openItem.offsetHeight
  const visibleTop = container.scrollTop
  const visibleBottom = visibleTop + container.clientHeight
  if (itemTop >= visibleTop && itemBottom <= visibleBottom) return
  container.scrollTo({ top: itemTop, behavior: 'smooth' })
})

const filtered = computed<Tool[]>(() => {
  const q = query.value.trim().toLowerCase()
  if (!q) return allTools
  return allTools.filter((t) => {
    const title = getTitle(t).toLowerCase()
    const desc = getDescription(t).toLowerCase()
    return t.name.toLowerCase().includes(q) || title.includes(q) || desc.includes(q)
  })
})
</script>

<template>
  <div class="mcp-tools">
    <div class="mcp-tools-search">
      <div class="mcp-tools-input-wrap">
        <svg
          class="mcp-tools-search-icon"
          xmlns="http://www.w3.org/2000/svg"
          width="16"
          height="16"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          aria-hidden="true"
        >
          <circle cx="11" cy="11" r="8" />
          <line x1="21" y1="21" x2="16.65" y2="16.65" />
        </svg>
        <input
          v-model="query"
          type="text"
          class="mcp-tools-input"
          :placeholder="i18n.search"
        />
        <button
          v-if="query"
          type="button"
          class="mcp-tools-clear"
          aria-label="Clear search"
          @click="query = ''"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="14"
            height="14"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <line x1="18" y1="6" x2="6" y2="18" />
            <line x1="6" y1="6" x2="18" y2="18" />
          </svg>
        </button>
      </div>
    </div>

    <div v-if="filtered.length === 0" class="mcp-tools-empty">
      {{ i18n.noMatch(query) }}
    </div>

    <div v-else ref="listRef" class="mcp-tools-list">
      <Accordion v-model="openValue" type="single" collapsible>
        <AccordionItem
          v-for="tool in filtered"
          :key="tool.name"
          :value="tool.name"
        >
          <AccordionTrigger>
            <span class="mcp-tool-trigger">
              <svg
                class="mcp-tool-icon"
                xmlns="http://www.w3.org/2000/svg"
                width="14"
                height="14"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
                aria-hidden="true"
              >
                <path d="M14.7 6.3a1 1 0 0 0 0 1.4l1.6 1.6a1 1 0 0 0 1.4 0l3.77-3.77a6 6 0 0 1-7.94 7.94l-6.91 6.91a2.12 2.12 0 0 1-3-3l6.91-6.91a6 6 0 0 1 7.94-7.94l-3.76 3.76z" />
              </svg>
              <span class="mcp-tool-meta">
                <span class="mcp-tool-title">{{ getTitle(tool) }}</span>
                <span v-if="getScopeLabel(tool)" class="mcp-scope-label">
                  {{ getScopeLabel(tool)!.label }}
                </span>
              </span>
            </span>
          </AccordionTrigger>
          <AccordionContent>
            <p class="mcp-tool-desc">{{ getDescription(tool) }}</p>

            <p v-if="getParams(tool).length === 0" class="mcp-tool-no-params">
              {{ i18n.noParams }}
            </p>

            <template v-else>
              <h4 class="mcp-params-title">{{ i18n.parameters }}</h4>
              <dl class="mcp-params">
                <div
                  v-for="p in getParams(tool)"
                  :key="p.name"
                  class="mcp-param"
                >
                  <dt class="mcp-param-head">
                    <code class="mcp-param-name">{{ p.name }}</code>
                    <span class="mcp-param-type">{{ p.type }}</span>
                    <span v-if="p.required" class="mcp-param-required">{{ i18n.required }}</span>
                  </dt>
                  <dd class="mcp-param-body">
                    <p v-if="p.description" class="mcp-param-desc">{{ p.description }}</p>
                    <p v-if="p.enum" class="mcp-param-meta">
                      Enum: {{ p.enum.map((e) => `"${e}"`).join(' | ') }}
                    </p>
                    <p v-if="p.default !== undefined" class="mcp-param-meta">
                      Default: {{ p.default }}
                    </p>
                  </dd>
                </div>
              </dl>
            </template>
          </AccordionContent>
        </AccordionItem>
      </Accordion>
    </div>
  </div>
</template>

<style scoped>
.mcp-tools {
  margin: 1rem 0;
}

.mcp-tools-search {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
  padding-bottom: 0.5rem;
  position: sticky;
  top: var(--vp-nav-height, 64px);
  z-index: 1;
  background: var(--vp-c-bg);
}

.mcp-tools-input-wrap {
  width: 320px;
  max-width: 100%;
  position: relative;
  display: flex;
  align-items: center;
}

.mcp-tools-search-icon {
  position: absolute;
  left: 0.75rem;
  color: var(--vp-c-text-3);
  pointer-events: none;
}

.mcp-tools-input {
  width: 100%;
  padding: 0.55rem 2.25rem 0.55rem 2.25rem;
  border: 1px solid var(--vp-c-divider);
  border-radius: 8px;
  background: transparent;
  color: var(--vp-c-text-1);
  font-size: 0.9rem;
  line-height: 1.25;
  outline: none;
}

.mcp-tools-input::placeholder {
  color: var(--vp-c-text-3);
}

.mcp-tools-clear {
  position: absolute;
  right: 0.5rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 1.5rem;
  height: 1.5rem;
  padding: 0;
  border: 0;
  border-radius: 4px;
  background: transparent;
  color: var(--vp-c-text-3);
  cursor: pointer;
  transition: color 0.15s ease, background 0.15s ease;
}

.mcp-tools-clear:hover {
  color: var(--vp-c-text-1);
  background: var(--vp-c-default-soft);
}

.mcp-tools-empty {
  padding: 1rem;
  text-align: center;
  color: var(--vp-c-text-2);
  font-size: 0.9rem;
}

.mcp-tools-list {
  border: 1px solid var(--vp-c-divider);
  border-radius: 6px;
  max-height: 500px;
  overflow-y: auto;
  padding: 0 0.75rem;
}

.mcp-tools-list :deep(.mcp-accordion-item:last-child) {
  border-bottom: 0;
}

.mcp-tool-trigger {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  flex: 1;
  min-width: 0;
}

.mcp-tool-icon {
  color: var(--vp-c-text-3);
  flex-shrink: 0;
}

.mcp-tools-list :deep(.mcp-accordion-trigger[data-state='open']) .mcp-tool-icon {
  color: var(--vp-c-brand-1);
}

.mcp-tool-meta {
  display: flex;
  flex-direction: row;
  align-items: baseline;
  flex: 1;
  min-width: 0;
}

.mcp-tool-title {
  font-size: 0.9rem;
  font-weight: 500;
  color: var(--vp-c-text-1);
  line-height: 1.3;
}

.mcp-tool-name {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, monospace;
  font-size: 0.72rem;
  color: var(--vp-c-text-3);
  background: transparent;
  padding: 0;
}

.mcp-tools-list :deep(.mcp-accordion-trigger) {
  padding: 0.55rem 0;
}

/* Scope label */
.mcp-scope-label {
  margin-left: auto;
  padding-right: 0.5rem;
  font-size: 0.65rem;
  line-height: 1;
  color: var(--vp-c-text-3);
  white-space: nowrap;
}


/* Content */
.mcp-tool-desc {
  margin: 0 0 0.5rem;
  color: var(--vp-c-text-2);
  font-size: 0.75rem;
}

.mcp-tool-no-params {
  margin: 0;
  color: var(--vp-c-text-3);
  font-style: italic;
  font-size: 0.75rem;
}

.mcp-params-title {
  margin: 0 0 0.35rem;
  font-size: 0.75rem;
  font-weight: 400;
  color: var(--vp-c-text-2);
}

.mcp-params {
  margin: 0;
  padding: 0;
}

.mcp-param {
  padding: 0.3rem 0;
}
.mcp-param:first-child { padding-top: 0; }
.mcp-param:last-child  { padding-bottom: 0; }

.mcp-param-head {
  display: flex;
  flex-wrap: wrap;
  align-items: baseline;
  gap: 0.4rem;
  margin: 0 0 0.2rem;
}

.mcp-param-name {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, monospace;
  font-size: 0.75rem;
  color: var(--vp-c-text-1);
  background: var(--vp-c-bg-soft);
  padding: 0.05rem 0.35rem;
  border-radius: 4px;
}

.mcp-param-type {
  font-size: 0.75rem;
  color: var(--vp-c-text-2);
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, monospace;
}

.mcp-param-required {
  font-size: 0.75rem;
  color: var(--vp-c-text-2);
  background: var(--vp-c-bg-soft);
  padding: 0.08rem 0.35rem;
  border-radius: 4px;
}

.mcp-param-body {
  margin: 0;
  padding-left: 0;
}

.mcp-param-desc {
  margin: 0;
  color: var(--vp-c-text-2);
  font-size: 0.75rem;
  line-height: 1.45;
}

.mcp-param-meta {
  margin: 0;
  color: var(--vp-c-text-3);
  font-size: 0.75rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, monospace;
}
</style>
