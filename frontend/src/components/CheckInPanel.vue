<template>
	<div class="flex flex-col bg-white rounded w-full py-6 px-4 border-none">
		<h2 class="text-lg font-bold text-gray-900">
			{{ __("Hey, {0} 👋", [employee?.data?.first_name]) }}
		</h2>

		<template v-if="allowCheckinFromMobile">
			<div
				v-if="lastLog && lastLog.time"
				class="font-medium text-sm text-gray-500 mt-1.5"
			>
				<span>
					{{ __("Last {0} was at {1}", [__(lastLogType), formatTimestamp(lastLog.time)]) }}
				</span>
				<span class="whitespace-pre"> &middot; </span>
				<router-link :to="{ name: 'EmployeeCheckinListView' }" v-slot="{ navigate }">
					<span @click="navigate" class="underline cursor-pointer">
						{{ __("View List") }}
					</span>
				</router-link>
			</div>

			<Button
				v-if="hasSubmittedWFHRequest"
				class="mt-4 mb-1 drop-shadow-sm py-5 text-base"
				:loading="checkins.list.loading || wfhRequests.loading"
				@click="handleEmployeeCheckin"
			>
				<template #prefix>
					<FeatherIcon
						:name="nextAction.action === 'IN' ? 'arrow-right-circle' : 'arrow-left-circle'"
						class="w-4"
					/>
				</template>
				{{ nextAction.label }}
			</Button>

			<div v-else-if="!wfhRequests.loading" class="mt-4 text-sm text-gray-500">
				{{
					__(
						"Check-in is available only for employees with a submitted Work From Home request covering today."
					)
				}}
			</div>
		</template>

		<div class="font-medium text-sm text-gray-500 mt-1.5">
			{{ dayjs().format("ddd, D MMMM, YYYY") }}
		</div>
	</div>

	<ion-modal
		v-if="allowCheckinFromMobile"
		ref="modal"
		:is-open="isModalOpen"
		:initial-breakpoint="1"
		:breakpoints="[0, 1]"
		@didPresent="handleModalPresented"
		@didDismiss="handleModalDismiss"
	>
		<div class="h-120 w-full flex flex-col items-center justify-center gap-5 p-4 mb-5">

			<div class="flex flex-col gap-1.5 mt-2 items-center justify-center">
				<div class="font-bold text-xl">
					{{ dayjs(checkinTimestamp).format("hh:mm:ss a") }}
				</div>
				<div class="font-medium text-gray-500 text-sm">
					{{ dayjs().format("D MMM, YYYY") }}
				</div>
			</div>

			<template v-if="allowGeolocationTracking">
				<div class="w-full flex items-center gap-2">
					<FeatherIcon v-if="locationFetching" name="loader" class="w-4 h-4 animate-spin text-gray-400" />
					<FeatherIcon v-else-if="locationReady" name="check-circle" class="w-4 h-4 text-green-500" />
					<FeatherIcon v-else-if="locationError" name="alert-circle" class="w-4 h-4 text-red-500" />
					<span class="font-medium text-gray-500 text-sm">{{ locationStatus }}</span>
					<button
						v-if="locationError"
						type="button"
						class="ml-auto text-sm underline text-gray-700"
						@click="fetchLocation"
					>
						{{ __("Retry") }}
					</button>
				</div>
				<div ref="mapContainer" class="rounded border-4 translate-z-0 block overflow-hidden w-full" style="height: 200px;" />
			</template>

			<Button
				:loading="insertCheckin.loading"
				:disabled="allowGeolocationTracking && !locationReady"
				variant="solid"
				class="w-full py-5 text-sm disabled:bg-gray-300 disabled:cursor-not-allowed"
				@click="submitLog(nextAction.action)"
			>
				{{
					allowGeolocationTracking && locationFetching
						? __("Fetching location…")
						: __("Confirm {0}", [nextAction.label])
				}}
			</Button>
		</div>
	</ion-modal>
</template>

<script setup>
import { createResource, createListResource, toast, FeatherIcon } from "frappe-ui"
import { computed, inject, nextTick, onMounted, onUnmounted, ref, watch } from "vue"
import { IonModal } from "@ionic/vue"

import { formatTimestamp } from "@/utils/formatters"

const DOCTYPE = "Employee Checkin"

const employee = inject("$employee")
const dayjs    = inject("$dayjs")
const __       = inject("$translate")

// ── Helpers ────────────────────────────────────────────────────────────────
function showError(text, title = __("Error")) {
	toast({
		title,
		text,
		icon: "alert-circle",
		position: "bottom-center",
		iconClasses: "text-red-500",
	})
}

function hasCsrfToken() {
	const token = window.csrf_token
	return !!token && token !== "None" && !String(token).includes("{{")
}

function isCsrfError(error) {
	try {
		return JSON.stringify(error || "").includes("CSRFTokenError")
	} catch {
		return false
	}
}

// ── HR Settings ────────────────────────────────────────────────────────────
const allowCheckinFromMobile   = ref(false)
const allowGeolocationTracking = ref(false)

const hrSettings = createResource({
	url: "hrms.api.get_hr_settings",
	onSuccess(data) {
		allowCheckinFromMobile.value   = !!data.allow_employee_checkin_from_mobile_app
		allowGeolocationTracking.value = !!data.allow_geolocation_tracking
	},
	// Fail closed: if settings can't be loaded, don't enable check-in
	onError() {
		allowCheckinFromMobile.value   = false
		allowGeolocationTracking.value = false
	},
})

// ── Insert resource ────────────────────────────────────────────────────────
// Dedicated createResource so latitude/longitude are included in the payload
// (createListResource.insert strips unknown fields)
const insertCheckin = createResource({
	url: "frappe.client.insert",
	makeParams(values) {
		return { doc: values }
	},
})

// ── Modal / checkin state ──────────────────────────────────────────────────
const checkinTimestamp = ref(null)
const isModalOpen      = ref(false)

// ── Location state ─────────────────────────────────────────────────────────
const locationFetching = ref(false)
const locationReady    = ref(false)
const locationError    = ref(false)
const latitude         = ref(0)
const longitude        = ref(0)
const locationStatus   = ref("")

const mapContainer = ref(null)
let leafletMap     = null
let leafletMarker  = null

// ── Computed ───────────────────────────────────────────────────────────────
const employeeId = computed(() => employee?.data?.name || "")

// `today` is a ref (not a computed) so it can be refreshed if the app stays open past midnight
const today = ref(dayjs().format("YYYY-MM-DD"))
let midnightTimer = null

function refreshToday() {
	const now = dayjs().format("YYYY-MM-DD")
	if (now !== today.value) today.value = now
}

// ── Leaflet (loaded from CDN, no npm package needed) ───────────────────────
let leafletPromise = null

function loadLeaflet() {
	if (window.L) return Promise.resolve(window.L)
	if (leafletPromise) return leafletPromise

	leafletPromise = new Promise((resolve, reject) => {
		if (!document.getElementById("leaflet-css")) {
			const link = document.createElement("link")
			link.id   = "leaflet-css"
			link.rel  = "stylesheet"
			link.href = "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
			document.head.appendChild(link)
		}

		const script   = document.createElement("script")
		script.src     = "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
		script.onload  = () => resolve(window.L)
		script.onerror = () => {
			leafletPromise = null // allow a retry later
			script.remove()
			reject(new Error("Failed to load Leaflet"))
		}
		document.head.appendChild(script)
	})

	return leafletPromise
}

async function initMap(lat, lng) {
	let L
	try {
		L = await loadLeaflet()
	} catch (e) {
		// Map is optional: the coordinates are still captured, so don't block check-in
		console.error(e)
		return
	}

	await nextTick()
	if (!mapContainer.value) return
	destroyMap()

	leafletMap = L.map(mapContainer.value).setView([lat, lng], 16)
	L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
		attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
		maxZoom: 19,
	}).addTo(leafletMap)
	leafletMarker = L.marker([lat, lng]).addTo(leafletMap)
	leafletMarker.bindPopup(__("You are here")).openPopup()

	// The modal is animating when the map is created, so Leaflet measures the wrong size
	setTimeout(() => leafletMap?.invalidateSize(), 200)
}

function destroyMap() {
	if (leafletMap) {
		leafletMap.remove()
		leafletMap = null
		leafletMarker = null
	}
}

// ── Location ───────────────────────────────────────────────────────────────
function fetchLocation() {
	if (!window.isSecureContext) {
		locationFetching.value = false
		locationReady.value    = false
		locationError.value    = true
		locationStatus.value   = __("Location requires a secure (HTTPS) connection")
		return
	}

	if (!navigator.geolocation) {
		locationFetching.value = false
		locationReady.value    = false
		locationError.value    = true
		locationStatus.value   = __("Geolocation is not supported by your browser")
		return
	}

	locationFetching.value = true
	locationReady.value    = false
	locationError.value    = false
	locationStatus.value   = __("Locating…")

	navigator.geolocation.getCurrentPosition(
		(position) => {
			latitude.value  = position.coords.latitude
			longitude.value = position.coords.longitude
			locationFetching.value = false
			locationReady.value    = true
			locationError.value    = false
			locationStatus.value   = [
				__("Lat: {0}°", [Number(latitude.value).toFixed(5)]),
				__("Lng: {0}°", [Number(longitude.value).toFixed(5)]),
			].join("  ·  ")

			if (leafletMap && leafletMarker) {
				leafletMarker.setLatLng([latitude.value, longitude.value])
				leafletMap.setView([latitude.value, longitude.value], 16)
			} else {
				initMap(latitude.value, longitude.value)
			}
		},
		(error) => {
			locationFetching.value = false
			locationReady.value    = false
			locationError.value    = true
			locationStatus.value   =
				error?.code === 1
					? __("Location permission denied. Allow location access and retry.")
					: __("Unable to retrieve location")
			if (error && error.code !== 1) locationStatus.value += `: (${error.code}) ${error.message}`
		},
		{ enableHighAccuracy: true, timeout: 15000, maximumAge: 0 }
	)
}

// ── List resources ─────────────────────────────────────────────────────────
const checkins = createListResource({
	doctype: DOCTYPE,
	fields: ["name", "employee", "employee_name", "log_type", "time", "device_id"],
	filters: [],
	orderBy: "time desc",
	auto: false,
})

const wfhRequests = createListResource({
	doctype: "Attendance Request",
	fields: ["name", "employee", "from_date", "to_date", "reason", "docstatus"],
	filters: [],
	auto: false,
})

watch(
	[employeeId, today],
	async ([id, date]) => {
		if (!id) return
		checkins.update({ filters: { employee: id } })
		wfhRequests.update({
			filters: [
				["employee", "=", id],
				["reason", "=", "Work From Home"],
				["from_date", "<=", date],
				["to_date", ">=", date],
				["docstatus", "=", 1],
			],
		})
		await Promise.all([checkins.reload(), wfhRequests.reload()])
	},
	{ immediate: true }
)

const hasSubmittedWFHRequest = computed(() => {
	const requests = wfhRequests.data || []
	return requests.some(
		(r) =>
			r.employee === employeeId.value &&
			r.reason === "Work From Home" &&
			Number(r.docstatus) === 1 &&
			r.from_date <= today.value &&
			r.to_date >= today.value
	)
})

const lastLog     = computed(() => (!checkins.list.loading && checkins.data ? checkins.data[0] || null : null))
const lastLogType = computed(() => (lastLog.value?.log_type === "IN" ? "check-in" : "check-out"))
const nextAction  = computed(() =>
	lastLog.value?.log_type === "IN"
		? { action: "OUT", label: __("Check Out") }
		: { action: "IN",  label: __("Check In")  }
)

// ── Handlers ───────────────────────────────────────────────────────────────
function handleEmployeeCheckin() {
	// Release focus from the button so Ionic's aria-hidden on the page doesn't hide a focused element
	document.activeElement?.blur()

	refreshToday()

	if (!hasSubmittedWFHRequest.value) {
		showError(__("A submitted Work From Home request covering today is required."), __("Not Allowed"))
		return
	}

	checkinTimestamp.value = dayjs().format("YYYY-MM-DD HH:mm:ss")
	latitude.value         = 0
	longitude.value        = 0
	locationFetching.value = false
	locationReady.value    = false
	locationError.value    = false
	locationStatus.value   = ""

	// Without geolocation there's nothing to wait for
	if (!allowGeolocationTracking.value) {
		locationReady.value = true
	}

	// Location fetching starts in handleModalPresented, once the modal content exists in the DOM
	isModalOpen.value = true
}

function handleModalPresented() {
	if (allowGeolocationTracking.value) fetchLocation()
}

function handleModalDismiss() {
	isModalOpen.value = false
	destroyMap()
}

async function submitLog(logType) {
	if (!hasSubmittedWFHRequest.value || !employeeId.value) return

	if (allowGeolocationTracking.value && !locationReady.value) {
		showError(__("Please wait for location to be fetched before checking in."), __("Location Required"))
		return
	}

	const actionLabel = logType === "IN" ? __("Check-in") : __("Check-out")

	// Frappe rejects POSTs without a valid CSRF token. A missing token means the app is not
	// being served by Frappe on this origin, so fail early with a clear message.
	if (!hasCsrfToken()) {
		showError(__("Session expired. Please reload the app and try again."))
		return
	}

	await insertCheckin.submit(
		{
			doctype:   DOCTYPE,
			employee:  employeeId.value,
			log_type:  logType,
			time:      checkinTimestamp.value,
			latitude:  latitude.value,
			longitude: longitude.value,
		},
		{
			onSuccess() {
				destroyMap()
				isModalOpen.value = false
				toast({
					title: __("Success"),
					text: __("{0} successful!", [actionLabel]),
					icon: "check-circle",
					position: "bottom-center",
					iconClasses: "text-green-500",
				})
				checkins.reload()
				wfhRequests.reload()
			},
			onError(error) {
				const msgs = isCsrfError(error)
					? [__("Session expired. Please reload the app and try again.")]
					: error?.messages?.length
						? error.messages
						: [__("{0} failed!", [actionLabel])]

				for (const msg of msgs) showError(msg)
			},
		}
	)
}

// ── Lifecycle ──────────────────────────────────────────────────────────────
onMounted(() => {
	hrSettings.fetch()
	// Re-check the date every minute so `today` rolls over at midnight
	midnightTimer = setInterval(refreshToday, 60 * 1000)
})

onUnmounted(() => {
	destroyMap()
	if (midnightTimer) clearInterval(midnightTimer)
})
</script>
