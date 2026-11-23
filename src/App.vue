<script setup lang="ts">
import { ref, computed } from 'vue'
import { VueFlow, MarkerType, useVueFlow } from '@vue-flow/core'
import type { Node, Edge } from '@vue-flow/core'

import '@vue-flow/core/dist/style.css'
import '@vue-flow/core/dist/theme-default.css'

const attackPathName = ref('Ukázková útočná cesta')

const nodes = ref<Node[]>([
  {
    id: '1',
    type: 'input',
    label: 'Webový server',
    position: { x: 100, y: 150 },
  },
  {
    id: '2',
    type: 'output',
    label: 'Interní server',
    position: { x: 450, y: 150 },
  },
])

const edges = ref<Edge[]>([
  {
    id: 'e1-2',
    source: '1',
    target: '2',
    markerEnd: MarkerType.ArrowClosed,
  },
])

const selectedNodeId = ref<string | null>(null)

type NodeInfo = {
  id: string
  label?: string
  data?: {
    description?: string
  }
}

const selectedNode = computed(() => {
  const nodeList = nodes.value as unknown as NodeInfo[]

  return nodeList.find((node) => node.id === selectedNodeId.value) ?? null
})

function selectNode({ node }: { node: Node }) {
  selectedNodeId.value = node.id
}

function updateNodeLabel(newLabel: string) {
  if (!selectedNode.value) return

  selectedNode.value.label = newLabel
}

function updateNodeDescription(newDescription: string) {
  if (!selectedNode.value) return

  selectedNode.value.data = {
    ...selectedNode.value.data,
    description: newDescription,
  }
}

const { onConnect, addEdges, removeNodes } = useVueFlow()

onConnect((connection) => {
  if (connection.source === connection.target) {
    return
  }

  const exists = edges.value.some(
    (edge) => edge.source === connection.source && edge.target === connection.target,
  )

  if (exists) {
    return
  }

  addEdges([
    {
      ...connection,
      markerEnd: MarkerType.ArrowClosed,
    },
  ])
})

let nextNodeId = 3

function addNode() {
  const id = String(nextNodeId)

  nodes.value.push({
    id: id,
    type: 'default',
    label: `Nový krok ${id}`,
    position: {
      x: 100 + (nextNodeId - 3) * 180,
      y: 350,
    },
  })

  nextNodeId++
}

function deleteSelectedNode() {
  const id = selectedNodeId.value

  if (!id) return

  removeNodes(id)

  selectedNodeId.value = null
}

function exportGraph() {
  const enteredName = window.prompt('Zadej název útočné cesty:', attackPathName.value)

  if (enteredName === null) return

  const graphName = enteredName
    .trim()
    .replace(/\.json$/i, '')
    .trim()

  const fileName = graphName.replace(/[\\/:*?"<>|]/g, '_').replace(/[. ]+$/g, '')

  if (!fileName) {
    window.alert('Zadej platný název.')
    return
  }

  attackPathName.value = graphName

  type ExportNode = {
    id: string
    type?: string
    label?: string
    position: { x: number; y: number }
    data?: { description?: string }
  }

  type ExportEdge = {
    id: string
    source: string
    target: string
  }

  const exportNodes = nodes.value as unknown as ExportNode[]
  const exportEdges = edges.value as unknown as ExportEdge[]

  const graph = {
    schemaVersion: 1,
    id: 'attack-path-1',
    name: attackPathName.value,

    steps: exportNodes.map((node) => ({
      id: node.id,
      name: node.label ?? '',
      description: node.data?.description ?? '',
      position: node.position,
      nodeType: node.type ?? 'default',
    })),

    edges: exportEdges.map((edge) => ({
      id: edge.id,
      sourceStepId: edge.source,
      targetStepId: edge.target,
    })),

    systems: [],
    vulnerabilities: [],
    findings: [],
  }

  const json = JSON.stringify(graph, null, 2)

  const blob = new Blob([json], {
    type: 'application/json',
  })

  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')

  link.href = url
  link.download = `${fileName}.json`

  document.body.appendChild(link)
  link.click()
  link.remove()

  URL.revokeObjectURL(url)
}

type ImportStep = {
  id: string
  name: string
  description: string
  position: { x: number; y: number }
  nodeType: 'input' | 'output' | 'default'
}

type ImportEdge = {
  id: string
  sourceStepId: string
  targetStepId: string
}

function isObject(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null && !Array.isArray(value)
}

function isImportStep(value: unknown): value is ImportStep {
  if (!isObject(value) || !isObject(value.position)) return false

  return (
    typeof value.id === 'string' &&
    value.id.length > 0 &&
    typeof value.name === 'string' &&
    typeof value.description === 'string' &&
    typeof value.position.x === 'number' &&
    Number.isFinite(value.position.x) &&
    typeof value.position.y === 'number' &&
    Number.isFinite(value.position.y) &&
    ['input', 'output', 'default'].includes(String(value.nodeType))
  )
}

function isImportEdge(value: unknown): value is ImportEdge {
  if (!isObject(value)) return false

  return (
    typeof value.id === 'string' &&
    value.id.length > 0 &&
    typeof value.sourceStepId === 'string' &&
    typeof value.targetStepId === 'string'
  )
}

async function importGraph(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]

  if (!file) return

  try {
    const parsed: unknown = JSON.parse(await file.text())

    if (
      !isObject(parsed) ||
      parsed.schemaVersion !== 1 ||
      typeof parsed.id !== 'string' ||
      typeof parsed.name !== 'string' ||
      !Array.isArray(parsed.steps) ||
      !Array.isArray(parsed.edges)
    ) {
      throw new Error('Soubor nemá správnou strukturu grafu.')
    }

    const steps: unknown[] = parsed.steps
    const connections: unknown[] = parsed.edges

    if (!steps.every(isImportStep) || !connections.every(isImportEdge)) {
      throw new Error('Soubor obsahuje neplatné uzly nebo propojení.')
    }

    if (
      !Array.isArray(parsed.systems) ||
      !Array.isArray(parsed.vulnerabilities) ||
      !Array.isArray(parsed.findings) ||
      parsed.systems.length > 0 ||
      parsed.vulnerabilities.length > 0 ||
      parsed.findings.length > 0
    ) {
      throw new Error('Soubor obsahuje nepodporovaná bezpečnostní data.')
    }

    const stepIds = new Set(steps.map((step) => step.id))
    const edgeIds = new Set(connections.map((edge) => edge.id))
    const edgePairs = new Set(
      connections.map((edge) => JSON.stringify([edge.sourceStepId, edge.targetStepId])),
    )

    if (
      stepIds.size !== steps.length ||
      edgeIds.size !== connections.length ||
      edgePairs.size !== connections.length ||
      connections.some(
        (edge) =>
          edge.sourceStepId === edge.targetStepId ||
          !stepIds.has(edge.sourceStepId) ||
          !stepIds.has(edge.targetStepId),
      )
    ) {
      throw new Error('Graf obsahuje duplicitní nebo neplatná propojení či ID.')
    }

    const importedNodes = steps.map((step) => ({
      id: step.id,
      type: step.nodeType,
      label: step.name,
      position: { ...step.position },
      data: { description: step.description },
    }))

    const importedEdges = connections.map((edge) => ({
      id: edge.id,
      source: edge.sourceStepId,
      target: edge.targetStepId,
      markerEnd: MarkerType.ArrowClosed,
    }))

    if (!window.confirm('Načtení souboru nahradí současný graf. Pokračovat?')) {
      return
    }

    nodes.value = importedNodes as unknown as Node[]
    edges.value = importedEdges as unknown as Edge[]
    attackPathName.value = parsed.name as string

    selectedNodeId.value = null

    nextNodeId = Math.max(
      3,
      ...steps.map((step) => (/^\d+$/.test(step.id) ? Number(step.id) + 1 : 3)),
    )

    window.alert('Graf byl úspěšně načten.')
  } catch (error) {
    window.alert(error instanceof Error ? error.message : 'Soubor se nepodařilo načíst.')
  } finally {
    input.value = ''
  }
}
</script>

<template>
  <div class="editor">
    <div class="toolbar">
      <button @click="addNode">+ Přidat uzel</button>
      <button @click="exportGraph">Exportovat JSON</button>

      <label class="import-button">
        Importovat JSON
        <input type="file" accept=".json,application/json" @change="importGraph" />
      </label>
    </div>

    <VueFlow
      v-model:nodes="nodes"
      v-model:edges="edges"
      fit-view-on-init
      @node-click="selectNode"
      @pane-click="selectedNodeId = null"
      :delete-key-code="['Backspace', 'Delete']"
    />

    <div v-if="selectedNode" class="properties-panel" @keydown.stop>
      <button class="close-panel" @click="selectedNodeId = null" title="Zavřít panel">×</button>

      <h3>Vlastnosti uzlu</h3>

      <p><strong>ID:</strong> {{ selectedNode.id }}</p>

      <label for="node-label">Název uzlu:</label>

      <input
        id="node-label"
        type="text"
        :value="selectedNode.label ?? ''"
        @input="updateNodeLabel(($event.target as HTMLInputElement).value)"
      />

      <label for="node-description">Popis uzlu:</label>

      <textarea
        id="node-description"
        :value="selectedNode.data?.description ?? ''"
        @input="updateNodeDescription(($event.target as HTMLTextAreaElement).value)"
        rows="5"
        placeholder="Zadejte popis kroku útoku..."
      ></textarea>

      <button type="button" class="delete-node-button" @click="deleteSelectedNode">
        Odstranit uzel
      </button>
    </div>
  </div>
</template>

<style>
html,
body,
#app {
  margin: 0;
  width: 100%;
  height: 100%;
}

.editor {
  width: 100%;
  height: 100vh;
  position: relative;
}

.toolbar {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;

  display: flex;
  align-items: center;
  gap: 10px;
}

.toolbar button {
  padding: 10px 18px;
  background-color: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
}

.toolbar button:hover {
  background-color: #1d4ed8;
}

.properties-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 10;

  width: 260px;
  padding: 16px;

  background: white;
  border: 1px solid #ccc;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
}

.properties-panel input {
  width: 100%;
  box-sizing: border-box;
  padding: 8px;
  margin-top: 8px;

  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 14px;
}

.close-panel {
  position: absolute;
  top: 10px;
  right: 10px;

  border: none;
  background: transparent;
  font-size: 24px;
  cursor: pointer;
  color: #666;
}

.close-panel:hover {
  color: #000;
}

.vue-flow__node-default,
.vue-flow__node-input,
.vue-flow__node-output {
  width: 200px;
  max-width: 200px;
  box-sizing: border-box;

  white-space: normal;
  overflow-wrap: anywhere;
}

.properties-panel textarea {
  width: 100%;
  box-sizing: border-box;
  padding: 8px;
  margin-top: 8px;
  margin-bottom: 12px;

  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 14px;

  resize: vertical;
}

.delete-node-button {
  width: 100%;
  padding: 10px;
  margin-top: 12px;

  background: #dc3545;
  color: white;

  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.delete-node-button:hover {
  background: #b02a37;
}

.import-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  position: relative;

  background: #2563eb;
  color: white;
  padding: 10px 18px;
  border-radius: 6px;

  font-size: 14px;
  cursor: pointer;
}

.import-button:hover {
  background: #1d4ed8;
}

.import-button input {
  position: absolute;
  width: 1px;
  height: 1px;
  opacity: 0;
}

.import-button:focus-within {
  outline: 2px solid #1d4ed8;
  outline-offset: 2px;
}
</style>
