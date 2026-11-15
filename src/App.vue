
<script setup lang="ts">
import { ref, computed } from 'vue'
import { VueFlow, MarkerType, useVueFlow } from '@vue-flow/core'
import type { Node, Edge } from '@vue-flow/core'

import '@vue-flow/core/dist/style.css'
import '@vue-flow/core/dist/theme-default.css'


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

  return nodeList.find(
    (node) => node.id === selectedNodeId.value
  ) ?? null
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
    description: newDescription
  }
}


const { onConnect, addEdges } = useVueFlow()


onConnect((connection) => {
  if (connection.source === connection.target) {
    return
  }

  const exists = edges.value.some(
    (edge) =>
      edge.source === connection.source &&
      edge.target === connection.target,
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
</script>

  
<template>
  <div class="editor">

    <div class="toolbar">
      <button @click="addNode">
        + Přidat uzel
      </button>
    </div>

    <VueFlow
      v-model:nodes="nodes"
      v-model:edges="edges"
      fit-view-on-init
      @node-click="selectNode"
      @pane-click="selectedNodeId = null"

    />

    <div
    v-if="selectedNode"
    class="properties-panel"
    @keydown.stop
    >
      <button
      class="close-panel"
      @click="selectedNodeId = null"
      title="Zavřít panel"
      >
      ×
      </button>

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
}


.editor {
  position: relative;
}

.toolbar {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
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

</style>
