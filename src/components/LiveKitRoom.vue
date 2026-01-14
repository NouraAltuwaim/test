<template>
  <div class="room-root">
    <!-- VIDEO CALL LAYOUT -->
    <div v-if="isVideoCall" class="videos">
      <video ref="remoteVideo" autoplay playsinline class="video remote"></video>
      <video ref="localVideo" autoplay playsinline muted class="video local"></video>
 
      <div v-if="remoteName" class="remote-label">
        {{ remoteName }}
      </div>
    </div>
 
    <!-- VOICE CALL LAYOUT -->
    <div v-else class="voice-layout" :class="{ pip: isPip }">
  <div class="voice-center">
    <img
      src="../assets/call1.png"
      alt="Call avatar"
      class="voice-avatar"
    />
 
    <div v-if="remoteName" class="voice-name">
      {{ remoteName }}
    </div>
  </div>
</div>
 
  </div>
</template>
 
<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import { Room, RoomEvent, Track, createLocalVideoTrack, createLocalAudioTrack } from 'livekit-client'
 
const room = ref(null)
const connected = ref(false)
const isMuted = ref(false)
const cameraOff = ref(false)
const roomName = ref('')
 
const remoteVideo = ref(null)
const localVideo = ref(null)
 
const isVideoCall = ref(false)
const remoteName = ref('')
 
const isPip = ref(false)
 
function updatePipState() {
  const w = window.innerWidth
  const h = window.innerHeight
 
  isPip.value = w < 400 || h < 300
}
 
window.addEventListener('resize', updatePipState)
 
 
function detachAllTracks(videoEl) {
  if (!videoEl) return
  try {
    videoEl.srcObject = null
  } catch (e) {
    console.error('Failed to detach tracks:', e)
  }
}
 
function updateRemoteNameFromParticipant(participant) {
  if (!participant) return
 
  // Only show real display names, never IDs
  const name = participant.name
 
  if (name && name.trim().length > 0) {
    remoteName.value = name.trim()
  } else {
    remoteName.value = ''
  }
}
 
 
/**
 * Force layout recalculation in JCEF when Swing resizes.
 * Works well for PiP toggle/maximize.
 */
function forceReflow() {
  try {
    const el = document.documentElement
    el.style.display = 'none'
    // eslint-disable-next-line no-unused-expressions
    el.offsetHeight
    el.style.display = ''
  } catch (e) {
    // ignore
  }
}
 
async function handleStartCall(payload) {
  console.log('[Vue] handleStartCall payload:', payload)
  const { wsUrl, token, roomName: rn, video } = payload || {}
 
  roomName.value = rn || ''
  isVideoCall.value = !!video
  remoteName.value = ''
 
  if (room.value) {
    try {
      await room.value.disconnect()
    } catch (e) {
      console.warn('Error disconnecting previous room:', e)
    }
    room.value = null
  }
 
  try {
    const newRoom = new Room({
      adaptiveStream: true,
      dynacast: true
    })
 
    newRoom.on(RoomEvent.ParticipantDisconnected, (participant) => {
      console.log('[Vue] ParticipantDisconnected:', participant?.identity)
 
      window.javaBridge?.sendToJava?.('REMOTE_LEFT', {
        identity: participant?.identity || '',
        roomName: roomName.value || ''
      })
 
      detachAllTracks(remoteVideo.value)
    })
 
    newRoom.on(RoomEvent.ParticipantConnected, (participant) => {
      console.log('[Vue] ParticipantConnected:', participant?.identity)
      updateRemoteNameFromParticipant(participant)
    })
 
    newRoom.on(RoomEvent.TrackSubscribed, (track, pub, participant) => {
      console.log('[Vue] Subscribed:', track.kind, 'from', participant.identity)
 
      updateRemoteNameFromParticipant(participant)
 
      if (track.kind === Track.Kind.Audio) {
        track.attach()
      }
 
      if (track.kind === Track.Kind.Video && remoteVideo.value) {
        track.attach(remoteVideo.value)
        forceReflow()
      }
    })
 
    newRoom.on(RoomEvent.LocalTrackPublished, (publication) => {
      if (publication.kind === Track.Kind.Video && localVideo.value) {
        const track = publication.track
        if (track) track.attach(localVideo.value)
        forceReflow()
      }
    })
 
    newRoom.on(RoomEvent.Disconnected, () => {
      console.log('[Vue] Room disconnected')
      connected.value = false
 
      window.javaBridge?.sendToJava?.('ROOM_DISCONNECTED', {
        roomName: roomName.value || ''
      })
 
      detachAllTracks(remoteVideo.value)
      detachAllTracks(localVideo.value)
    })
 
    console.log('[Vue] Connecting to LiveKit:', wsUrl)
    await newRoom.connect(wsUrl, token)
 
    connected.value = true
    room.value = newRoom
 
    // publish mic
    try {
      const micTrack = await createLocalAudioTrack()
      await newRoom.localParticipant.publishTrack(micTrack)
      isMuted.value = false
    } catch (e) {
      console.error('❌ Failed to create/publish audio track', e)
    }
 
    // publish camera only for video calls
    if (video) {
      try {
        const camTrack = await createLocalVideoTrack({
          resolution: { width: 1280, height: 720 },
          frameRate: 30
        })
 
        await newRoom.localParticipant.publishTrack(camTrack)
        cameraOff.value = false
 
        if (localVideo.value) camTrack.attach(localVideo.value)
      } catch (e) {
        console.error('❌ Failed to create/publish video track', e)
        cameraOff.value = true
      }
    } else {
      cameraOff.value = true
    }
 
    window.javaBridge?.sendToJava?.('callConnected', {
      roomName: roomName.value
    })
 
    forceReflow()
  } catch (e) {
    console.error('[Vue] Failed to connect to LiveKit:', e)
    connected.value = false
    window.javaBridge?.sendToJava?.('callError', {
      message: e?.message || 'Unknown error'
    })
  }
}
 
async function handleToggleMute(payload) {
  if (!room.value) return
  const mute = !!payload?.mute
 
  try {
    const audioPub =
      room.value.localParticipant.audioTrackPublications.values().next().value
 
    if (!audioPub?.track) return
 
    if (mute) await audioPub.track.mute()
    else await audioPub.track.unmute()
 
    isMuted.value = mute
  } catch (e) {
    console.error('[Vue] Failed to toggle mute:', e)
  }
}
 
async function handleToggleCamera(payload) {
  if (!room.value) return
  const off = !!payload?.off
 
  try {
    const videoPub =
      room.value.localParticipant.videoTrackPublications.values().next().value
 
    if (off) await videoPub?.track?.mute()
    else await videoPub?.track?.unmute()
 
    cameraOff.value = off
    forceReflow()
  } catch (e) {
    console.error('[Vue] Failed to toggle camera:', e)
  }
}
 
async function handleEndCall() {
  if (!room.value) return
  try {
    await room.value.disconnect()
    room.value = null
    connected.value = false
    detachAllTracks(remoteVideo.value)
    detachAllTracks(localVideo.value)
  } catch (e) {
    console.error('[Vue] Failed to end call:', e)
  }
}
 
function handleResize() {
  forceReflow()
}
 
onMounted(async () => {
  window.addEventListener('resize', handleResize)
 
  function registerBridgeListeners(attempt = 1) {
    if (!window.javaBridge || typeof window.javaBridge.on !== 'function') {
      if (attempt < 30) setTimeout(() => registerBridgeListeners(attempt + 1), 200)
      return
    }
 
    window.javaBridge.on('startCall', handleStartCall)
    window.javaBridge.on('toggleMute', handleToggleMute)
    window.javaBridge.on('toggleCamera', handleToggleCamera)
    window.javaBridge.on('endCall', handleEndCall)
 
    window.javaBridge.sendToJava?.('vueReady', {})
    updatePipState()
 
  }
 
  registerBridgeListeners()
})
 
onBeforeUnmount(async () => {
  window.removeEventListener('resize', handleResize)
 
  if (room.value) {
    try {
      await room.value.disconnect()
    } catch (e) {
      console.warn('Error disconnecting on unmount:', e)
    }
  }
})
</script>
 
<style scoped>
.room-root {
  position: relative;
  width: 100%;
  height: 100%;
  background: #000;
  overflow: hidden;
}
 
/* ================= VIDEO ================= */
 
.videos {
  position: absolute;
  inset: 0;
  background: #000;
}
 
/* remote fills container and stays inside (PiP safe) */
.remote {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  background: #000;
  object-fit: contain; /* KEY for PiP */
}
 
/* local preview scales down in PiP */
.local {
  position: absolute;
  right: 12px;
  bottom: 12px;
 
  width: min(30vw, 220px);
  aspect-ratio: 16 / 9;
  height: auto;
 
  border-radius: 8px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  box-shadow: 0 0 12px rgba(0, 0, 0, 0.7);
  object-fit: cover;
}
 
/* Remote name overlay */
.remote-label {
  position: absolute;
  left: 12px;
  top: 12px;
  padding: 6px 12px;
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.6);
  color: #f9fafb;
  font-size: 14px;
  letter-spacing: 0.02em;
}
 
/* ================= VOICE ================= */
 
.voice-layout {
  position: absolute;
  inset: 0;
  background: radial-gradient(circle at top, #1e293b 0%, #020617 60%, #000 100%);
  display: flex;
  align-items: center;
  justify-content: center;
}
 
.voice-center {
  text-align: center;
  color: #e5e7eb;
  display: flex;
  flex-direction: column;
  align-items: center;
}
 
/* ===== DEFAULT (MAXIMIZED) ===== */
 
.voice-avatar {
  width: 180px;
  height: 180px;
  border-radius: 50%;
  object-fit: cover;
  background: #1f2937;
  box-shadow:
    0 0 0 12px rgba(255, 255, 255, 0.03),
    0 0 40px rgba(0, 0, 0, 0.6);
  margin-bottom: 18px;
}
 
.voice-name {
  font-size: 18px;
  font-weight: 500;
  letter-spacing: 0.03em;
}
 
/* ===== PiP MODE ===== */
 
.voice-layout.pip .voice-avatar {
  width: 96px;
  height: 96px;
  margin-bottom: 8px;
  box-shadow:
    0 0 0 6px rgba(255, 255, 255, 0.04);
}
 
.voice-layout.pip .voice-name {
  font-size: 13px;
  opacity: 0.85;
}
</style>
