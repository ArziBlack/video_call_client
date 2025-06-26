<template>
  <div class="home-container">
    <div class="welcome-card">
      <h2>Welcome to Video Call App</h2>
      <p>Enter your name to join a video call session</p>
      <div class="form-group">
        <label for="username">Your Name:</label>
        <input 
          type="text" 
          id="username" 
          v-model="userName" 
          placeholder="Enter your name..."
          @keyup.enter="joinCall"
        />
      </div>
      <button @click="joinCall" :disabled="!userName">Join Call</button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'HomeView',
  data() {
    return {
      userName: ''
    }
  },
  methods: {
    joinCall() {
      if (this.userName.trim()) {
        // Store username in sessionStorage for persistence
        sessionStorage.setItem('userName', this.userName);
        this.$router.push('/call');
      }
    }
  }
}
</script>

<style scoped>
.home-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 80vh;
}

.welcome-card {
  background-color: white;
  border-radius: 10px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  padding: 2rem;
  width: 100%;
  max-width: 500px;
  text-align: center;
}

h2 {
  margin-bottom: 1rem;
  color: #4285f4;
}

p {
  margin-bottom: 2rem;
  color: #666;
}

.form-group {
  margin-bottom: 1.5rem;
  text-align: left;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: bold;
}

input {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
}

button {
  background-color: #4285f4;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #3367d6;
}

button:disabled {
  background-color: #b3b3b3;
  cursor: not-allowed;
}
</style>
