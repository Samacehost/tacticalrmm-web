<template>
  <q-card style="width: 900px; max-width: 95vw;">
    <q-card-section class="row items-center q-pb-none bg-primary text-white">
      <div class="text-h6">
        <q-icon name="security" size="md" class="q-mr-sm" />
        Authorized Selected-Tab Co-Browsing
      </div>
      <q-space />
      <q-btn icon="close" flat round dense v-close-popup @click="endSession" />
    </q-card-section>

    <!-- Client Binding Status Disclaimer Banner -->
    <q-banner class="bg-warning text-dark q-ma-md rounded-borders">
      <template v-slot:avatar>
        <q-icon name="warning" color="dark" />
      </template>
      <strong>Client Binding Status: Unverified browser extension client.</strong>
      <div>Device attestation and unattended access are strictly disabled. Fresh visible client pairing and explicit dual consent are required.</div>
    </q-banner>

    <q-card-section>
      <!-- SECTION 1: REQUEST SESSION -->
      <div v-if="sessionState === 'Idle'" class="text-center q-pa-md">
        <div class="text-subtitle1 q-mb-md">Target Agent ID: <code>{{ agentId }}</code></div>
        <q-btn
          color="primary"
          icon="add_link"
          label="Request Co-Browsing Session"
          :loading="loading"
          @click="requestSession"
        />
      </div>

      <!-- SECTION 2: PAIRING CODE DISPLAY -->
      <div v-else-if="sessionState === 'Pairing'" class="text-center q-pa-lg">
        <div class="text-subtitle1">Instruct client to open Samace Extension and enter Pairing Code:</div>
        <div class="text-h2 text-weight-bold text-primary q-my-md letter-spacing-2">
          {{ pairingCode }}
        </div>
        <div class="text-caption text-grey-7">Pairing Code Expires in 05:00. Single-Use Only.</div>
        <q-spinner-dots color="primary" size="40px" class="q-mt-md" />
      </div>

      <!-- SECTION 3: LIVE WEBRTC VIDEO CONTAINER & CONTROL -->
      <div v-else class="column items-center">
        <!-- Control Toolbar & Status Badges -->
        <div class="row justify-between items-center full-width q-mb-sm bg-grey-2 q-pa-sm rounded-borders">
          <div class="row items-center q-gutter-x-sm">
            <q-badge :color="stateBadgeColor" class="q-pa-xs text-uppercase font-weight-bold">
              {{ sessionState }}
            </q-badge>
            <span class="text-caption text-grey-8">Target Origin: <strong>http://127.0.0.1:8765</strong></span>
          </div>

          <div class="row items-center q-gutter-x-sm">
            <q-btn
              v-if="sessionState === 'ViewOnly' && canCoBrowseControl"
              color="negative"
              size="sm"
              icon="touch_app"
              label="Request Remote Control"
              @click="requestControl"
            />
            <q-btn
              color="negative"
              flat
              size="sm"
              icon="call_end"
              label="End Session"
              @click="endSession"
            />
          </div>
        </div>

        <!-- Video Stream Container -->
        <div class="video-container relative-position full-width flex flex-center bg-black rounded-borders overflow-hidden">
          <div v-if="sessionState === 'Paused'" class="paused-overlay text-warning text-h6 flex flex-center">
            ⏸️ Session Paused by Client
          </div>
          <video
            ref="remoteVideo"
            autoplay
            playsinline
            class="remote-video"
            @mousedown="handleMouseEvent($event, 'mousePressed')"
            @mouseup="handleMouseEvent($event, 'mouseReleased')"
            @mousemove="handleMouseMove($event)"
            @wheel="handleWheelEvent($event)"
          ></video>
        </div>
      </div>
    </q-card-section>
  </q-card>
</template>

<script>
const CANONICAL_SIGNAL_TYPES = new Set([
  "pair_request",
  "pair_accepted",
  "authenticate",
  "auth_success",
  "offer",
  "answer",
  "ice_candidate",
  "view_consent",
  "control_request",
  "control_approved",
  "control_revoked",
  "session_stop",
  "ping",
  "pong",
  "error",
]);

export default {
  name: "CoBrowseModal",
  props: {
    agentId: { type: String, required: true }
  },
  data() {
    return {
      sessionState: "Idle",
      sessionId: null,
      pairingCode: "",
      loading: false,
      ws: null,
      pc: null,
      dataChannel: null,
      lastMouseMoveTime: 0,
      sequenceNumber: 0
    };
  },
  computed: {
    canCoBrowseControl() {
      return this.$store?.state?.user?.role?.can_cobrowse_control ?? false;
    },
    stateBadgeColor() {
      if (this.sessionState === "ViewAndControl") return "negative";
      if (this.sessionState === "ViewOnly") return "positive";
      if (this.sessionState === "Paused") return "warning";
      return "grey";
    }
  },
  methods: {
    async requestSession() {
      this.loading = true;
      try {
        const res = await this.$axios.post("/api/v4/cobrowsing/request/", { agent_id: this.agentId });
        this.sessionId = res.data.session_id;
        this.pairingCode = res.data.pairing_code;
        this.sessionState = "Pairing";
        this.connectSignalingWebSocket();
      } catch (err) {
        this.$q.notify({ type: "negative", message: err.response?.data?.detail || "Failed to create session." });
      } finally {
        this.loading = false;
      }
    },
    connectSignalingWebSocket() {
      const wsProto = window.location.protocol === "https:" ? "wss:" : "ws:";
      const wsUrl = `${wsProto}//${window.location.host}/ws/cobrowse/signal/`;
      this.ws = new WebSocket(wsUrl);

      this.ws.onopen = () => {
        this.ws.send(JSON.stringify({
          type: "authenticate",
          session_id: this.sessionId,
          senderRole: "technician"
        }));
      };

      this.ws.onmessage = async (event) => {
        try {
          const data = JSON.parse(event.data);
          if (!data || typeof data !== "object" || !data.type) return;

          // Reject non-canonical message types
          if (!CANONICAL_SIGNAL_TYPES.has(data.type)) return;

          // Reject mismatched session IDs when provided
          if (data.session_id && data.session_id !== this.sessionId) return;

          if (data.type === "pair_accepted") {
            this.sessionState = "AwaitingViewConsent";
            await this.initiateWebRTCOffer();
          } else if (data.type === "view_consent") {
            if (data.authorized) {
              this.sessionState = "ViewOnly";
            }
          } else if (data.type === "answer") {
            if (this.pc && data.sdp) {
              await this.pc.setRemoteDescription(new RTCSessionDescription(data.sdp));
              this.sessionState = "ViewOnly";
            }
          } else if (data.type === "ice_candidate") {
            if (this.pc && data.candidate) {
              await this.pc.addIceCandidate(new RTCIceCandidate(data.candidate));
            }
          } else if (data.type === "control_approved") {
            if (this.canCoBrowseControl) {
              this.sessionState = "ViewAndControl";
            }
          } else if (data.type === "control_revoked") {
            this.sessionState = "ViewOnly";
          } else if (data.type === "session_stop") {
            this.endSession();
          } else if (data.type === "ping") {
            if (this.ws && this.ws.readyState === WebSocket.OPEN) {
              this.ws.send(JSON.stringify({ type: "pong", timestamp: Date.now() }));
            }
          }
        } catch (e) {
          // Fail closed silently on malformed JSON
        }
      };
    },
    async initiateWebRTCOffer() {
      try {
        const turnRes = await this.$axios.get("/api/v4/cobrowsing/turn-credentials/");
        const iceServers = turnRes.data.ice_servers || [];

        this.pc = new RTCPeerConnection({ iceServers });

        // Create DataChannel 'samace-control-v1' (Offerer role)
        this.dataChannel = this.pc.createDataChannel("samace-control-v1", { ordered: true });

        this.pc.ontrack = (event) => {
          if (event.streams && event.streams[0] && this.$refs.remoteVideo) {
            this.$refs.remoteVideo.srcObject = event.streams[0];
          }
        };

        this.pc.onicecandidate = (event) => {
          if (event.candidate && this.ws) {
            this.ws.send(JSON.stringify({
              type: "ice_candidate",
              session_id: this.sessionId,
              candidate: event.candidate,
              senderRole: "technician"
            }));
          }
        };

        const offer = await this.pc.createOffer({ offerToReceiveVideo: true, offerToReceiveAudio: false });
        await this.pc.setLocalDescription(offer);

        this.ws.send(JSON.stringify({
          type: "offer",
          session_id: this.sessionId,
          sdp: offer,
          senderRole: "technician"
        }));

        this.sessionState = "Negotiating";
      } catch (err) {
        this.$q.notify({ type: "negative", message: "Failed to initialize WebRTC peer connection." });
      }
    },
    requestControl() {
      if (this.canCoBrowseControl && this.ws) {
        this.ws.send(JSON.stringify({
          type: "control_request",
          session_id: this.sessionId,
          senderRole: "technician"
        }));
        this.sessionState = "AwaitingControlConsent";
      }
    },
    handleMouseEvent(e, action) {
      if (this.sessionState !== "ViewAndControl" || !this.canCoBrowseControl || !this.dataChannel || this.dataChannel.readyState !== "open") return;
      const coords = this.getMappedCoordinates(e);
      if (!coords) return;

      this.sendDataChannelMessage("pointerButton", {
        action: action,
        x: coords.targetX,
        y: coords.targetY,
        button: "left",
        clickCount: 1
      });
    },
    handleMouseMove(e) {
      if (this.sessionState !== "ViewAndControl" || !this.canCoBrowseControl || !this.dataChannel || this.dataChannel.readyState !== "open") return;
      const now = Date.now();
      if (now - this.lastMouseMoveTime < 20) return; // 50 Hz rate limit
      this.lastMouseMoveTime = now;

      const coords = this.getMappedCoordinates(e);
      if (!coords) return;

      this.sendDataChannelMessage("pointerMove", {
        x: coords.targetX,
        y: coords.targetY
      });
    },
    handleWheelEvent(e) {
      if (this.sessionState !== "ViewAndControl" || !this.canCoBrowseControl || !this.dataChannel || this.dataChannel.readyState !== "open") return;
      const coords = this.getMappedCoordinates(e);
      if (!coords) return;

      this.sendDataChannelMessage("wheel", {
        x: coords.targetX,
        y: coords.targetY,
        deltaX: e.deltaX,
        deltaY: e.deltaY
      });
    },
    getMappedCoordinates(e) {
      const video = this.$refs.remoteVideo;
      if (!video || !video.videoWidth || !video.videoHeight) return null;

      const rect = video.getBoundingClientRect();
      const nw = video.videoWidth;
      const nh = video.videoHeight;
      const naturalAspect = nw / nh;
      const containerAspect = rect.width / rect.height;

      let renderW, renderH, offsetX, offsetY;
      if (containerAspect > naturalAspect) {
        renderH = rect.height;
        renderW = renderH * naturalAspect;
        offsetX = (rect.width - renderW) / 2;
        offsetY = 0;
      } else {
        renderW = rect.width;
        renderH = renderW / naturalAspect;
        offsetX = 0;
        offsetY = (rect.height - renderH) / 2;
      }

      const relX = e.clientX - rect.left - offsetX;
      const relY = e.clientY - rect.top - offsetY;

      if (relX < 0 || relX > renderW || relY < 0 || relY > renderH) return null;

      return {
        targetX: (relX / renderW) * nw,
        targetY: (relY / renderH) * nh
      };
    },
    sendDataChannelMessage(messageType, payload) {
      this.sequenceNumber++;
      const msg = {
        protocolVersion: 1,
        sessionId: this.sessionId,
        sequence: this.sequenceNumber,
        messageType: messageType,
        senderRole: "technician",
        timestamp: Date.now(),
        payload: payload
      };
      const jsonStr = JSON.stringify(msg);
      if (jsonStr.length <= 4096) {
        this.dataChannel.send(jsonStr);
      }
    },
    endSession() {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        try {
          this.ws.send(JSON.stringify({
            type: "session_stop",
            session_id: this.sessionId,
            senderRole: "technician",
            reason: "UserRequestedStop"
          }));
        } catch (e) {
          // Ignore close send error
        }
      }
      if (this.dataChannel) {
        try { this.dataChannel.close(); } catch (e) {}
        this.dataChannel = null;
      }
      if (this.pc) {
        try { this.pc.close(); } catch (e) {}
        this.pc = null;
      }
      if (this.ws) {
        try { this.ws.close(); } catch (e) {}
        this.ws = null;
      }
      if (this.$refs.remoteVideo && this.$refs.remoteVideo.srcObject) {
        try {
          const tracks = this.$refs.remoteVideo.srcObject.getTracks();
          tracks.forEach(track => track.stop());
        } catch (e) {}
        this.$refs.remoteVideo.srcObject = null;
      }
      this.sessionState = "Idle";
      this.sessionId = null;
      this.pairingCode = "";
    }
  },
  beforeUnmount() {
    this.endSession();
  }
};
</script>

<style scoped>
.letter-spacing-2 {
  letter-spacing: 4px;
}
.video-container {
  width: 100%;
  height: 480px;
}
.remote-video {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}
.paused-overlay {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.85);
  z-index: 10;
}
</style>
