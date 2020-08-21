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
            <button>Connect</button>
            <div class="call-container">
                <video class="remote-video" autoplay/>
                <video class="local-video" autoplay muted/>
            </div>
            <div class="peer-box">
                <p v-for="peer in connectedPeers" :key="peer" v-if="peer !== peerID">{{peer}}</p>
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
        this.connectPeer();
    },
    data: () => ({
        peer: null,
        peerID: null,
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
        async getConnectedPeers(){
            const url = `http://${CONN_DETAILS.host}:${CONN_DETAILS.port}/${CONN_DETAILS.key}/peers`;
            const res = await axios.get(url);
            return res.data;
        },
        connectPeer() {
            const peer = new Peer(this.peerID, {
                ...CONN_DETAILS,
                debug: 3
            });

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

            window.addEventListener('beforeunload', (e) => {
                peer.disconnect();
            }, false)
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
