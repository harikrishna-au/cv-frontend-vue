<template>
    <template v-if="sessionExpired">
        <div style="position:fixed;inset:0;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,0.5);z-index:9999;">
            <div style="background:#fff;padding:2rem;border-radius:8px;text-align:center;max-width:400px;">
                <h2>Session Expired</h2>
                <p>Your session has expired. Please log in again.</p>
                <p style="color:#888;font-size:0.875rem;">Redirecting to login...</p>
            </div>
        </div>
    </template>
    <template v-if="isLoading">
        <h1>Loading...</h1>
    </template>
    <template v-else-if="!isLoading && !hasAccess">
        <h1>403</h1>
    </template>
    <template v-else-if="!isLoading && hasAccess">
        <simulator />
    </template>
</template>

<script lang="ts">
export function getToken(name: string) {
    var match = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'))
    if (match) return match[2]
}
</script>

<script setup lang="ts">
import simulator from './simulator.vue'
import { onBeforeMount, ref, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { useAuthStore } from '#/store/authStore'
import { useSimulatorMobileStore } from '#/store/simulatorMobileStore'

const route = useRoute()
const hasAccess = ref(true)
const isLoading = ref(true)
const sessionExpired = ref(false)
const authStore = useAuthStore()
const simulatorMobileStore = useSimulatorMobileStore()

// check if user has edit access to the project
async function checkEditAccess() {
    await fetch(`/api/v1/projects/${window.logixProjectId}/check_edit_access`, {
        method: 'GET',
        headers: {
            Accept: 'application/json',
            Authorization: `Token ${getToken('cvt')}`,
        },
    }).then((res) => {
        // if user has edit access load circuit data
        if (res.ok) {
            res.json().then((data) => {
                authStore.setUserInfo(data.data)
                ;(window as any).isUserLoggedIn = true
                isLoading.value = false
            })
        } else if (res.status === 403) {
            // if user has no edit access show edit access denied page
            hasAccess.value = false
            isLoading.value = false
        } else if (res.status === 404) {
            hasAccess.value = false
            isLoading.value = false
        } else if (res.status === 401) {
            // if user is not logged in redirect to login page
            authStore.signOut()
            window.location.href = '/users/sign_in'
        }
    })
}

// get logged in user informaton when blank simulator is opened
async function getLoginData() {
    try {
        const response = await fetch('/api/v1/me', {
            method: 'GET',
            headers: {
                Accept: 'application/json',
                Authorization: `Token ${getToken('cvt')}`,
            },
        })
        if (response.ok) {
            const data = await response.json()
            authStore.setUserInfo(data.data)
            ;(window as any).isUserLoggedIn = true
        } else if (response.status === 401) {
            ;(window as any).isUserLoggedIn = false
            authStore.signOut()
            sessionExpired.value = true
            setTimeout(() => { window.location.href = '/users/sign_in' }, 3000)
        }
    } catch (err) {
        console.error(err)
    }
}

onBeforeMount(() => {
    // prioritize logixProjectId from window
    const windowLogixProjectId = (window as any).logixProjectId
    if (windowLogixProjectId && windowLogixProjectId !== '0') {
        ;(window as any).logixProjectId = windowLogixProjectId
        checkEditAccess()
    } else if (route.params.projectId) {
        ;(window as any).logixProjectId = route.params.projectId
        checkEditAccess()
    } else {
        // if projectId is not defined open blank simulator
        getLoginData()
        isLoading.value = false
    }
})

onMounted(() => {
    window.addEventListener('resize', checkShowSidebar)
    document.addEventListener('visibilitychange', onVisibilityChange)
})

function onVisibilityChange() {
    if (document.visibilityState === 'visible') {
        getLoginData()
    }
}

function checkShowSidebar() {
    simulatorMobileStore.showMobileView = window.innerWidth < simulatorMobileStore.minWidthToShowMobile ? true : false
}
</script>
