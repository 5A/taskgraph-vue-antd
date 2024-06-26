<template>
  <a-layout-content
    :style="{
      background: '#fff',
      padding: '24px',
      margin: '24px 0 24px 0',
      minHeight: '280px'
    }"
  >
    <a-flex gap="small">
      <a-card title="DAG View" style="width: 70%">
        <template #extra>
          <a-flex gap="small">
            <a-select
              ref="select"
              v-model:value="projectOperationInputStore.projectDagViewInputState.layout"
              style="width: 120px"
              :options="cytoscapeLayoutOptions"
              @change="handleCytoscapeLayoutSelectChange"
            ></a-select>
            <a-button @click="onResetDAGView"> Reset View </a-button>
          </a-flex>
        </template>
        <div class="cytoscape-container" ref="cytoscapeContainer"></div>
      </a-card>
      <a-card title="Task Toolbox" style="width: 30%">
        <a-form :model="projectInputState" layout="vertical">
          <a-form-item label="Name">
            <a-input
              v-model:value="projectInputState.add_new_task_name"
              style="width: 100%"
              placeholder="Task Name"
            />
          </a-form-item>
          <a-form-item label="Detail">
            <a-textarea
              v-model:value="projectInputState.add_new_task_detail"
              placeholder="Description of task"
              :rows="4"
            />
          </a-form-item>
          <a-form-item label="Relationship to Selected Task">
            <a-radio-group
              v-model:value="projectInputState.task_toolbox_location_select"
              style="width: 100%"
            >
              <a-radio-button value="sub">Sub-task</a-radio-button>
              <a-radio-button value="super">Super-task</a-radio-button>
              <a-radio-button value="self">Self</a-radio-button>
            </a-radio-group>
          </a-form-item>
          <a-form-item>
            <a-flex gap="small" wrap="wrap">
              <a-button type="primary" @click="onAddOrModifyTask">
                <PlusCircleOutlined /> Add/Modify Task
              </a-button>
              <a-button danger @click="onClearNewTaskInput"> Clear </a-button>
            </a-flex>
          </a-form-item>
          <a-divider />
          <a-form-item label="Depends On">
            <a-select
              v-model:value="projectInputState.add_dependency_selected_nodes"
              mode="tags"
              style="width: 100%"
              placeholder="Select Dependency"
              optionFilterProp="label"
              :options="selectOptionsOfTasks"
            ></a-select>
          </a-form-item>
          <a-form-item>
            <a-flex gap="small" wrap="wrap">
              <a-button type="primary" @click="onAddDependencies">
                <PlusCircleOutlined /> Add Dependencies
              </a-button>
              <a-button danger @click="onRemoveDependency"> Remove Selected </a-button>
            </a-flex>
          </a-form-item>
        </a-form>
      </a-card>
    </a-flex>
    <br />
    <a-card :title="`Task: ${selectedTaskName}`" style="width: 100%">
      <template #extra>
        <a-flex gap="small">
          <a-input
            v-if="isEditingTaskInDetailView"
            key="1"
            v-model:value="projectInputState.edit_task_name"
            style="width: 100%"
            placeholder="New Title"
          />
          <a-button v-if="!isEditingTaskInDetailView" key="2" @click="onEditTaskInDetailView">
            <EditOutlined />
            Edit
          </a-button>
          <a-button
            v-if="isEditingTaskInDetailView"
            key="3"
            @click="onSaveTaskInDetailView"
            type="primary"
          >
            <SaveOutlined />
            Save
          </a-button>
          <a-button key="4" @click="onSetTaskDone" type="primary">
            <CheckCircleOutlined />
            Done
          </a-button>
          <a-button key="5" @click="showSnoozeTaskModal" type="primary">
            <PauseCircleOutlined />
            Snooze
          </a-button>
          <a-modal
            v-model:open="snoozeTaskModalOpen"
            :title="`Snooze: ${selectedTaskName}`"
            @ok="handleSnoozeTaskModalOk"
          >
            <br />
            <a-form :model="projectInputState" layout="vertical">
              <a-form-item label="Snooze Until:">
                <a-date-picker
                  show-time
                  placeholder="Select Time"
                  v-model:value="projectInputState.snooze_task_until"
                  style="width: 100%"
                />
              </a-form-item>
              <a-form-item label="Reason:">
                <a-input
                  v-model:value="projectInputState.snooze_task_reason"
                  style="width: 100%"
                  placeholder="Reason"
                />
              </a-form-item>
            </a-form>
          </a-modal>
          <a-button key="6" type="primary" danger @click="onRemoveTask">
            <DeleteOutlined />
            Remove
          </a-button>
        </a-flex>
      </template>
      <template v-if="selectedTaskStatus == 'Snoozed'">
        <a-alert :message="selectedTaskSnoozeFormatted" type="success" />
        <br />
      </template>
      <a-textarea
        v-if="isEditingTaskInDetailView"
        v-model:value="projectInputState.edit_task_detail"
        style="width: 100%"
        placeholder="Detail"
        :rows="20"
      />
      <p v-else style="white-space: pre-wrap">{{ selectedTaskDetail }}</p>
    </a-card>
    <br />
    <a-card title="Issues" style="width: 100%">
      <template #extra>
        <a-flex gap="small">
          <a-button key="1" type="primary" @click="showNewIssueModal">
            <PlusCircleOutlined />
            Open New Issue
          </a-button>
          <a-modal
            v-model:open="newIssueModalOpen"
            :title="`New Issue: ${selectedTaskName}`"
            @ok="handleNewIssueModalOk"
          >
            <a-flex gap="small" vertical>
              <a-input
                v-model:value="projectInputState.new_issue_title"
                style="width: 100%"
                placeholder="Title"
              />
              <a-textarea
                v-model:value="projectInputState.new_issue_description"
                style="width: 100%"
                placeholder="Description"
                :rows="4"
              />
            </a-flex>
          </a-modal>
        </a-flex>
      </template>
      <a-table :columns="issue_columns" :data-source="issue_data_source.open" bordered>
        <template #bodyCell="{ column, record }">
          <template v-if="column.dataIndex === 'operation'">
            <a-button key="1" type="primary" @click="onCloseIssue(record.key)">
              <template #icon>
                <CheckOutlined />
              </template>
              Close
            </a-button>
          </template>
        </template>
        <template #footer>
          <a-flex gap="small" horizontal>
            <a-input
              v-model:value="projectInputState.new_issue_title"
              style="width: 30%"
              placeholder="Title (Enter to open)"
              @pressEnter="onOpenNewIssue"
            />
            <a-textarea
              v-model:value="projectInputState.new_issue_description"
              placeholder="Description"
              :rows="1"
            />
            <a-button key="1" type="primary" @click="onOpenNewIssue">
              <PlusCircleOutlined />
              Quick Open
            </a-button>
          </a-flex>
        </template>
      </a-table>
    </a-card>
    <br />
    <a-card title="Closed Issues" style="width: 100%">
      <a-table :columns="issue_columns" :data-source="issue_data_source.closed" bordered>
        <template #bodyCell="{ column, record }">
          <template v-if="column.dataIndex === 'operation'">
            <a-button key="1" type="primary" danger @click="onDeleteIssue(record.key)">
              <template #icon>
                <DeleteOutlined />
              </template>
              Delete
            </a-button>
          </template>
        </template>
      </a-table>
    </a-card>
  </a-layout-content>
  <a-page-header
    class="project-page-header"
    style="border: 1px solid rgb(235, 237, 240)"
    :title="projectName"
    :sub-title="projectUUID"
  >
    <template #extra>
      <a-button key="1" type="primary" @click="onSyncStatus">
        <SyncOutlined />
        Sync
      </a-button>
      <a-button key="2" type="primary" danger @click="showDeleteProjectModal">
        <DeleteOutlined />
        Delete
      </a-button>
    </template>
    <a-descriptions size="small" :column="{ xxl: 8, xl: 6, lg: 6, md: 4, sm: 2, xs: 2 }">
      <a-descriptions-item label="Total Tasks"> {{ numberOfTasks }} </a-descriptions-item>
      <a-descriptions-item label="Done"> {{ projectTaskStatistics.done }} </a-descriptions-item>
      <a-descriptions-item label="Active"> {{ projectTaskStatistics.active }} </a-descriptions-item>
      <a-descriptions-item label="Pending">
        {{ projectTaskStatistics.pending }}
      </a-descriptions-item>
      <a-descriptions-item label="Snoozed">
        {{ projectTaskStatistics.snoozed }}
      </a-descriptions-item>
      <a-descriptions-item label="Unknown Status">
        {{ projectTaskStatistics.unknown }}
      </a-descriptions-item>
      <a-descriptions-item label="Stated Tasks">
        {{ projectTaskStatistics.total }}
      </a-descriptions-item>
    </a-descriptions>
    <a-modal
      v-model:open="deleteProjectModalOpen"
      title="Delete This Project?"
      @ok="handleDeleteProjectModalOk"
    >
      <p>Name: {{ projectName }}</p>
      <p>UUID: {{ projectUUID }}</p>
    </a-modal>
  </a-page-header>
</template>

<script lang="ts" setup>
// Vue and vue ecosystem
import { computed, ref, watch, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
// Theme
import { message } from 'ant-design-vue'
import type { SelectProps } from 'ant-design-vue'
import {
  CheckCircleOutlined,
  CheckOutlined,
  EditOutlined,
  SaveOutlined,
  DeleteOutlined,
  SyncOutlined,
  PlusCircleOutlined,
  PauseCircleOutlined
} from '@ant-design/icons-vue'
// Graph plotting
import cytoscape from 'cytoscape'
import dagre from 'cytoscape-dagre'
// This project
import {
  useProjectListStore,
  useTaskGraphDataStore,
  useProjectOperationInputStore
} from '@/store/projects'
import { callRESTfulAPI } from '@/common/connection'
import { type TaskGraphProjectData } from '@/store/projects'

cytoscape.use(dagre)

const route = useRoute()
const router = useRouter()
const projectListStore = useProjectListStore()
const taskgraphDataStore = useTaskGraphDataStore()
const projectOperationInputStore = useProjectOperationInputStore()

async function readProject(project_uuid: string) {
  // get project data for given uuid
  await callRESTfulAPI(`projects/${project_uuid}`, 'GET').then((response) => {
    if (response?.name) {
      taskgraphDataStore.taskgraphData.projects[project_uuid] = response
    }
  })
}

const projectUUID = ref<string>()
if (typeof route.params.project_uuid === 'string') {
  projectUUID.value = route.params.project_uuid
}

// ======== Aliases for Convenience ===========
const currentProject = computed(() =>
  projectUUID.value ? taskgraphDataStore.taskgraphData.projects[projectUUID.value] : null
)
const projectName = computed(() =>
  currentProject.value ? currentProject.value.name : 'Loading...'
)
const listOfTasks = computed(() =>
  currentProject.value ? currentProject.value.DAG.nodes.map((item) => item.id) : null
)
const numberOfTasks = computed(() => (listOfTasks.value ? listOfTasks.value.length : null))
const selectOptionsOfTasks = computed(() =>
  listOfTasks.value
    ? listOfTasks.value.map((item) => ({
        value: item,
        label: currentProject.value?.metadata[item]['name']
      }))
    : null
)
const projectTaskStatistics = computed(() => {
  let n_tasks_done = 0
  let n_tasks_active = 0
  let n_tasks_pending = 0
  let n_tasks_snoozed = 0
  let n_tasks_unknown_status = 0
  if (currentProject.value && currentProject.value.metadata) {
    for (const item in currentProject.value.metadata) {
      const meta = currentProject.value.metadata[item]
      if (meta.status) {
        const task_status = meta.status
        if (task_status === 'Done') {
          n_tasks_done++
        } else if (task_status === 'Active') {
          n_tasks_active++
        } else if (task_status === 'Pending') {
          n_tasks_pending++
        } else if (task_status === 'Snoozed') {
          n_tasks_snoozed++
        } else {
          n_tasks_unknown_status++
        }
      }
    }
  }
  return {
    done: n_tasks_done,
    active: n_tasks_active,
    pending: n_tasks_pending,
    snoozed: n_tasks_snoozed,
    unknown: n_tasks_unknown_status,
    total: n_tasks_done + n_tasks_active + n_tasks_pending + n_tasks_snoozed
  }
})

// ======== React to router parameter changes when loading project page
onMounted(() => {
  if (typeof route.params.project_uuid === 'string') {
    projectUUID.value = route.params.project_uuid
    readProject(projectUUID.value).then(() => initCytoscape())
    projectInputState.selected_node = null
    projectInputState.add_dependency_selected_nodes = []
  }
})

watch(
  () => route.params.project_uuid,
  (newId, _oldId) => {
    if (typeof newId === 'string') {
      projectUUID.value = newId
      readProject(projectUUID.value).then(() => initCytoscape())
      projectInputState.selected_node = null
      projectInputState.add_dependency_selected_nodes = []
    }
  }
)

// ========== Project Management Header

const deleteProjectModalOpen = ref<boolean>(false)

const showDeleteProjectModal = () => {
  deleteProjectModalOpen.value = true
}

const handleDeleteProjectModalOk = (_e: MouseEvent) => {
  deleteProject()
  deleteProjectModalOpen.value = false
}

async function deleteProject() {
  // Delete project by DELETE
  await callRESTfulAPI(
    'projects',
    'DELETE',
    JSON.stringify({
      uuid: projectUUID.value
    })
  ).then((response) => {
    if (response?.id) {
      console.log('Project Deleted')
    }
  })
  // Purge records locally, resync with remote
  // retrive a list of projects by GET
  await callRESTfulAPI('projects', 'GET').then((response) => {
    if (response?.projects) {
      projectListStore.projectListState.projects = {}
      for (const item in response.projects) {
        projectListStore.projectListState.projects[item] = response.projects[item]
      }
    }
  })
  // Navigate user to landing page
  router.push({ path: '/projects/manage' })
}

async function onSyncStatus() {
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

// ========== Cytoscape Render ========
const cytoscapeLayoutOptions = ref<SelectProps['options']>([
  {
    value: 'dagre',
    label: 'Dagre'
  },
  {
    value: 'breadthfirst',
    label: 'Breadth First'
  },
  {
    value: 'cose',
    label: 'CoSE'
  },
  {
    value: 'circle',
    label: 'Circle'
  },
  {
    value: 'concentric',
    label: 'Concentric'
  },
  {
    value: 'random',
    label: 'Random'
  }
])

// construct cytoscape data from taskgraph data, if got null/undefined, empty data is generated.
function construct_cytoscape_data(tgdata: TaskGraphProjectData | null) {
  let nodes = []
  let edges = []
  if (tgdata) {
    for (const item of tgdata.DAG.nodes) {
      let data: Record<string, string | number> = { id: item.id }
      if (item.id in tgdata.metadata) {
        // task exists
        const task = tgdata.metadata[item.id]
        if (task.name) {
          data['name'] = task.name
        }
        if (task.status) {
          data['status'] = task.status
        }
      }
      nodes.push({
        data: data
      })
    }
    for (const item of tgdata.DAG.links) {
      edges.push({
        data: { id: item.id, source: item.source, target: item.target }
      })
    }
  }
  return {
    nodes: nodes,
    edges: edges
  }
}

function get_status_color(ele: any): any {
  const status: string = ele.data('status')
  if (status === 'Active') {
    return '#5bc0de' // light blue
  } else if (status === 'Pending') {
    return '#FFFFFF' // white
  } else if (status === 'Done') {
    return '#000000' // black
  } else if (status === 'Snoozed') {
    return '#5cb85c' // light green
  } else {
    return '#00FF00' // ERROR: big green
  }
}

// we cannot use Array<cytoscape.Stylesheet> because we are using mapping functions,
// which breaks the type notations of cytoscape... so we just use any here
function get_cytoscape_style(): any {
  let node_basic_style = {
    // basic shape
    shape: 'round-tag',
    height: '20px',
    width: '20px',
    'background-color': get_status_color,
    // boarder
    'border-width': 2,
    'border-style': 'solid',
    'border-color': '#000000',
    // padding
    padding: '0px',
    // label
    label: 'data(name)',
    'font-family': 'Arial, SimHei',
    'font-size': 18,
    'font-weight': 'normal',
    'text-halign': 'right',
    'text-valign': 'center',
    'text-margin-x': '4px',
    'text-background-padding': '2px',
    'text-border-opacity': 0.2,
    'text-border-width': 2,
    'text-border-color': '#000000'
  }
  let node_style = { ...node_basic_style }
  let node_selected_style = { ...node_basic_style }
  node_selected_style['background-color'] = () => '#FF4D4F'
  let edge_basic_style = {
    width: 3,
    'line-color': '#CCCCCC',
    'target-arrow-color': '#CCCCCC',
    'target-arrow-shape': 'triangle',
    'arrow-scale': 1.5,
    'curve-style': 'bezier'
  }
  let edge_style = { ...edge_basic_style }
  let edge_selected_style = { ...edge_basic_style }
  edge_selected_style['line-color'] = '#333333'
  edge_selected_style['target-arrow-color'] = '#333333'
  let export_style = [
    {
      selector: 'node',
      style: node_style
    },
    {
      selector: 'node.selected',
      style: node_selected_style
    },
    {
      selector: 'edge',
      style: edge_style
    },
    {
      selector: 'edge.selected',
      style: edge_selected_style
    }
  ]
  return export_style
}

function get_cytoscape_layout(): any {
  return {
    name: projectOperationInputStore.projectDagViewInputState.layout,
    nodeDimensionsIncludeLabels: true, // for Dagre,
    rankDir: 'LR' // for Dagre, 'TB' for top to bottom flow, 'LR' for left to right,
  }
}

const cytoscapeContainer = ref(null)
const cytoscapeInstance = ref<cytoscape.Core>()

const initCytoscape = () => {
  if (cytoscapeContainer.value === null) {
    // Component not loaded yet, call this later.
    return
  }
  console.log('Cytoscape initialized.')
  cytoscapeInstance.value = cytoscape({
    container: cytoscapeContainer.value,
    elements: construct_cytoscape_data(currentProject.value),
    style: get_cytoscape_style(),
    layout: get_cytoscape_layout(),
    minZoom: 0.1,
    maxZoom: 2.0
  })
  cytoscapeInstance.value.on('tap', 'node', function (evt) {
    // [TODO]: move this piece of code to a seperate function, e.g. onTaskSelected(str new_id)
    // a task is selected, save the selection in input store and highlight it in DAG view
    var node = evt.target
    projectInputState.previous_selected_node = projectInputState.selected_node
    projectInputState.selected_node = node.id()
    if (projectInputState.previous_selected_node) {
      cytoscapeInstance.value?.$id(projectInputState.previous_selected_node).removeClass('selected')
    }
    node.addClass('selected')
    // when selecting a task, if the original task is under editing, abort the editing.
    if (isEditingTaskInDetailView.value) {
      isEditingTaskInDetailView.value = false
    }
  })
  cytoscapeInstance.value.on('tap', 'edge', function (evt) {
    var edge = evt.target
    projectInputState.previous_selected_edge = projectInputState.selected_edge
    projectInputState.selected_edge = edge.id()
    if (projectInputState.previous_selected_edge) {
      cytoscapeInstance.value?.$id(projectInputState.previous_selected_edge).removeClass('selected')
    }
    edge.addClass('selected')
  })
}

// ========== TaskGraph Logic =========
const projectInputState = projectOperationInputStore.projectWorkspaceInputState

const selectedTaskMeta = computed(() => {
  if (!currentProject.value) {
    return null
  }
  if (!projectInputState.selected_node) {
    return null
  }
  if (!(projectInputState.selected_node in currentProject.value.metadata)) {
    return null
  }
  return currentProject.value.metadata[projectInputState.selected_node]
})

const selectedTaskName = computed(() =>
  selectedTaskMeta.value ? selectedTaskMeta.value.name ?? 'No Name Available' : 'No Task Selected'
)

const selectedTaskDetail = computed(() =>
  selectedTaskMeta.value
    ? selectedTaskMeta.value.detail ?? 'No Detail Available'
    : 'No Detail Available'
)

const selectedTaskStatus = computed(() =>
  selectedTaskMeta.value ? selectedTaskMeta.value.status : null
)

const selectedTaskIssues = computed(() =>
  selectedTaskMeta.value ? selectedTaskMeta.value.issues : null
)

const selectedTaskSnoozeFormatted = computed(() => {
  const wake_after = selectedTaskMeta.value?.wake_after
  const snooze_reason = selectedTaskMeta.value?.snooze_reason
  if (wake_after) {
    const date_snooze = new Date(wake_after * 1000).toString()
    if (snooze_reason) {
      return 'Snoozed until: ' + date_snooze + ', Reason: ' + snooze_reason
    } else {
      return 'Snoozed until: ' + date_snooze
    }
  }
  return null
})

async function handleCytoscapeLayoutSelectChange() {
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

async function onResetDAGView() {
  cytoscapeInstance.value?.reset()
}

async function onAddOrModifyTask() {
  if (projectInputState.task_toolbox_location_select === 'sub') {
    await onAddSubTask()
  } else if (projectInputState.task_toolbox_location_select === 'super') {
    await onAddSuperTask()
  } else if (projectInputState.task_toolbox_location_select == 'self') {
    await onModifyTask()
  } else {
    console.log(
      `Unexpected task_toolbox_location_select value: ${projectInputState.task_toolbox_location_select}`
    )
  }
}

async function onAddSubTask() {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        add_sub_task: {
          parent: projectInputState.selected_node,
          name: projectInputState.add_new_task_name,
          detail: projectInputState.add_new_task_detail
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.info('New sub-task created')
      }
    })
  } else {
    message.error(
      'Please select a parent task for the new task before adding \
      (click a task in the DAG View to select it)'
    )
  }
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

async function onAddSuperTask() {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        add_super_task: {
          child: projectInputState.selected_node,
          name: projectInputState.add_new_task_name,
          detail: projectInputState.add_new_task_detail
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.info('New super-task created')
      }
    })
  } else {
    message.error(
      'Please select a child task for the new task before adding \
      (click a task in the DAG View to select it)'
    )
  }
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

async function onModifyTask() {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        modify_task: {
          uuid: projectInputState.selected_node,
          name: projectInputState.add_new_task_name,
          detail: projectInputState.add_new_task_detail
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.info('Task modified')
      }
    })
  } else {
    message.error(
      'Please select a task before editing \
      (click a task in the DAG View to select it)'
    )
  }
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

async function onClearNewTaskInput() {
  projectInputState.add_new_task_detail = ''
  projectInputState.add_new_task_name = ''
}

async function onSetTaskDone() {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        update_task_status: {
          uuid: projectInputState.selected_node,
          status: 'Done'
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.info('Task marked as done')
      }
    })
  } else {
    message.error(
      'Please select a task before the operation \
      (click a task in the DAG View to select it)'
    )
  }
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

async function onRemoveTask() {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        remove_task: {
          uuid: projectInputState.selected_node
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.warning('Task removed')
      }
    })
  } else {
    message.error(
      'Please select a task before the operation \
      (click a task in the DAG View to select it)'
    )
  }
  projectInputState.selected_node = null
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

async function onAddDependencies() {
  if (projectInputState.selected_node && projectInputState.add_dependency_selected_nodes) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        add_dependencies: {
          uuid: projectInputState.selected_node,
          dependencies: projectInputState.add_dependency_selected_nodes
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.warning('Dependencies added')
      }
    })
  } else {
    message.error(
      'Please select the task and dependencies before the operation \
      (click a task in the DAG View to select it as dependent, \
      select dependencies in the Task Toolbox)'
    )
  }
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

async function onRemoveDependency() {
  if (projectInputState.selected_edge) {
    const selected_edge = cytoscapeInstance.value?.$id(projectInputState.selected_edge)
    if (selected_edge) {
      await callRESTfulAPI(
        `projects/${projectUUID.value}`,
        'POST',
        JSON.stringify({
          remove_dependency: {
            uuid_super_task: selected_edge.target().id(),
            uuid_sub_task: selected_edge.source().id()
          }
        })
      ).then((response) => {
        if (response?.result == 'OK') {
          message.warning('Dependency removed')
        }
      })
    } else {
      console.error('Unexpected: selected edge does not exist in cytoscape data')
      console.log(projectInputState.selected_edge)
    }
  } else {
    message.error(
      'Please select an edge before the operation \
      (click an edge in the DAG View to select it)'
    )
  }
  projectInputState.selected_edge = null
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

const snoozeTaskModalOpen = ref<boolean>(false)

const showSnoozeTaskModal = () => {
  snoozeTaskModalOpen.value = true
}

const handleSnoozeTaskModalOk = (_e: MouseEvent) => {
  snoozeTask()
  snoozeTaskModalOpen.value = false
}

async function snoozeTask() {
  if (projectInputState.selected_node && projectInputState.snooze_task_until) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        snooze_task: {
          uuid: projectInputState.selected_node,
          snooze_until: projectInputState.snooze_task_until.unix(),
          reason: projectInputState.snooze_task_reason
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.warning('Snoozed task')
      }
    })
  } else {
    message.error(
      'Please select a task and a date before the operation \
      (click a task in the DAG View to select it)'
    )
  }
  projectInputState.selected_node = null
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

const isEditingTaskInDetailView = ref<boolean>(false)

function onEditTaskInDetailView() {
  if (projectInputState.selected_node) {
    // copy current task name and detail to inputs
    projectInputState.edit_task_name = selectedTaskName.value
    projectInputState.edit_task_detail = selectedTaskDetail.value
    // show editing input elements
    isEditingTaskInDetailView.value = true
  } else {
    message.error(
      'Please select a task before editing \
      (click a task in the DAG View to select it)'
    )
  }
}

async function onSaveTaskInDetailView() {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        modify_task: {
          uuid: projectInputState.selected_node,
          name: projectInputState.edit_task_name,
          detail: projectInputState.edit_task_detail
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.info('Task modified')
        isEditingTaskInDetailView.value = false
      }
    })
  } else {
    message.error(
      'Please select a task before editing \
      (click a task in the DAG View to select it)'
    )
  }
  if (projectUUID.value) await readProject(projectUUID.value).then(() => initCytoscape())
}

const newIssueModalOpen = ref<boolean>(false)

const showNewIssueModal = () => {
  if (projectInputState.selected_node) {
    newIssueModalOpen.value = true
  } else {
    message.error(
      'Please select a task before the operation \
      (click a task in the DAG View to select it)'
    )
  }
}

const handleNewIssueModalOk = (_e: MouseEvent) => {
  onOpenNewIssue()
  newIssueModalOpen.value = false
}

const issue_data_source = computed(() => {
  if (selectedTaskIssues.value) {
    let dataSourceOpen = []
    let dataSourceClosed = []
    for (const item in selectedTaskIssues.value) {
      if ('Open' == selectedTaskIssues.value[item].status) {
        dataSourceOpen.push({
          title: selectedTaskIssues.value[item].title,
          description: selectedTaskIssues.value[item].description,
          key: item
        })
      } else {
        dataSourceClosed.push({
          title: selectedTaskIssues.value[item].title,
          description: selectedTaskIssues.value[item].description,
          key: item
        })
      }
    }
    return {
      open: dataSourceOpen,
      closed: dataSourceClosed
    }
  } else
    return {
      open: [],
      closed: []
    }
})

const issue_columns = [
  {
    title: 'Title',
    dataIndex: 'title',
    key: 'title'
  },
  {
    title: 'Description',
    dataIndex: 'description',
    key: 'description'
  },
  {
    title: 'Operation',
    dataIndex: 'operation',
    key: 'operation'
  }
]

async function onOpenNewIssue() {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        open_issue: {
          task_uuid: projectInputState.selected_node,
          title: projectInputState.new_issue_title,
          description: projectInputState.new_issue_description
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        // issue open successful, notify the user and clear input
        message.warning('Issue opened')
        projectInputState.new_issue_title = ''
        projectInputState.new_issue_description = ''
      }
    })
  } else {
    message.error(
      'Please select a task before the operation \
      (click a task in the DAG View to select it)'
    )
  }
  // because issue operations do not change task topology, no need to re-render cytoscape,
  // just read back from API server to confirm changes.
  if (projectUUID.value) await readProject(projectUUID.value)
}

async function onCloseIssue(issue_uuid: string) {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        close_issue: {
          task_uuid: projectInputState.selected_node,
          issue_uuid: issue_uuid
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.warning('Issue closed')
      }
    })
  } else {
    message.error(
      'Please select a task before the operation \
      (click a task in the DAG View to select it)'
    )
  }
  // because issue operations do not change task topology, no need to re-render cytoscape,
  // just read back from API server to confirm changes.
  if (projectUUID.value) await readProject(projectUUID.value)
}

async function onDeleteIssue(issue_uuid: string) {
  if (projectInputState.selected_node) {
    await callRESTfulAPI(
      `projects/${projectUUID.value}`,
      'POST',
      JSON.stringify({
        delete_issue: {
          task_uuid: projectInputState.selected_node,
          issue_uuid: issue_uuid
        }
      })
    ).then((response) => {
      if (response?.result == 'OK') {
        message.warning('Issue deleted')
      }
    })
  } else {
    message.error(
      'Please select a task before the operation \
      (click a task in the DAG View to select it)'
    )
  }
  // because issue operations do not change task topology, no need to re-render cytoscape,
  // just read back from API server to confirm changes.
  if (projectUUID.value) await readProject(projectUUID.value)
}
</script>

<style scoped>
.project-page-header :deep(tr:last-child td) {
  padding-bottom: 0;
}
.project-page-header :deep(tr:last-child td) {
  padding-bottom: 0;
}
.cytoscape-container {
  width: 100%;
  height: 500px;
}
</style>
