<template>
  <div id="app">
    <h1>{{ libraryName }} Room Reservation System</h1>

    <div class="header">
      <p>Viewing: {{ formatDate(currentDate) }}</p>
      <div>
        <button v-if="isToday" @click="goToTomorrow">Tomorrow</button>
        <button v-if="!isToday" @click="goToToday">Today</button>
      </div>
    </div>

  <nav class="filters">
      <label>
        Zone
        <select v-model="selectedZone">
          <option v-for="zone in zones" :key="zone" :value="zone">{{ zone }}</option>
        </select>
      </label>
      <label>
        Room
        <select v-model="selectedRoomName">
          <option v-for="name in roomNames" :key="name" :value="name">{{ name }}</option>
        </select>
      </label>
      <label>
        Capacity
        <select v-model="selectedCapacity">
          <option value="All">All</option>
          <option v-for="cap in capacities" :key="cap" :value="cap">{{ cap }}</option>
        </select>
      </label>
    </nav>

    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>

    <div v-else class="rooms">
      <div v-if="filteredSortedRooms.length === 0" class="no-rooms">No rooms available for this date.</div>
      <div v-for="room in filteredSortedRooms" :key="room.id" class="room-card" :class="{ expanded: expandedRooms.includes(room.id) }" @click="toggleExpanded(room.id)">
  <h3>{{ room.name }}</h3>
  <div class="room-meta">Capacity: <strong>{{ room.capacity ?? 'Unknown' }}</strong></div>
        <div class="timeline">
          <template v-if="!expandedRooms.includes(room.id)">
            <div class="time-label" style="left: 0%;">8 AM</div>
            <div class="time-label" style="left: 50%;">12 PM</div>
            <div class="time-label" style="right: 0;">11 PM</div>
          </template>
          <template v-else>
            <div
              v-for="i in 16"
              :key="i"
              class="time-label"
              :style="{ left: `${((i - 1) / 15) * 100}%` }"
            >
              {{ ((i + 7) % 12 || 12) }} {{ i + 7 >= 12 ? 'PM' : 'AM' }}
            </div>
          </template>
          <div
            v-for="booking in getBookingsForRoom(room.id)"
            :key="booking.id"
            class="booking-segment"
            :style="getBookingStyle(booking)"
            :title="formatTime(booking.fromDate) + ' - ' + formatTime(booking.toDate)"
          ></div>
        </div>
      </div>
    </div>

    <div v-if="expandedRooms.length > 0" class="backdrop" @click="expandedRooms = []"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue'
import roomZones from './roomZones.json'
import roomMeta from './roomMeta.json'

const libraryName = import.meta.env.VITE_LIBRARY_NAME
const baseUrl = import.meta.env.VITE_LIBCAL_BASE_URL
const token = import.meta.env.VITE_LIBCAL_TOKEN
const locationId = import.meta.env.VITE_LOCATION_ID
const roomItemIds = import.meta.env.VITE_ROOM_ITEM_IDS ? import.meta.env.VITE_ROOM_ITEM_IDS.split(',').map(id => Number(id.trim())) : []

// Simple in-memory cache for fetched item metadata to avoid refetching the same ids
const itemCache = {}

// Helpers to normalize booking and room shapes and to dedupe bookings
const normalizeBooking = (b) => {
  if (!b) return b
  const fromDate = b.fromDate || b.from || b.start || b.startDate || b.start_time || b.startTime
  const toDate = b.toDate || b.to || b.end || b.endDate || b.end_time || b.endTime
  const eid = (b.eid !== undefined && b.eid !== null) ? Number(b.eid) : (b.item_id !== undefined && b.item_id !== null ? Number(b.item_id) : undefined)
  const id = (b.id !== undefined && b.id !== null) ? Number(b.id) : undefined
  const status = (b.status !== undefined && b.status !== null) ? String(b.status) : undefined
  return { ...b, eid, id, status, fromDate, toDate }
}

const dedupeBookings = (arr) => {
  const seen = new Set()
  const out = []
  for (const b of (arr || [])) {
    const key = `${b.eid}|${b.fromDate}|${b.toDate}`
    if (seen.has(key)) continue
    seen.add(key)
    out.push(b)
  }
  return out
}

// Global request throttling wrapper
const REQUEST_DELAY_MS = 150 // spacing between requests
let _lastRequestAt = 0
const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))
const apiFetch = async (input, init = {}) => {
  const now = Date.now()
  const since = now - _lastRequestAt
  if (since < REQUEST_DELAY_MS) {
    await sleep(REQUEST_DELAY_MS - since)
  }
  try {
    const res = await fetch(input, init)
    _lastRequestAt = Date.now()
    if (res.status === 401 || res.status === 403) {
      // authorization problems - surface a clear error and stop further requests
      let txt = ''
      try { txt = await res.text() } catch (e) { /* ignore */ }
      const msg = `Authorization error (${res.status}). Token may be missing or expired. ${txt}`
      console.error(msg)
      if (typeof error !== 'undefined') error.value = 'Authorization error: please update VITE_LIBCAL_TOKEN in .env (token missing or expired)'
      throw new Error(msg)
    }
    if (res.status >= 400 && res.status < 500) {
      // log client errors with body
      try {
        const txt = await res.text()
        console.warn(`apiFetch client error ${res.status} ${input}: ${txt}`)
      } catch (e) {
        console.warn(`apiFetch client error ${res.status} ${input}`)
      }
    }
    return res
  } catch (e) {
    _lastRequestAt = Date.now()
    throw e
  }
}

const rooms = ref([])
const bookings = ref([])
const loading = ref(true)
const error = ref('')
const expandedRooms = ref([])
const currentDate = ref(new Date())
const selectedZone = ref('All')
const selectedRoomName = ref('All')
const selectedCapacity = ref('All')

const fetchRooms = async () => {
  // Rooms will be set from bookings
}

const inferZoneFromName = (name) => {
  if (!name) return 'Uncategorized'
  const n = name.toLowerCase()
  if (n.includes('llsr') || n.includes('lower')) return 'Lower Level'
  const studyMatch = name.match(/study\s*(\d+)/i)
  if (studyMatch) {
    const num = parseInt(studyMatch[1], 10)
    if (!isNaN(num)) {
      if (num >= 300) return 'Third Floor'
      if (num >= 200) return 'Second Floor'
      return 'First Floor'
    }
  }
  // fallback to first-floor style if the name contains '1' or 'Main'
  if (n.match(/\b1\b/) || n.includes('main')) return 'First Floor'
  return 'Uncategorized'
}

const buildRoomsFromBookings = (bookingList) => {
  // Build a base rooms list from bookings (names/zones) and include env-configured item ids
  const bookingIds = Array.isArray(bookingList) ? bookingList.map(b => Number(b.eid)) : []
  const idSet = new Set([ ...(Array.isArray(roomItemIds) ? roomItemIds.map(Number) : []), ...bookingIds ])
  const ids = Array.from(idSet)
  const baseRooms = ids.map(id => {
    const b = (bookingList || []).find(x => Number(x.eid) === Number(id))
    return {
      id: Number(id),
      name: b ? b.item_name : `Room ${id}`,
      zone: b ? b.category_name : 'Uncategorized',
      capacity: roomMeta && roomMeta[id] ? Number(roomMeta[id]) : null
    }
  })
  rooms.value = baseRooms
  return ids
}

const fetchBookings = async () => {
  try {
    const dateStr = currentDate.value.toISOString().split('T')[0]
    console.log('Fetching bookings for', dateStr)
  const bookingsUrl = `${baseUrl}/space/bookings?lid=${locationId}&date=${dateStr}`
  console.log('API Request:', bookingsUrl)
  const response = await apiFetch(bookingsUrl, {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    })
    console.log('Response status', response.status)
    if (!response.ok) {
      // No bookings; still build rooms from env ids and enrich
      bookings.value = []
      const ids = buildRoomsFromBookings([])
      try {
        await enrichRoomsWithAPIData(ids)
      } catch (e) {
        console.warn('Failed to enrich room metadata for empty bookings:', e)
      }
      return
    }
    const data = await response.json()
    console.log('Data length', data.length)
  const confirmedBookings = data.filter(booking => booking.status === 'Confirmed')
  // Normalize and dedupe bookings immediately
  const normalized = confirmedBookings.map(normalizeBooking)
  bookings.value = dedupeBookings(normalized)

  // Build rooms list from normalized bookings plus env IDs, then enrich with item details
  const ids = buildRoomsFromBookings(bookings.value)
    try {
      await enrichRoomsWithAPIData(ids)
    } catch (e) {
      console.warn('Failed to enrich room metadata:', e)
    }

    // Merge availability segments for ids so timelines reflect availability for env items too
    try {
      const avail = await fetchAvailabilityForIds(ids, dateStr)
      for (const seg of avail) {
        const segEid = Number(seg.eid)
        const normSeg = normalizeBooking({ ...seg, eid: segEid, status: seg.status || 'Confirmed' })
        const exists = bookings.value.some(b => b.eid === normSeg.eid && b.fromDate === normSeg.fromDate && b.toDate === normSeg.toDate)
        if (!exists) bookings.value.push(normSeg)
      }
      // Normalize and dedupe after merging availability
      bookings.value = dedupeBookings(bookings.value.map(normalizeBooking))
      // Re-run enrichment to pick up any metadata for ids added via availability
      try {
        await enrichRoomsWithAPIData(ids)
      } catch (e) {
        console.warn('Failed to re-enrich rooms after availability merge:', e)
      }
    } catch (e) {
      // ignore availability merge errors
    }
  } catch (err) {
    console.error('Fetch error', err)
    error.value = err.message
  }
}

const getBookingsForRoom = (roomId) => {
  const rid = Number(roomId)
  return bookings.value.filter(booking => Number(booking.eid) === rid && String(booking.status).toLowerCase() === 'confirmed')
}

const fetchItemDetails = async (ids) => {
  const map = {}
  // sanitize ids to numeric positives and populate from cache first
  const sanitized = (ids || []).map(id => Number(id)).filter(id => Number.isFinite(id) && id > 0)
  const invalid = (ids || []).filter(id => !sanitized.includes(Number(id)))
  if (invalid.length) console.warn('fetchItemDetails: skipping invalid ids', invalid)
  for (const id of sanitized) {
    if (itemCache[id]) map[id] = itemCache[id]
  }
  const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))
  // Throttle: run sequential requests with a small delay to avoid hitting API rate limits
  const delayMs = 150 // ~6-7 requests per second (well below 25 req/s)
  for (const id of sanitized) {
    // skip if we already have it from cache or a previous fetch
    if (map[id]) continue
    let attempts = 0
    const maxAttempts = 3
    let ok = false
    while (!ok && attempts < maxAttempts) {
      attempts += 1
      try {
  const itemUrl = `${baseUrl}/space/item/${id}`
  console.log('API Request:', itemUrl)
  const res = await apiFetch(itemUrl, { headers: { 'Authorization': `Bearer ${token}` } })
        console.log(`fetchItemDetails: id=${id} attempt=${attempts} status=${res.status}`)
        if (res.ok) {
          const data = await res.json()
          const obj = Array.isArray(data) ? data[0] : data
          if (obj) {
            map[id] = obj
            itemCache[id] = obj
          }
          ok = true
          break
        } else if (res.status >= 500) {
          // server error - retry
          await sleep(delayMs * attempts)
        } else {
          // client error or not found - don't retry, but log body for debugging
          try {
            const txt = await res.text()
            console.warn(`fetchItemDetails: id=${id} client error ${res.status}: ${txt}`)
          } catch (e) {
            console.warn(`fetchItemDetails: id=${id} client error ${res.status}`)
          }
          ok = true
          break
        }
      } catch (e) {
        console.warn(`fetchItemDetails error id=${id} attempt=${attempts}`, e)
        await sleep(delayMs * attempts)
      }
    }
    // small pause between requests even on success
    await sleep(delayMs)
  }
  return map
}

const fetchAvailabilityForIds = async (ids, dateStr) => {
  const availBookings = []
  const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))
  const delayMs = 150
  // sanitize ids
  const sanitized = (ids || []).map(id => Number(id)).filter(id => Number.isFinite(id) && id > 0)
  const invalid = (ids || []).filter(id => !sanitized.includes(Number(id)))
  if (invalid.length) console.warn('fetchAvailabilityForIds: skipping invalid ids', invalid)
  for (const id of sanitized) {
    let attempts = 0
    const maxAttempts = 3
    let ok = false
    while (!ok && attempts < maxAttempts) {
      attempts += 1
      try {
  const availUrl = `${baseUrl}/space/availability?lid=${locationId}&date=${dateStr}&eid=${id}`
  console.log('API Request:', availUrl)
  const res = await apiFetch(availUrl, { headers: { 'Authorization': `Bearer ${token}` } })
        console.log(`fetchAvailabilityForIds: id=${id} attempt=${attempts} status=${res.status}`)
        if (!res.ok) {
          if (res.status >= 500) {
            await sleep(delayMs * attempts)
            continue
          }
          try {
            const txt = await res.text()
            console.warn(`fetchAvailabilityForIds: id=${id} client error ${res.status}: ${txt}`)
          } catch (e) {
            console.warn(`fetchAvailabilityForIds: id=${id} client error ${res.status}`)
          }
          break
        }
        const data = await res.json()
        const arr = Array.isArray(data) ? data : (data.availability || data.items || [])
        if (!Array.isArray(arr)) { ok = true; break }
        for (const seg of arr) {
          const status = (seg.status || '').toString().toLowerCase()
          const bookedCandidate = seg.bookId || seg.id || seg.booked || (status && (status.includes('book') || status.includes('unavail') || status.includes('busy') || seg.available === false))
          if (!bookedCandidate) continue
          const from = seg.fromDate || seg.from || seg.start || seg.startDate || seg.start_time || seg.startTime
          const to = seg.toDate || seg.to || seg.end || seg.endDate || seg.end_time || seg.endTime
          if (!from || !to) continue
          availBookings.push({ id: `${id}-${from}-${to}`, eid: Number(id), fromDate: from, toDate: to, status: 'Confirmed', source: 'availability' })
        }
        ok = true
      } catch (e) {
        console.warn(`fetchAvailabilityForIds error id=${id} attempt=${attempts}`, e)
        await sleep(delayMs * attempts)
      }
    }
    await sleep(delayMs)
  }
  return availBookings
}

const enrichRoomsWithAPIData = async (idsParam = null) => {
  const envIds = Array.isArray(roomItemIds) && roomItemIds.length ? roomItemIds.map(Number) : []
  const bookingIds = bookings.value.map(b => Number(b.eid))
  const existingIds = rooms.value.map(r => Number(r.id))
  let idSet
  if (Array.isArray(idsParam) && idsParam.length) {
    idSet = new Set(idsParam.map(Number))
  } else {
    idSet = new Set([ ...envIds, ...bookingIds, ...existingIds ])
  }
  const ids = Array.from(idSet)

  // Fetch item details for these ids
  const itemMap = await fetchItemDetails(ids)

  // If any items missing, try bulk endpoints to locate them
  const missing = ids.filter(id => !itemMap[id])
  if (missing.length) {
    const bulkEndpoints = [
  `${baseUrl}/space/item?lid=${locationId}`,
  `${baseUrl}/space/item`,
      `${baseUrl}/space/locations/${locationId}`,
      `${baseUrl}/space/locations/${locationId}/items`
    ]
    for (const ep of bulkEndpoints) {
      try {
  console.log('API Request:', ep)
  const res = await apiFetch(ep, { headers: { 'Authorization': `Bearer ${token}` } })
        if (!res.ok) continue
        const data = await res.json()
        const arr = Array.isArray(data) ? data : (data.items || data.spaces || data.rooms || [])
        if (!Array.isArray(arr)) continue
        for (const item of arr) {
          const id = item.id || item.eid || item.iid || item.item_id || item.space_id
          if (!id) continue
          const numericId = Number(id)
          if (idSet.has(numericId) && !itemMap[numericId]) {
            itemMap[numericId] = item
            itemCache[numericId] = item
          }
        }
        // refresh missing
        const stillMissing = ids.filter(id => !itemMap[id])
        if (!stillMissing.length) break
      } catch (e) {
        // ignore
      }
    }
  }

  // Build rooms list from itemMap, bookings, and roomMeta fallback
  const newRooms = ids.map(id => {
    const numId = Number(id)
    const item = itemMap[numId]
    const booking = bookings.value.find(b => Number(b.eid) === numId)
    let name = (item && (item.name || item.item_name)) || (booking && booking.item_name) || `Room ${numId}`
    if (!name || String(name).trim() === '') name = `Room ${numId}`
    const zone = (item && (item.zone || item.zone_name || item.location_name)) || (booking && booking.category_name) || inferZoneFromName(name)
      const capacity = (item && item.capacity != null) ? Number(item.capacity) : (roomMeta && roomMeta[numId] != null ? Number(roomMeta[numId]) : null)
      return { id: numId, name, zone, capacity }
  })
    // Ensure ids and capacities are normalized to numbers
    rooms.value = newRooms.map(r => ({ ...r, id: Number(r.id), capacity: r.capacity != null ? Number(r.capacity) : null }))
  console.log('Room metadata updated from API/items endpoint; items fetched:', Object.keys(itemMap).length)
}

const getOpenRange = (date) => {
  // Default weekday hours: 8:00 - 23:00
  // Weekend hours default to 10:00 - 18:00 (adjust if your library differs)
  const day = date.getDay() // 0 = Sunday, 6 = Saturday
  if (day === 0 || day === 6) {
    return { openStart: 10 * 60, openEnd: 18 * 60 }
  }
  return { openStart: 8 * 60, openEnd: 23 * 60 }
}

const getTotalBookedMinutes = (roomId) => {
  const roomBookings = getBookingsForRoom(roomId)
  const { openStart, openEnd } = getOpenRange(currentDate.value)
  return roomBookings.reduce((total, booking) => {
    const start = new Date(booking.fromDate)
    const end = new Date(booking.toDate)
    const startMinutes = start.getHours() * 60 + start.getMinutes()
    const endMinutes = end.getHours() * 60 + end.getMinutes()
    const adjStart = Math.max(startMinutes, openStart)
    const adjEnd = Math.min(endMinutes, openEnd)
    if (adjStart >= adjEnd) return total
    return total + (adjEnd - adjStart)
  }, 0)
}

const getAvailabilityPercent = (roomId) => {
  const { openStart, openEnd } = getOpenRange(currentDate.value)
  const totalOpen = Math.max(0, openEnd - openStart)
  if (totalOpen === 0) return 0
  const booked = getTotalBookedMinutes(roomId)
  const avail = Math.max(0, totalOpen - booked)
  return (avail / totalOpen) * 100
}

const sortedRooms = computed(() => {
  return rooms.value.slice().sort((a, b) => {
    const availA = getAvailabilityPercent(a.id)
    const availB = getAvailabilityPercent(b.id)
    const diff = availB - availA
    if (Math.abs(diff) > 0.0001) return diff
  // Use natural numeric-aware comparison for names (e.g., LLSR 2 before LLSR 10)
  const nameCmp = (a.name || '').localeCompare(b.name || '', undefined, { numeric: true, sensitivity: 'base' })
  if (nameCmp !== 0) return nameCmp
  return a.id - b.id
  })
})

const getBookingStyle = (booking) => {
  const start = new Date(booking.fromDate)
  const end = new Date(booking.toDate)
  const startMinutes = start.getHours() * 60 + start.getMinutes()
  const endMinutes = end.getHours() * 60 + end.getMinutes()
  const openStart = 8 * 60
  const openEnd = 23 * 60
  const totalMinutes = 15 * 60
  const adjStart = Math.max(startMinutes, openStart)
  const adjEnd = Math.min(endMinutes, openEnd)
  if (adjStart >= adjEnd) return { display: 'none' }
  const left = ((adjStart - openStart) / totalMinutes) * 100
  const width = ((adjEnd - adjStart) / totalMinutes) * 100
  return {
    left: `${left}%`,
    width: `${width}%`
  }
}

const formatTime = (isoString) => {
  const date = new Date(isoString)
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
}

const formatHalfHour = (i) => {
  const hour = 8 + Math.floor(i / 2)
  const min = (i % 2) * 30
  const displayHour = hour % 12 || 12
  const ampm = hour >= 12 ? 'PM' : 'AM'
  return `${displayHour}:${min.toString().padStart(2, '0')} ${ampm}`
}

const isToday = computed(() => currentDate.value.toDateString() === new Date().toDateString())

const formatDate = (date) => date.toLocaleDateString()

const goToTomorrow = () => {
  currentDate.value = new Date(currentDate.value.getTime() + 24 * 60 * 60 * 1000)
  fetchBookings()
}

const goToToday = () => {
  currentDate.value = new Date()
  fetchBookings()
}

// Filter state and derived lists
const zones = computed(() => {
  const set = new Set(rooms.value.map(r => r.zone).filter(Boolean))
  const allZones = Array.from(set)
  const preferred = ['Lower Level', 'First Floor', 'Second Floor', 'Third Floor']
  const sorted = allZones.sort((a, b) => {
    const ai = preferred.findIndex(p => p.toLowerCase() === a.toLowerCase())
    const bi = preferred.findIndex(p => p.toLowerCase() === b.toLowerCase())
    if (ai === -1 && bi === -1) return a.localeCompare(b)
    if (ai === -1) return 1
    if (bi === -1) return -1
    return ai - bi
  })
  return ['All', ...sorted]
})

const roomNames = computed(() => {
  let list = rooms.value
  if (selectedZone.value !== 'All') list = list.filter(r => r.zone === selectedZone.value)
  const set = new Set(list.map(r => r.name))
  return ['All', ...Array.from(set).sort((a, b) => a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' }))]
})

// Reset room filter if it doesn't exist for the selected zone
watch(selectedZone, () => {
  if (selectedRoomName.value !== 'All' && !roomNames.value.includes(selectedRoomName.value)) {
    selectedRoomName.value = 'All'
  }
})

const filteredSortedRooms = computed(() => {
  const filtered = rooms.value.filter(r => {
    if (selectedZone.value !== 'All' && r.zone !== selectedZone.value) return false
    if (selectedRoomName.value !== 'All' && r.name !== selectedRoomName.value) return false
    if (selectedCapacity.value !== 'All') {
      const minCap = Number(selectedCapacity.value)
      if (!r.capacity || Number(r.capacity) < minCap) return false
    }
    return true
  })
  return filtered.slice().sort((a, b) => {
    const availDiff = getAvailabilityPercent(b.id) - getAvailabilityPercent(a.id)
    if (Math.abs(availDiff) > 0.0001) return availDiff
    const nameCmp = (a.name || '').localeCompare(b.name || '', undefined, { numeric: true, sensitivity: 'base' })
    if (nameCmp !== 0) return nameCmp
    return a.id - b.id
  })
})

const EXPECTED_CAPACITIES = [1, 2, 4]
const capacities = computed(() => {
  // Only expose the expected capacities (1,2,4) in the dropdown so sample or incorrect
  // values (e.g. 6,8) from the mapping file won't appear.
  const nums = rooms.value.map(r => Number(r.capacity)).filter(c => Number.isFinite(c))
  const uniq = Array.from(new Set(nums))
  const filtered = uniq.filter(n => EXPECTED_CAPACITIES.includes(n)).sort((a, b) => a - b)
  // If there are no mapped capacities, fall back to the expected list so the dropdown still shows options
  return filtered.length ? filtered : EXPECTED_CAPACITIES
})

const toggleExpanded = (id) => {
  const index = expandedRooms.value.indexOf(id)
  if (index > -1) {
    expandedRooms.value.splice(index, 1)
  } else {
    expandedRooms.value.push(id)
  }
}

onMounted(async () => {
  try {
    await fetchBookings()
  } finally {
    loading.value = false
  }
})
</script>

<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.header {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
  margin-bottom: 20px;
}

.header button {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.header button:hover {
  background-color: #0056b3;
}

.filters {
  display: flex;
  justify-content: center;
  gap: 16px;
  margin-bottom: 12px;
  align-items: center;
}

.filters label {
  font-weight: 600;
}

.filters select {
  margin-left: 8px;
  padding: 6px 8px;
  border-radius: 4px;
  border: 1px solid #ccc;
}

.no-rooms {
  font-style: italic;
  color: #666;
  margin: 8px;
}

.rooms {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

.room-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 20px 16px 16px 16px;
  margin: 16px;
  width: 300px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  transition: transform 0.2s, width 0.3s, height 0.3s;
  cursor: pointer;
  position: relative;
}

.room-card:hover {
  transform: scale(1.05);
}

.room-card.expanded {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 80%;
  height: 80%;
  z-index: 1001;
  margin: 0;
  background: white;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.backdrop {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.5);
  z-index: 1000;
}

.timeline {
  position: relative;
  height: 20px;
  background-color: green;
  border-radius: 10px;
  margin-top: 8px;
}

.room-card.expanded .timeline {
  height: 40px;
  width: 80%;
}

.booking-segment {
  position: absolute;
  top: 0;
  height: 100%;
  background-color: red;
  border-radius: 10px;
}

.time-label {
  position: absolute;
  top: 22px;
  font-size: 10px;
  color: #333;
  opacity: 0;
  transition: opacity 0.2s;
  pointer-events: none;
  white-space: nowrap;
}

.room-card:hover .time-label,
.room-card.expanded .time-label {
  opacity: 1;
  font-weight: bold;
}

.room-card.expanded h3 {
  font-size: 24px;
}

.room-card.expanded .time-label {
  top: 42px;
  font-size: 10px;
}

.room-meta {
  font-size: 13px;
  color: #555;
  margin-top: 6px;
}
</style>