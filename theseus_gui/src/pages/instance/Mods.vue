<template>
  <Card v-if="projects.length > 0" class="mod-card">
    <div class="dropdown-input">
      <DropdownSelect
        v-model="selectedProjectType"
        :options="Object.keys(selectableProjectTypes)"
        default-value="All"
        name="project-type-dropdown"
        color="primary"
      />
      <div class="iconified-input">
        <SearchIcon />
        <input
          v-model="searchFilter"
          type="text"
          :placeholder="`Search ${search.length} ${(['All', 'Other'].includes(selectedProjectType)
            ? 'projects'
            : selectedProjectType.toLowerCase()
          ).slice(0, search.length === 1 ? -1 : 64)}...`"
          class="text-input"
          autocomplete="off"
        />
        <Button @click="() => (searchFilter = '')">
          <XIcon />
        </Button>
      </div>
    </div>
    <Button
      v-if="isPackLinked"
      v-tooltip="'Modpack is up to date'"
      :disabled="updatingModpack || !canUpdatePack"
      color="secondary"
      @click="updateModpack"
    >
      <UpdatedIcon />
      {{ updatingModpack ? 'Updating' : 'Update modpack' }}
    </Button>
    <Button v-else @click="exportModal.show()">
      <PackageIcon />
      Export modpack
    </Button>
    <DropdownButton
      v-if="!isPackLinked"
      :options="['search', 'from_file']"
      default-value="search"
      name="add-content-dropdown"
      color="primary"
      @option-click="handleContentOptionClick"
    >
      <template #search>
        <SearchIcon />
        <span class="no-wrap"> Add content </span>
      </template>
      <template #from_file>
        <FolderOpenIcon />
        <span class="no-wrap"> Add from file </span>
      </template>
    </DropdownButton>
  </Card>
  <Pagination
    v-if="projects.length > 0"
    :page="currentPage"
    :count="Math.ceil(search.length / 20)"
    class="pagination-before"
    :link-function="(page) => `?page=${page}`"
    @switch-page="switchPage"
  />
  <Card v-if="projects.length > 0" class="list-card">
    <div class="table">
      <div class="table-row table-head" :class="{ 'show-options': selected.length > 0 }">
        <div class="table-cell table-text">
          <Checkbox v-model="selectAll" class="select-checkbox" />
        </div>
        <div v-if="selected.length === 0" class="table-cell table-text name-cell actions-cell">
          <Button class="transparent" @click="sortProjects('Name')">
            Name
            <DropdownIcon v-if="sortColumn === 'Name'" :class="{ down: ascending }" />
          </Button>
        </div>
        <div v-if="selected.length === 0" class="table-cell table-text version">
          <Button class="transparent" @click="sortProjects('Version')">
            Version
            <DropdownIcon v-if="sortColumn === 'Version'" :class="{ down: ascending }" />
          </Button>
        </div>
        <div v-if="selected.length === 0" class="table-cell table-text actions-cell">
          <Button class="transparent" @click="sortProjects('Enabled')">
            Actions
            <DropdownIcon v-if="sortColumn === 'Enabled'" :class="{ down: ascending }" />
          </Button>
        </div>
        <div v-else class="options table-cell name-cell">
          <div>
            <Button
              class="transparent share"
              @click="() => (showingOptions = !showingOptions)"
              @mouseover="selectedOption = 'Share'"
            >
              <MenuIcon :class="{ open: showingOptions }" />
            </Button>
          </div>
          <Button
            class="transparent share"
            @click="shareNames()"
            @mouseover="selectedOption = 'Share'"
          >
            <ShareIcon />
            Share
          </Button>
          <div v-tooltip="isPackLinked ? 'Unpair this instance to remove mods' : ''">
            <Button
              :disabled="isPackLinked"
              class="transparent trash"
              @click="deleteWarning.show()"
              @mouseover="selectedOption = 'Delete'"
            >
              <TrashIcon />
              Delete
            </Button>
          </div>
          <div v-tooltip="isPackLinked ? 'Unpair this instance to update mods' : ''">
            <Button
              :disabled="isPackLinked || offline"
              class="transparent update"
              @click="updateAll()"
              @mouseover="selectedOption = 'Update'"
            >
              <UpdatedIcon />
              Update
            </Button>
          </div>
          <div v-tooltip="isPackLinked ? 'Unpair this instance to toggle mods' : ''">
            <Button
              :disabled="isPackLinked"
              class="transparent"
              @click="toggleSelected()"
              @mouseover="selectedOption = 'Toggle'"
            >
              <ToggleIcon />
              Toggle
            </Button>
          </div>
        </div>
      </div>
      <div v-if="showingOptions && selected.length > 0" class="more-box">
        <section v-if="selectedOption === 'Share'" class="options">
          <Button class="transparent" @click="shareNames()">
            <TextInputIcon />
            Share names
          </Button>
          <Button class="transparent" @click="shareUrls()">
            <GlobeIcon />
            Share URLs
          </Button>
          <Button class="transparent" @click="shareFileNames()">
            <FileIcon />
            Share file names
          </Button>
          <Button class="transparent" @click="shareMarkdown()">
            <CodeIcon />
            Share as markdown
          </Button>
        </section>
        <section v-if="selectedOption === 'Delete'" class="options">
          <Button class="transparent" @click="deleteWarning.show()">
            <TrashIcon />
            Delete selected
          </Button>
          <Button class="transparent" @click="deleteDisabledWarning.show()">
            <ToggleIcon />
            Delete disabled
          </Button>
        </section>
        <section v-if="selectedOption === 'Update'" class="options">
          <Button class="transparent" :disabled="offline" @click="updateAll()">
            <UpdatedIcon />
            Update all
          </Button>
          <Button class="transparent" @click="selectUpdatable()">
            <CheckIcon />
            Select updatable
          </Button>
        </section>
        <section v-if="selectedOption === 'Toggle'" class="options">
          <Button class="transparent" @click="enableAll()">
            <CheckIcon />
            Toggle on
          </Button>
          <Button class="transparent" @click="disableAll()">
            <XIcon />
            Toggle off
          </Button>
          <Button class="transparent" @click="hideShowAll()">
            <EyeIcon v-if="hideNonSelected" />
            <EyeOffIcon v-else />
            {{ hideNonSelected ? 'Show' : 'Hide' }} untoggled
          </Button>
        </section>
      </div>
      <div
        v-for="mod in search.slice((currentPage - 1) * 20, currentPage * 20)"
        :key="mod.file_name"
        class="table-row"
        @contextmenu.prevent.stop="(c) => handleRightClick(c, mod)"
      >
        <div class="table-cell table-text checkbox">
          <Checkbox
            :model-value="selectionMap.get(mod.path)"
            class="select-checkbox"
            @update:model-value="(newValue) => selectionMap.set(mod.path, newValue)"
          />
        </div>
        <div class="table-cell table-text name-cell">
          <router-link
            v-if="mod.slug"
            :to="{ path: `/project/${mod.slug}/`, query: { i: props.instance.path } }"
            :disabled="offline"
            class="mod-content"
          >
            <Avatar :src="mod.icon" />
            <div v-tooltip="`${mod.name} by ${mod.author}`" class="mod-text">
              <div class="title">{{ mod.name }}</div>
              <span class="no-wrap">by {{ mod.author }}</span>
            </div>
          </router-link>
          <div v-else class="mod-content">
            <Avatar :src="mod.icon" />
            <span v-tooltip="`${mod.name}`" class="title">{{ mod.name }}</span>
          </div>
        </div>
        <div class="table-cell table-text version">
          <span v-tooltip="`${mod.version}`">{{ mod.version }}</span>
        </div>
        <div class="table-cell table-text manage">
          <div v-tooltip="isPackLinked ? 'Unpair this instance to remove mods.' : ''">
            <Button
              v-tooltip="'Remove project'"
              :disabled="isPackLinked"
              icon-only
              @click="removeMod(mod)"
            >
              <TrashIcon />
            </Button>
          </div>
          <AnimatedLogo v-if="mod.updating" class="btn icon-only updating-indicator"></AnimatedLogo>
          <div v-tooltip="isPackLinked ? 'Unpair this instance to update mods.' : ''">
            <Button
              v-tooltip="'Update project'"
              :disabled="!mod.outdated || offline || isPackLinked"
              icon-only
              @click="updateProject(mod)"
            >
              <UpdatedIcon v-if="mod.outdated" />
              <CheckIcon v-else />
            </Button>
          </div>
          <div v-tooltip="isPackLinked ? 'Unpair this instance to toggle mods.' : ''">
            <input
              id="switch-1"
              :disabled="isPackLinked"
              autocomplete="off"
              type="checkbox"
              class="switch stylized-toggle"
              :checked="!mod.disabled"
              @change="toggleDisableMod(mod)"
            />
          </div>
          <Button
            v-tooltip="`Show ${mod.file_name}`"
            icon-only
            @click="showProfileInFolder(mod.path)"
          >
            <FolderOpenIcon />
          </Button>
        </div>
      </div>
    </div>
  </Card>
  <div v-else class="empty-prompt">
    <div class="empty-icon">
      <AddProjectImage />
    </div>
    <h3>No projects found</h3>
    <p class="empty-subtitle">Add a project to get started</p>
    <div class="empty-action">
      <DropdownButton
        :options="['search', 'from_file']"
        default-value="search"
        name="add-content-dropdown-from-empty"
        color="primary"
        @option-click="handleContentOptionClick"
      >
        <template #search>
          <SearchIcon />
          <span class="no-wrap"> Add content </span>
        </template>
        <template #from_file>
          <FolderOpenIcon />
          <span class="no-wrap"> Add from file </span>
        </template>
      </DropdownButton>
    </div>
  </div>
  <Modal ref="deleteWarning" header="Are you sure?">
    <div class="modal-body">
      <div class="markdown-body">
        <p>
          Are you sure you want to remove
          <strong>{{ functionValues.length }} project(s)</strong> from {{ instance.metadata.name }}?
          <br />
          This action <strong>cannot</strong> be undone.
        </p>
      </div>
      <div class="button-group push-right">
        <Button @click="deleteWarning.hide()"> Cancel </Button>
        <Button color="danger" @click="deleteSelected">
          <TrashIcon />
          Remove
        </Button>
      </div>
    </div>
  </Modal>
  <Modal ref="deleteDisabledWarning" header="Are you sure?">
    <div class="modal-body">
      <div class="markdown-body">
        <p>
          Are you sure you want to remove
          <strong
            >{{ Array.from(projects.values()).filter((x) => x.disabled).length }} disabled
            project(s)</strong
          >
          from {{ instance.metadata.name }}?
          <br />
          This action <strong>cannot</strong> be undone.
        </p>
      </div>
      <div class="button-group push-right">
        <Button @click="deleteDisabledWarning.hide()"> Cancel </Button>
        <Button color="danger" @click="deleteDisabled">
          <TrashIcon />
          Remove
        </Button>
      </div>
    </div>
  </Modal>
  <ShareModal
    ref="shareModal"
    share-title="Sharing modpack content"
    share-text="Check out the projects I'm using in my modpack!"
  />
  <ExportModal v-if="projects.length > 0" ref="exportModal" :instance="instance" />
</template>
<script setup>
import {
  Avatar,
  Button,
  TrashIcon,
  Card,
  CheckIcon,
  SearchIcon,
  UpdatedIcon,
  AnimatedLogo,
  FolderOpenIcon,
  Checkbox,
  formatProjectType,
  DropdownButton,
  Modal,
  XIcon,
  ShareIcon,
  DropdownIcon,
  GlobeIcon,
  FileIcon,
  EyeIcon,
  EyeOffIcon,
  ShareModal,
  CodeIcon,
  Pagination,
  DropdownSelect,
} from 'omorphia'
import { computed, onUnmounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import {
  add_project_from_path,
  get,
  remove_project,
  toggle_disable_project,
  update_all,
  update_managed_modrinth,
  update_project,
} from '@/helpers/profile.js'
import { handleError } from '@/store/notifications.js'
import { mixpanel_track } from '@/helpers/mixpanel'
import { open } from '@tauri-apps/api/dialog'
import { listen } from '@tauri-apps/api/event'
import { convertFileSrc } from '@tauri-apps/api/tauri'
import { showProfileInFolder } from '@/helpers/utils.js'
import { MenuIcon, ToggleIcon, TextInputIcon, AddProjectImage, PackageIcon } from '@/assets/icons'
import ExportModal from '@/components/ui/ExportModal.vue'

const router = useRouter()

const props = defineProps({
  instance: {
    type: Object,
    default() {
      return {}
    },
  },
  options: {
    type: Object,
    default() {
      return {}
    },
  },
  offline: {
    type: Boolean,
    default() {
      return false
    },
  },
})

const projects = ref([])
const selectionMap = ref(new Map())
const showingOptions = ref(false)
const isPackLinked = computed(() => {
  return props.instance.metadata.linked_data
})
const canUpdatePack = computed(() => {
  return props.instance.metadata.linked_data.version_id !== props.instance.modrinth_update_version
})
const exportModal = ref(null)

console.log(props.instance)
const initProjects = (initInstance) => {
  projects.value = []
  if (!initInstance || !initInstance.projects) return
  for (const [path, project] of Object.entries(initInstance.projects)) {
    if (project.metadata.type === 'modrinth' && !props.offline) {
      let owner = project.metadata.members.find((x) => x.role === 'Owner')
      projects.value.push({
        path,
        name: project.metadata.project.title,
        slug: project.metadata.project.slug,
        author: owner ? owner.user.username : null,
        version: project.metadata.version.version_number,
        file_name: project.file_name,
        icon: project.metadata.project.icon_url,
        disabled: project.disabled,
        updateVersion: project.metadata.update_version,
        outdated: !!project.metadata.update_version,
        project_type: project.metadata.project.project_type,
        id: project.metadata.project.id,
      })
    } else if (project.metadata.type === 'inferred') {
      projects.value.push({
        path,
        name: project.metadata.title ?? project.file_name,
        author: project.metadata.authors[0],
        version: project.metadata.version,
        file_name: project.file_name,
        icon: project.metadata.icon ? convertFileSrc(project.metadata.icon) : null,
        disabled: project.disabled,
        outdated: false,
        project_type: project.metadata.project_type,
      })
    } else {
      projects.value.push({
        path,
        name: project.file_name,
        author: '',
        version: null,
        file_name: project.file_name,
        icon: null,
        disabled: project.disabled,
        outdated: false,
        project_type: null,
      })
    }
  }

  const newSelectionMap = new Map()
  for (const project of projects.value) {
    newSelectionMap.set(
      project.path,
      selectionMap.value.get(project.path) ??
        selectionMap.value.get(project.path.slice(0, -9)) ??
        selectionMap.value.get(project.path + '.disabled') ??
        false
    )
  }
  selectionMap.value = newSelectionMap
}

initProjects(props.instance)

watch(
  () => props.instance.projects,
  () => {
    initProjects(props.instance)
  }
)

watch(
  () => props.offline,
  () => {
    if (props.instance) initProjects(props.instance)
  }
)

const searchFilter = ref('')
const selectAll = ref(false)
const selectedProjectType = ref('All')
const deleteWarning = ref(null)
const deleteDisabledWarning = ref(null)
const hideNonSelected = ref(false)
const selectedOption = ref('Share')
const shareModal = ref(null)
const ascending = ref(true)
const sortColumn = ref('Name')
const currentPage = ref(1)

const selected = computed(() =>
  Array.from(selectionMap.value)
    .filter((args) => {
      return args[1]
    })
    .map((args) => {
      return projects.value.find((x) => x.path === args[0])
    })
)

const functionValues = computed(() =>
  selected.value.length > 0 ? selected.value : Array.from(projects.value.values())
)

const selectableProjectTypes = computed(() => {
  const obj = { All: 'all' }

  for (const project of projects.value) {
    obj[project.project_type ? formatProjectType(project.project_type) + 's' : 'Other'] =
      project.project_type
  }

  return obj
})

const search = computed(() => {
  const projectType = selectableProjectTypes.value[selectedProjectType.value]
  const filtered = projects.value
    .filter((mod) => {
      return (
        mod.name.toLowerCase().includes(searchFilter.value.toLowerCase()) &&
        (projectType === 'all' || mod.project_type === projectType)
      )
    })
    .filter((mod) => {
      if (hideNonSelected.value) {
        return !mod.disabled
      }
      return true
    })

  return updateSort(filtered)
})

const updateSort = (projects) => {
  switch (sortColumn.value) {
    case 'Version':
      return projects.slice().sort((a, b) => {
        if (a.version < b.version) {
          return ascending.value ? -1 : 1
        }
        if (a.version > b.version) {
          return ascending.value ? 1 : -1
        }
        return 0
      })
    case 'Author':
      return projects.slice().sort((a, b) => {
        if (a.author < b.author) {
          return ascending.value ? -1 : 1
        }
        if (a.author > b.author) {
          return ascending.value ? 1 : -1
        }
        return 0
      })
    case 'Enabled':
      return projects.slice().sort((a, b) => {
        if (a.disabled && !b.disabled) {
          return ascending.value ? 1 : -1
        }
        if (!a.disabled && b.disabled) {
          return ascending.value ? -1 : 1
        }
        return 0
      })
    default:
      return projects.slice().sort((a, b) => {
        if (a.name < b.name) {
          return ascending.value ? -1 : 1
        }
        if (a.name > b.name) {
          return ascending.value ? 1 : -1
        }
        return 0
      })
  }
}

const sortProjects = (filter) => {
  if (sortColumn.value === filter) {
    ascending.value = !ascending.value
  } else {
    sortColumn.value = filter
    ascending.value = true
  }
}

const updateAll = async () => {
  const setProjects = []
  for (const [i, project] of projects.value.entries()) {
    if (project.outdated) {
      project.updating = true
      setProjects.push(i)
    }
  }

  const paths = await update_all(props.instance.path).catch(handleError)

  for (const [oldVal, newVal] of Object.entries(paths)) {
    const index = projects.value.findIndex((x) => x.path === oldVal)
    projects.value[index].path = newVal
    projects.value[index].outdated = false

    if (projects.value[index].updateVersion) {
      projects.value[index].version = projects.value[index].updateVersion.version_number
      projects.value[index].updateVersion = null
    }
  }
  for (const project of setProjects) {
    projects.value[project].updating = false
  }

  mixpanel_track('InstanceUpdateAll', {
    loader: props.instance.metadata.loader,
    game_version: props.instance.metadata.game_version,
    count: setProjects.length,
    selected: selected.value.length > 1,
  })
}

const selectUpdatable = () => {
  for (const project of projects.value) {
    if (project.outdated) {
      selectionMap.value.set(project.path, true)
    }
  }
}

const updateProject = async (mod) => {
  mod.updating = true
  mod.path = await update_project(props.instance.path, mod.path).catch(handleError)
  mod.updating = false

  mod.outdated = false
  mod.version = mod.updateVersion.version_number
  mod.updateVersion = null

  mixpanel_track('InstanceProjectUpdate', {
    loader: props.instance.metadata.loader,
    game_version: props.instance.metadata.game_version,
    id: mod.id,
    name: mod.name,
    project_type: mod.project_type,
  })
}

let locks = {}

const toggleDisableMod = async (mod) => {
  // Use mod's id as the key for the lock. If mod doesn't have a unique id, replace `mod.id` with some unique property.
  if (!locks[mod.id]) {
    locks[mod.id] = ref(null)
  }

  let lock = locks[mod.id]

  while (lock.value) {
    await lock.value
  }

  lock.value = toggle_disable_project(props.instance.path, mod.path)
    .then((newPath) => {
      mod.path = newPath
      mod.disabled = !mod.disabled
      mixpanel_track('InstanceProjectDisable', {
        loader: props.instance.metadata.loader,
        game_version: props.instance.metadata.game_version,
        id: mod.id,
        name: mod.name,
        project_type: mod.project_type,
        disabled: mod.disabled,
      })
    })
    .catch(handleError)
    .finally(() => {
      lock.value = null
    })

  await lock.value
}

const removeMod = async (mod) => {
  await remove_project(props.instance.path, mod.path).catch(handleError)
  projects.value = projects.value.filter((x) => mod.path !== x.path)

  mixpanel_track('InstanceProjectRemove', {
    loader: props.instance.metadata.loader,
    game_version: props.instance.metadata.game_version,
    id: mod.id,
    name: mod.name,
    project_type: mod.project_type,
  })
}

const deleteSelected = async () => {
  for (const project of functionValues.value) {
    await remove_project(props.instance.path, project.path).catch(handleError)
  }

  projects.value = projects.value.filter((x) => !x.selected)
  deleteWarning.value.hide()
}

const deleteDisabled = async () => {
  for (const project of Array.of(projects.value.values().filter((x) => x.disabled))) {
    await remove_project(props.instance.path, project.path).catch(handleError)
  }

  projects.value = projects.value.filter((x) => !x.selected)
  deleteDisabledWarning.value.hide()
}

const shareNames = async () => {
  await shareModal.value.show(functionValues.value.map((x) => x.name).join('\n'))
}

const shareFileNames = async () => {
  await shareModal.value.show(functionValues.value.map((x) => x.file_name).join('\n'))
}

const shareUrls = async () => {
  await shareModal.value.show(
    functionValues.value
      .filter((x) => x.slug)
      .map((x) => `https://modrinth.com/${x.project_type}/${x.slug}`)
      .join('\n')
  )
}

const shareMarkdown = async () => {
  await shareModal.value.show(
    functionValues.value
      .map((x) => {
        if (x.slug) {
          return `[${x.name}](https://modrinth.com/${x.project_type}/${x.slug})`
        }
        return x.name
      })
      .join('\n')
  )
}

const toggleSelected = async () => {
  for (const project of functionValues.value) {
    await toggleDisableMod(project, !project.disabled)
  }
}

const enableAll = async () => {
  for (const project of functionValues.value) {
    if (project.disabled) {
      await toggleDisableMod(project, false)
    }
  }
}

const disableAll = async () => {
  for (const project of functionValues.value) {
    if (!project.disabled) {
      await toggleDisableMod(project, false)
    }
  }
}

const hideShowAll = async () => {
  hideNonSelected.value = !hideNonSelected.value
}

const handleRightClick = (event, mod) => {
  if (mod.slug && mod.project_type) {
    props.options.showMenu(
      event,
      {
        link: `https://modrinth.com/${mod.project_type}/${mod.slug}`,
      },
      [{ name: 'open_link' }, { name: 'copy_link' }]
    )
  }
}

const handleContentOptionClick = async (args) => {
  if (args.option === 'search') {
    await router.push({
      path: `/browse/${props.instance.metadata.loader === 'vanilla' ? 'datapack' : 'mod'}`,
      query: { i: props.instance.path },
    })
  } else if (args.option === 'from_file') {
    const newProject = await open({ multiple: true })
    if (!newProject) return

    for (const project of newProject) {
      await add_project_from_path(props.instance.path, project, 'mod').catch(handleError)
    }
    initProjects(await get(props.instance.path).catch(handleError))
  }
}

const updatingModpack = ref(false)
const updateModpack = async () => {
  updatingModpack.value = true
  await update_managed_modrinth(props.instance.path).catch(handleError)
  updatingModpack.value = false
}

watch(selectAll, () => {
  for (const [key, value] of Array.from(selectionMap.value)) {
    if (value !== selectAll.value) {
      selectionMap.value.set(key, selectAll.value)
    }
  }
})

const unlisten = await listen('tauri://file-drop', async (event) => {
  for (const file of event.payload) {
    if (file.endsWith('.mrpack')) continue
    await add_project_from_path(props.instance.path, file, 'mod').catch(handleError)
  }
  initProjects(await get(props.instance.path).catch(handleError))
})

const switchPage = (page) => {
  currentPage.value = page
}

onUnmounted(() => {
  unlisten()
})
</script>

<style scoped lang="scss">
.text-input {
  width: 100%;
}

.manage {
  display: flex;
  gap: 0.5rem;
}

.table {
  margin-block-start: 0;
  border-radius: var(--radius-lg);
  border: 2px solid var(--color-bg);
}

.table-row {
  grid-template-columns: min-content 2fr 1fr 13.25rem;

  &.show-options {
    grid-template-columns: min-content auto;

    .options {
      display: flex;
      flex-direction: row;
      align-items: center;
      gap: var(--gap-md);
    }
  }
}

.static {
  .table-row {
    grid-template-areas: 'manage name version';
    grid-template-columns: 4.25rem 1fr 1fr;
  }

  .name-cell {
    grid-area: name;
  }

  .version {
    grid-area: version;
  }

  .manage {
    justify-content: center;
    grid-area: manage;
  }
}

.table-cell {
  align-items: center;
}

.card-row {
  display: flex;
  align-items: center;
  gap: var(--gap-md);
  justify-content: space-between;
  background-color: var(--color-raised-bg);
}

.mod-card {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  gap: var(--gap-sm);
  justify-content: flex-start;
  margin-bottom: 0.5rem;
  white-space: nowrap;
  align-items: center;

  :deep(.dropdown-row) {
    .btn {
      height: 2.5rem !important;
    }
  }

  .btn {
    height: 2.5rem;
  }

  .dropdown-input {
    flex-grow: 1;

    .iconified-input {
      width: 100%;

      input {
        flex-basis: unset;
      }
    }

    :deep(.animated-dropdown) {
      .render-down {
        border-radius: var(--radius-md) 0 0 var(--radius-md) !important;
      }

      .options-wrapper {
        margin-top: 0.25rem;
        width: unset;
        border-radius: var(--radius-md);
      }

      .options {
        border-radius: var(--radius-md);
        border: 1px solid var(--color);
      }
    }
  }
}

.list-card {
  margin-top: 0.5rem;
}

.text-combo {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.name-cell {
  padding-left: 0;

  .btn {
    margin-left: var(--gap-sm);
    min-width: unset;
  }
}

.dropdown {
  width: 7rem !important;
}

.sort {
  padding-left: 0.5rem;
}

.second-row {
  display: flex;
  align-items: flex-start;
  flex-wrap: wrap;
  gap: var(--gap-sm);

  .chips {
    flex-grow: 1;
  }
}

.modal-body {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: var(--gap-lg);

  .button-group {
    display: flex;
    justify-content: flex-end;
    gap: 0.5rem;
  }

  strong {
    color: var(--color-contrast);
  }
}

.mod-content {
  display: flex;
  align-items: center;
  gap: 1rem;

  .mod-text {
    display: flex;
    flex-direction: column;
  }

  .title {
    color: var(--color-contrast);
    font-weight: bolder;
  }
}

.actions-cell {
  display: flex;
  align-items: center;
  gap: 0.5rem;

  .btn {
    height: unset;
    width: unset;
    padding: 0;

    &.trash {
      color: var(--color-red);
    }

    &.update {
      color: var(--color-green);
    }

    &.share {
      color: var(--color-blue);
    }
  }
}

.more-box {
  display: flex;
  background-color: var(--color-bg);
  padding: var(--gap-lg);

  .options {
    display: flex;
    flex-wrap: wrap;
    flex-direction: row;
    gap: var(--gap-md);
    flex-grow: 1;
  }
}

.btn {
  &.transparent {
    height: unset;
    width: unset;
    padding: 0;
    color: var(--color-base);
    gap: var(--gap-xs);
    white-space: nowrap;

    svg {
      margin-right: 0 !important;
      transition: transform 0.2s ease-in-out;

      &.open {
        transform: rotate(90deg);
      }

      &.down {
        transform: rotate(180deg);
      }
    }
  }
}
.empty-prompt {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: var(--gap-md);
  height: 100%;
  width: 100%;
  margin: auto;

  .empty-icon {
    svg {
      width: 10rem;
      height: 10rem;
      color: var(--color-contrast);
    }
  }

  p,
  h3 {
    margin: 0;
  }
}
</style>
<style lang="scss">
.updating-indicator {
  svg {
    margin-left: 0.5rem !important;
  }
}

.v-popper--theme-tooltip .v-popper__inner {
  background: #fff !important;
}

.select-checkbox {
  button.checkbox {
    border: none;
    margin: 0;
  }
}

.dropdown-input {
  .selected {
    height: 2.5rem;
  }
}
</style>
