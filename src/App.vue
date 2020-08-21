<template>
    <div id="app">
        <div class="container">
            <h1>Status: {{ status }}</h1>
            <h3>Your Peer ID: {{ peerID || 'Fetching...' }}</h3>
            <h3 class=""></h3>
            <label>
                Target Unique ID
                <input type="text" v-model="targetID"/>
            </label>
            <button @click="connectToPeer(targetID)">Connect</button>
            <div class="call-container">
                <video id="remote-video" autoplay height="400px"/>
                <video id="local-video" autoplay height="200px" muted/>
            </div>
            <div class="peer-box">
                <p v-for="peer in connectedPeers" :key="peer" v-if="peer !== peerID">{{ peer }}</p>
            </div>
        </div>
    </div>
</template>

<script>
import axios from 'axios';
import Peer from 'peerjs';

const CONN_DETAILS = {
    host: '68.183.197.101',
    port: 9000,
    key: 'connect_me'
}

export default {
    name: 'App',
    components: {},
    async created() {
        this.peerID = await this.getUniqueID();

        this.getLocalMedia({
            success: (stream) => {
                this.localStream = stream;
                // this.onReceiveStream(stream, 'local-video');
            },
            error: (e) => {
                alert('Cant access A/V stream.');
                console.error(e);
            }
        });

        this.startConnection();
    },
    data: () => ({
        peer: null,
        peerID: null,
        peerStream: null,
        localStream: null,
        connectedPeers: [],
        targetID: '',
        status: 'Offline.'
    }),
    methods: {
        async getUniqueID() {
            const url = `http://${CONN_DETAILS.host}:${CONN_DETAILS.port}/${CONN_DETAILS.key}/id`;
            const res = await axios.get(url);
            return res.data;
        },
        async getConnectedPeers() {
            const url = `http://${CONN_DETAILS.host}:${CONN_DETAILS.port}/${CONN_DETAILS.key}/peers`;
            const res = await axios.get(url);
            return res.data;
        },
        startConnection() {
            const peer = new Peer(this.peerID, {
                ...CONN_DETAILS,
                debug: 3
            });

            this.peer = peer;

            const checkConnection = async () => {
                if (peer.disconnected)
                    peer.reconnect();
                this.connectedPeers = await this.getConnectedPeers();
            }

            const timer = setInterval(checkConnection, 2000);

            peer.on('open', id => {
                this.status = 'Standing By For Connections.';
            });

            peer.on('close', id => {
                this.status = 'Offline.';
            });

            peer.on('disconnected', () => {
                this.status = 'Disconnected.';
            });

            peer.on('call', call => {
                call.answer(this.localStream);

                call.on('stream', stream => {
                    this.peerStream = stream;
                    this.onReceiveStream(stream, 'remote-video');
                });

                call.on('close', () => {
                    console.log('Video call ended.')
                });
            })


            window.addEventListener('beforeunload', (e) => {
                peer.disconnect();
            }, false)
        },
        connectToPeer(id) {
            console.log('Trying to place!');
            const call = this.peer.call(id, this.localStream);
            call.on('stream', stream => {
                this.peerStream = stream;
                this.onReceiveStream(stream, 'remote-video');
            });
        },
        getLocalMedia(callbacks) {
            navigator.getUserMedia({audio: true, video: true}, (s) => callbacks.success(s), callbacks.error);
        },
        onReceiveStream(stream, elementID) {
            const video = document.getElementById(elementID);
            video.srcObject = (stream);
        }
    }
}
</script>

<style>
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
}
</style>
