<template>
  <div class="booking-container">
    <form @submit.prevent="submitBooking" class="booking-form">
      <h4>Book this room from {{ startLabel }} to {{ endLabel }}</h4>
      <input v-model="fname" type="text" placeholder="First Name" required />
      <input v-model="lname" type="text" placeholder="Last Name" required />
      <input v-model="email" type="email" placeholder="URI.edu Email" required pattern=".*@uri\.edu$" />
      <div class="terms-checkbox">
        <input type="checkbox" :id="`terms-${roomId}`" v-model="termsAccepted" required />
        <label :for="`terms-${roomId}`">I have read and agree to the Terms and Conditions</label>
      </div>
      <button type="submit" :disabled="!termsAccepted">Book Now</button>
    </form>
    <div class="terms-box">
      <h4>Terms and Conditions</h4>
      <p><strong>Once you reserve a room, you will receive a confirmation email with a link and a confirmation code. You must check-in to the room upon arrival. If you do not check in within 15 minutes of arriving for your reservation, your reservation will be cancelled.</strong></p>
      <ol>
        <li>No curtains nor rolling white boards shall obstruct windows nor block the doors of the study room. No papers should be taped to the walls.</li>
        <li>No Library materials, including books, should be left in any Group Study Room.</li>
        <li>The Library is not responsible for loss, theft, or damage of any material left in the room.</li>
        <li>No electrical appliances are allowed in the Group Study Rooms.</li>
        <li>The individual to whom the Group Study Room is assigned is responsible for any infraction of the policies or damage to the room. Any damage must be reported to Circulation as soon as possible.</li>
        <li>The <a href="https://web.uri.edu/library/wp-content/uploads/sites/1549/Patron_Code_of_Conduct_2023.pdf" target="_blank">Library Patron Code of Conduct</a> will apply to all users of the Group Study Rooms.</li>
      </ol>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const props = defineProps({
  roomId: {
    type: [Number, String],
    required: true,
  },
  startLabel: {
    type: String,
    required: true,
  },
  endLabel: {
    type: String,
    required: true,
  },
})

const emit = defineEmits(['submit'])

const fname = ref('')
const lname = ref('')
const email = ref('')
const termsAccepted = ref(false)

const submitBooking = () => {
  emit('submit', {
    fname: fname.value,
    lname: lname.value,
    email: email.value,
    termsAccepted: termsAccepted.value,
  })
}
</script>