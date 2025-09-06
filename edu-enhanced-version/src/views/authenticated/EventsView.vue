<script setup>
import { ref, onMounted } from 'vue'
import { db, auth } from '@/config/firebaseConfig'
import { 
    collection, 
    addDoc, 
    getDocs, 
    query, 
    orderBy, 
    serverTimestamp, 
    doc, 
    getDoc 
} from 'firebase/firestore'
import { toast } from 'vue3-toastify'
import { onAuthStateChanged } from 'firebase/auth'
import EventCard from '@/components/ui/events/EventCard.vue'

const currentUser = ref(null)
const userDetails = ref(null)

onMounted(() => {
  onAuthStateChanged(auth, async (user) => {
    if (user) {
      currentUser.value = user

      const docRef = doc(db, 'users', user.uid)
      const docSnap = await getDoc(docRef)
      if (docSnap.exists()) {
        userDetails.value = docSnap.data()
      }
    } else {
      currentUser.value = null
      userDetails.value = null
    }
  })
})

const eventTitle = ref('')
const eventDesc = ref('')
const eventLocation = ref('')
const eventDate = ref('')
const eventStartTime = ref('')
const eventEndTime = ref('')

const isSubmitting = ref(false)
const isAddEventShown = ref(false)

const events = ref([])
const loading = ref(true)

const toggleCreateEvent = () => {
  isAddEventShown.value = !isAddEventShown.value
}

const fetchEvents = async () => {
  try {
    const q = query(collection(db, 'events'), orderBy('createdAt', 'desc'))
    const snapshot = await getDocs(q)
    events.value = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }))
  } catch (err) {
    toast.error('Failed to load events')
    console.error(err)
  } finally {
    loading.value = false
  }
}
const createEvent = async () => {
  // Trimmed values for validation
  const title = eventTitle.value.trim()
  const desc = eventDesc.value.trim()
  const location = eventLocation.value.trim()
  const date = eventDate.value.trim()
  const startTime = eventStartTime.value.trim()
  const endTime = eventEndTime.value.trim()

  if (!title || !desc || !location || !date || !startTime || !endTime) {
    toast.error('Please fill in all required fields (no empty spaces)')
    return
  }

  if (!currentUser.value) {
    toast.error('You must be logged in to create an event')
    return
  }

  const today = new Date()
  today.setHours(0, 0, 0, 0)

  const selectedDate = new Date(date)
  if (selectedDate < today) {
    toast.error('Event date cannot be in the past')
    return
  }

  // ⏰ Time validation
  const startDateTime = new Date(`${date}T${startTime}`)
  const endDateTime = new Date(`${date}T${endTime}`)

  if (endDateTime <= startDateTime) {
    toast.error('End time must be later than start time')
    return
  }

  isSubmitting.value = true

  try {
    await addDoc(collection(db, 'events'), {
      title,
      description: desc,
      location,
      date,
      startTime,
      endTime,
      createdAt: serverTimestamp(),
      uid: currentUser.value.uid,
      user: {
        email: currentUser.value.email,
        firstName: userDetails.value?.firstName ?? '',
        lastName: userDetails.value?.lastName ?? '',
        displayName: userDetails.value?.displayName ?? currentUser.value.displayName ?? '',
        photoUrl: userDetails.value?.photoUrl ?? currentUser.value.photoURL ?? '',
      },
    })

    toast.success('Event created successfully')
    isAddEventShown.value = false

    // reset form
    eventTitle.value = ''
    eventDesc.value = ''
    eventLocation.value = ''
    eventDate.value = ''
    eventStartTime.value = ''
    eventEndTime.value = ''

    await fetchEvents()
  } catch (err) {
    toast.error('Failed to create event, please try again')
    console.error(err)
  } finally {
    isSubmitting.value = false
  }
}


onMounted(() => {
  fetchEvents()
})
</script>


<template>
  <div class="p-6">
    <!-- Header with add event button -->
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-2xl font-bold">Explore Events</h1>
      <button
        @click="toggleCreateEvent"
        class="bg-indigo-600 text-white px-4 py-2 rounded-md hover:bg-indigo-700"
      >
        Add Event
      </button>
    </div>

    <div class="space-y-6 w-full">
      <div v-if="loading" class="col-span-full text-center">Loading events...</div>
      <EventCard v-for="event in events" :key="event.id" :post="event" @deleted="events = events.filter(e => e.id !== $event)" @edit="openEditModal"/>
    </div>

    <div
        v-if="isAddEventShown"
        class="fixed inset-0 flex items-center justify-center bg-black/40 backdrop-blur-sm z-50"
    >

      <div class="bg-white rounded-lg shadow-lg max-w-md w-full p-6 relative">
            <button @click="isAddEventShown = false" class="absolute top-4 right-4 text-gray-500 hover:text-gray-800" aria-label="Close modal">
                ✕
            </button>

        <h2 class="text-xl font-semibold mb-4">Create New Event</h2>

        <form @submit.prevent="createEvent" class="space-y-4">
          <div>
            <label class="block font-medium mb-1" for="title">Title *</label>
            <input
              id="title"
              v-model="eventTitle"
              type="text"
              class="w-full border rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
              required
            />
          </div>

          <div>
            <label class="block font-medium mb-1" for="description">Description *</label>
            <textarea
              id="description"
              v-model="eventDesc"
              rows="3"
              class="w-full border rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
              required
            ></textarea>
          </div>

          <div>
            <label class="block font-medium mb-1" for="location">Location *</label>
            <input
              id="location"
              v-model="eventLocation"
              type="text"
              class="w-full border rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
            />
          </div>

          <div class="flex-1">
              <label class="block font-medium mb-1" for="date">Date *</label>
              <input
                id="date"
                v-model="eventDate"
                type="date"
                class="w-full border rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
                required
              />
            </div>

          <div class="flex space-x-4">
            <div class="flex-1">
              <label class="block font-medium mb-1" for="time"> Start Time *</label>
              <input
                id="time"
                v-model="eventStartTime"
                type="time"
                class="w-full border rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
                required
              />
            </div>
            <div class="flex-1">
              <label class="block font-medium mb-1" for="time"> End Time *</label>
              <input
                id="time"
                v-model="eventEndTime"
                type="time"
                class="w-full border rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
                required
              />
            </div>
          </div>

          <button
            type="submit"
            :disabled="isSubmitting"
            class="w-full bg-indigo-600 text-white py-2 rounded-md hover:bg-indigo-700 disabled:bg-indigo-400"
          >
            {{ isSubmitting ? 'Creating...' : 'Create Event' }}
          </button>
        </form>
      </div>
    </div>
  </div>
</template>
