<template>
  <div class="video-call-container">
    <div class="video-grid">
      <div class="video-item local-video">
        <video ref="localVideo" autoplay muted></video>
        <div class="user-name">You ({{ userName }})</div>
      </div>
      <div 
        v-for="peer in peers" 
        :key="peer.id" 
        class="video-item remote-video"
      >
        <video :id="'video-' + peer.id" autoplay></video>
        <div class="user-name">{{ peer.name }}</div>
      </div>
    </div>

    <div class="controls">
      <button @click="toggleVideo" :class="{ 'active': isVideoEnabled }">
        <span v-if="isVideoEnabled">Stop Video</span>
        <span v-else>Start Video</span>
      </button>
      <button @click="toggleAudio" :class="{ 'active': isAudioEnabled }">
        <span v-if="isAudioEnabled">Mute</span>
        <span v-else>Unmute</span>
      </button>
      <button @click="endCall" class="end-call">End Call</button>
    </div>
  </div>
</template>

<script>
import { io } from 'socket.io-client';

export default {
  name: 'VideoCall',
  data() {
    return {
      socket: null,
      localStream: null,
      peerConnections: {},
      peers: [],
      userName: '',
      isVideoEnabled: true,
      isAudioEnabled: true,
      configuration: {
        iceServers: [
          { urls: 'stun:stun.l.google.com:19302' },
          { urls: 'stun:stun1.l.google.com:19302' }
        ]
      }
    };
  },
  async mounted() {
    // Check if user has a name
    this.userName = sessionStorage.getItem('userName');
    if (!this.userName) {
      this.$router.push('/');
      return;
    }

    try {
      // Get local media stream
      this.localStream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true
      });
      
      // Display local video
      this.$refs.localVideo.srcObject = this.localStream;
      
      // Connect to signaling server
      this.connectToServer();
    } catch (error) {
      console.error('Error accessing media devices:', error);
      alert('Failed to access camera and microphone. Please ensure they are connected and permissions are granted.');
    }
  },
  beforeUnmount() {
    // Clean up resources
    this.endCall();
  },
  methods: {
    connectToServer() {
      // Get the server URL from environment or use the current hostname
      const serverUrl = process.env.VUE_APP_SERVER_URL || 
        `${window.location.protocol}//${window.location.hostname}:5000`;
      console.log('Connecting to server:', serverUrl);
      
      // Connect to the signaling server
      this.socket = io(serverUrl);
      
      // Join the call
      this.socket.emit('join', { name: this.userName });
      
      // Handle new user connected
      this.socket.on('user-connected', (userData) => {
        console.log('New user connected:', userData.name);
        this.createPeerConnection(userData);
        this.peers.push(userData);
      });
      
      // Get existing users
      this.socket.on('get-users', (users) => {
        const otherUsers = users.filter(user => user.id !== this.socket.id);
        this.peers = otherUsers;
        
        // Create peer connections with existing users
        otherUsers.forEach(user => {
          this.createPeerConnection(user);
          this.createOffer(user.id);
        });
      });
      
      // Handle user disconnected
      this.socket.on('user-disconnected', (userId) => {
        console.log('User disconnected:', userId);
        
        if (this.peerConnections[userId]) {
          this.peerConnections[userId].close();
          delete this.peerConnections[userId];
        }
        
        this.peers = this.peers.filter(peer => peer.id !== userId);
      });
      
      // Handle signaling events
      this.socket.on('offer', async (data) => {
        await this.handleOffer(data);
      });
      
      this.socket.on('answer', (data) => {
        this.handleAnswer(data);
      });
      
      this.socket.on('ice-candidate', (data) => {
        this.handleIceCandidate(data);
      });
    },
    
    createPeerConnection(user) {
      try {
        const peerConnection = new RTCPeerConnection(this.configuration);
        
        // Add local tracks to the peer connection
        this.localStream.getTracks().forEach(track => {
          peerConnection.addTrack(track, this.localStream);
        });
        
        // Handle ICE candidate events
        peerConnection.onicecandidate = (event) => {
          if (event.candidate) {
            this.socket.emit('ice-candidate', {
              target: user.id,
              candidate: event.candidate
            });
          }
        };
        
        // Handle track events
        peerConnection.ontrack = (event) => {
          const stream = event.streams[0];
          const videoElement = document.getElementById(`video-${user.id}`);
          
          if (videoElement) {
            videoElement.srcObject = stream;
          }
        };
        
        this.peerConnections[user.id] = peerConnection;
        return peerConnection;
      } catch (error) {
        console.error('Error creating peer connection:', error);
        return null;
      }
    },
    
    async createOffer(userId) {
      try {
        const peerConnection = this.peerConnections[userId];
        const offer = await peerConnection.createOffer();
        await peerConnection.setLocalDescription(offer);
        
        this.socket.emit('offer', {
          target: userId,
          offer: peerConnection.localDescription
        });
      } catch (error) {
        console.error('Error creating offer:', error);
      }
    },
    
    async handleOffer(data) {
      try {
        const peerId = data.caller;
        let peerConnection = this.peerConnections[peerId];
        
        if (!peerConnection) {
          const user = { id: peerId, name: data.name };
          this.peers.push(user);
          peerConnection = this.createPeerConnection(user);
        }
        
        await peerConnection.setRemoteDescription(new RTCSessionDescription(data.offer));
        const answer = await peerConnection.createAnswer();
        await peerConnection.setLocalDescription(answer);
        
        this.socket.emit('answer', {
          target: peerId,
          answer: peerConnection.localDescription
        });
      } catch (error) {
        console.error('Error handling offer:', error);
      }
    },
    
    async handleAnswer(data) {
      try {
        const peerConnection = this.peerConnections[data.caller];
        if (peerConnection) {
          await peerConnection.setRemoteDescription(new RTCSessionDescription(data.answer));
        }
      } catch (error) {
        console.error('Error handling answer:', error);
      }
    },
    
    async handleIceCandidate(data) {
      try {
        const peerConnection = this.peerConnections[data.caller];
        if (peerConnection) {
          await peerConnection.addIceCandidate(new RTCIceCandidate(data.candidate));
        }
      } catch (error) {
        console.error('Error handling ICE candidate:', error);
      }
    },
    
    toggleVideo() {
      if (this.localStream) {
        const videoTrack = this.localStream.getVideoTracks()[0];
        if (videoTrack) {
          videoTrack.enabled = !videoTrack.enabled;
          this.isVideoEnabled = videoTrack.enabled;
        }
      }
    },
    
    toggleAudio() {
      if (this.localStream) {
        const audioTrack = this.localStream.getAudioTracks()[0];
        if (audioTrack) {
          audioTrack.enabled = !audioTrack.enabled;
          this.isAudioEnabled = audioTrack.enabled;
        }
      }
    },
    
    endCall() {
      // Stop all local tracks
      if (this.localStream) {
        this.localStream.getTracks().forEach(track => track.stop());
      }
      
      // Close all peer connections
      Object.values(this.peerConnections).forEach(pc => pc.close());
      this.peerConnections = {};
      
      // Disconnect from the server
      if (this.socket) {
        this.socket.disconnect();
      }
      
      // Navigate back to home
      this.$router.push('/');
    }
  }
};
</script>

<style scoped>
.video-call-container {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 150px);
}

.video-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
  flex: 1;
  overflow-y: auto;
  padding: 20px 0;
}

.video-item {
  position: relative;
  border-radius: 12px;
  overflow: hidden;
  aspect-ratio: 16/9;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  background-color: #3c4043;
}

.user-name {
  position: absolute;
  bottom: 10px;
  left: 10px;
  background-color: rgba(0, 0, 0, 0.5);
  color: white;
  padding: 5px 10px;
  border-radius: 4px;
  font-size: 0.9rem;
}

.controls {
  display: flex;
  justify-content: center;
  gap: 20px;
  padding: 20px 0;
}

.controls button {
  background-color: #4285f4;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 50px;
  cursor: pointer;
  font-size: 1rem;
  min-width: 120px;
}

.controls button.active {
  background-color: #3367d6;
}

.controls button.end-call {
  background-color: #ea4335;
}

.controls button:hover {
  opacity: 0.9;
}
</style>
