<template>
    <!-- Existing Open Project Dialog -->
    <v-dialog
        v-model="SimulatorState.dialogBox.open_project_dialog"
        :persistent="false"
    >
        <v-card class="messageBoxContent">
            <v-card-text>
                <p class="dialogHeader">Open Offline</p>
                <v-btn
                    size="x-small"
                    icon
                    class="dialogClose"
                    @click="
                        SimulatorState.dialogBox.open_project_dialog = false
                    "
                >
                    <v-icon>mdi-close</v-icon>
                </v-btn>
                <div id="openProjectDialog" title="Open Project">
                    <label
                        v-for="(projectName, projectId) in projectList"
                        :key="projectId"
                        class="option custom-radio"
                    >
                        <input
                            type="radio"
                            name="projectId"
                            :value="projectId"
                        />
                        {{ projectName }}<span></span>
                        <i
                            class="fa fa-trash deleteOfflineProject"
                            @click="deleteOfflineProject(projectId)"
                        ></i>
                    </label>
                    <p v-if="JSON.stringify(projectList) == '{}'">
                        Looks like no circuit has been saved yet. Create a new
                        one and save it!
                    </p>
                </div>
            </v-card-text>
            <v-card-actions>
                <v-btn
                    v-if="JSON.stringify(projectList) != '{}'"
                    class="messageBtn"
                    block
                    @click="openProjectOffline()"
                >
                    open project
                </v-btn>
                <v-btn
                    v-else
                    class="messageBtn"
                    block
                    @click.stop="OpenImportProjectDialog"
                >
                    open CV file
                </v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>

    <!--  Version Mismatch Dialog -->
    <v-dialog
        v-model="SimulatorState.dialogBox.version_mismatch_dialog"
        :persistent="true"
    >
        <v-card class="messageBoxContent">
            <v-card-text>
                <p class="dialogHeader">Version Mismatch</p>
                <p>The version of the simulator does not match the expected version for this project.</p>
                <p>
                    You are attempting to open this project in version: 
                    {{ targetVersion }}
                </p>
            </v-card-text>
            <v-card-actions>
                <v-btn class="messageBtn" block @click="confirmOpenProject">
                    Continue Anyway
                </v-btn>
                <v-btn class="messageBtn" block @click="cancelOpenProject">Cancel</v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>
</template>

<script lang="ts" setup>
import load from '#/simulator/src/data/load'
import { useState } from '#/store/SimulatorStore/state'
import { onMounted, onUpdated, ref, toRaw } from '@vue/runtime-core'
const SimulatorState = useState()
const projectList = ref({})
const targetVersion = ref('') 
let projectName = '' 
onMounted(() => {
    SimulatorState.dialogBox.open_project_dialog = false
})

onUpdated(() => {
    var data = localStorage.getItem('projectList')
    projectList.value = JSON.parse(localStorage.getItem('projectList')) || {}
})

function deleteOfflineProject(id) {
    localStorage.removeItem(id)
    const temp = JSON.parse(localStorage.getItem('projectList')) || {}
    delete temp[id]
    projectList.value = temp
    localStorage.setItem('projectList', JSON.stringify(temp))
}

function openProjectOffline() {
    SimulatorState.dialogBox.open_project_dialog = false
    let ele = $('input[name=projectId]:checked')
    if (!ele.val()) return
    const projectData = JSON.parse(localStorage.getItem(ele.val()))
    const simulatorVersion = projectData.simulatorVersion
    projectName = projectData.name
    
    // Handle version mismatch logic
    if (!simulatorVersion) {
        // If no version, proceed directly
        targetVersion.value = "Legacy"
        SimulatorState.dialogBox.version_mismatch_dialog = true
    } else if (simulatorVersion && simulatorVersion != "v1") {
        // Set the targetVersion and show the version mismatch dialog
        targetVersion.value = simulatorVersion
        SimulatorState.dialogBox.version_mismatch_dialog = true
    } else {
        // For other cases, proceed normally
        load(projectData)
        window.projectId = ele.val()
    }
}

function confirmOpenProject() {
    SimulatorState.dialogBox.version_mismatch_dialog = false
    if(targetVersion.value == "Legacy"){
        window.location.href = `/simulator/edit/${projectName}`  
    }else{
        window.location.href = `/simulatorvue/edit/${projectName}?simver=${targetVersion.value}`
    }
}

function cancelOpenProject() {
    SimulatorState.dialogBox.version_mismatch_dialog = false
}

function OpenImportProjectDialog() {
    SimulatorState.dialogBox.open_project_dialog = false
    SimulatorState.dialogBox.import_project_dialog = true
}
</script>
