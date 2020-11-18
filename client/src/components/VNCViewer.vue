<template>
  <div>
    <p>Password:</p>
    <p>
      <input type="password" v-model="password" />
      <button @click="startCapture">Connect</button>
    </p>
    <div />
  </div>
</template>

<script>
import RFB from "@novnc/novnc/core/rfb";
import uuid from "uuid/dist/v4";

const serverHost = "localhost"; // TODO: get from env
const serverPort = 1337; // TODO: get from env

export default {
  name: 'VNCViewer',
  props: {
    clientName: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      rfb: null,
      serviceClient: null,
      connectionInfo: null,
      password: "",
    };
  },
  methods: {
    setupTunnel() {
      const connect = () => {
        this.serviceClient = new WebSocket(`ws://${serverHost}/${serverPort}`);

        this.serviceClient.onopen = () => {
          console.info("Connection to server established, waiting for agent.");
          let msg = { type: "client", name: this.clientName };
          if (this.connectionInfo && this.connectionInfo.uuid) {
            msg.uuid = this.connectionInfo.uuid;
          }
          this.serviceClient.send(JSON.stringify(msg));
        };

        this.serviceClient.onmessage = async (message) => {
          try {
            const dataText = await message.data.text();
            let dataJson = JSON.parse(dataText);
            if (dataJson.pong) return;
            if (dataJson.agentDied || !dataJson.port) {
              this.connectionInfo = null;
              return;
            }
            this.connectionInfo = dataJson;
          } catch (err) {
            console.error("SERVICE_SOCKET", err.name || err.code, err.message);
          }
        };

        this.serviceClient.onerror = (err) => {
          console.log("SERVICE_SOCKET", err.name || err.code, err.message);
        };

        this.serviceClient.onclose = () => {
          if (this.serviceClient.readyState !== WebSocket.CLOSED) {
            this.serviceClient.close();
          }
          connectWithDelay(5000);
        };
      };

      const connectWithDelay = (delay) => {
        if (!delay) {
          connect();
        } else {
          setTimeout(connect, delay);
        }
      };

      connectWithDelay(500);
    },
    startCapture() {
      this.rfb.sendCredentials({ password: this.password });
    },
  },
  watch: {
    connectionInfo(cInfo) {
      if (this.rfb) {
        this.rfb.disconnect();
        this.rfb = null;
      }
      if (cInfo) {
        this.rfb = new RFB(this.$el.lastChild, `ws://${serverHost}/${this.connectionInfo.port}`);
        this.rfb._sock.on("open", () => {
          if (this.rfb._rfbConnectionState === "connecting" && !this.rfb._rfbInitState) {
            this.rfb._sock._websocket.send(`{ "type": "client", "uuid": "${uuid()}" }`);
            this.rfb._rfbInitState = "ProtocolVersion";
          } else {
            this.rfb._fail("Unexpected server connection while " + this.rfb._rfbConnectionState);
          }
        });
      }
    },
  },
  mounted() {
    this.setupTunnel();
  },
  beforeDestroy() {
    if (this.rfb) {
      this.rfb.disconnect();
    }
    if (this.serviceClient && this.serviceClient.readyState !== WebSocket.CLOSED) {
      this.serviceClient.close();
    }
  },
};
</script>
