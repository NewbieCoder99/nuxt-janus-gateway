<template>
    <div class="container">
        <div style="width: 100%;">
            <button class="button_cls primary" @click="start" veriant="info">Go Live</button>
        </div>
        <br>

        <!-- Streaming -->
        <!-- <div style="width:100%;background-color:green;">
            <video class="rtc_video" id="rtc_video" ref="rtc_mv" width="700px" height="600px" autoplay :srcObject.prop="stream"></video>
        </div> -->

        <br>
        <div style="width:100%%;background-color:white;">
            <div class="videoright"></div>
            <!-- Streamer -->
            <video autoplay :srcObject.prop="streamer"></video>
        </div>
    </div>
</template>

<script type="text/javascript">

    import Janus from '~/components/janus'

    export default {
      name: 'App',
      data() {
            return {
                janus_url : '',
                sessionId: null,
                add_snap: false,
                janus: null,
                plugin: null,
                stream: null,
                streamer: null,
                echotest: null,
                spinner: null,
                error: null,
                status: null,
                streamBody: null,
                canvasWidth: 300,
                canvasHeight: 300,
                jans_domain: 'app.omelo.co',
                streamJsep: null,
                opaqueId: "echotest-"+Janus.randomString(12),
                simulcastStarted: false,
                streamList: {
                    selected: null,
                    options: []
                }
            }
      },
      mounted() {
        Janus.init({
            debug: "all",
            dependencies: Janus.useDefaultDependencies(),
            callback: () => {
                if(!Janus.isWebrtcSupported()){
                    alert('Not WebRTC support...');
                    return;
                }
                this.connect(this.janus_url)
            }
        })
      },
      methods: {
        connect(server) {
            this.janus = new Janus({
                server,
                success: () => {
                    this.attachPlugin()
                },
                error: (error) => {
                    this.onError('Failed to connect to janus server', error)
                },
                destroyed: () => {
                    window.location.reload()
                }
            })
        },
        attachPlugin() {

            console.log("this is my chcekning wrk");
            console.log(this.opaqueId);
            this.janus.attach({
                plugin: "janus.plugin.echotest",
                opaqueId: this.opaqueId,
                success: (pluginHandle) => {
                    console.log(pluginHandle);
                    this.echotest = pluginHandle;
                    Janus.log("Plugin attached! ("+this.echotest.getPlugin()+", id"+this.echotest.getId()+")");
                    this.sessionId = this.echotest.getId();
                    let body = {audio: true, video: true};
                    this.streamBody = body;
                    Janus.debug("sending message:", body);
                    this.echotest.send({message: body});
                    Janus.debug("Trying a createOffer to (audio/video sendrecv)");
                },
                error: (error) => {
                    this.onError('Error attaching plugin... ', error)
                },
                onmessage: (msg, jsep) => {
                    Janus.debug("... Got a message ..." , msg);
                    if(jsep) {
                        Janus.debug("Handling SDP as well...", jsep);
                        this.echotest.handleRemoteJsep({jsep: jsep});
                    }

                    var result = msg['result'];

                    if(result) {

                        if(result === "done") {
                            alert("over")
                            if(this.spinner) {
                                this.spinner.stop();
                            }
                            this.spinner = null;
                            return;
                        }

                        var status = result['status'];
                        if(status === "slow_link") {
                            alert("waiting networking");
                        }
                    }

                    var substream = msg['substream'];
                    var temporal = msg['temporal'];
                    if((substream !== null && substream !==undefined) || (temporal !== null && temporal !== undefined)) {
                        if(!this.simulcastStarted) {
                            this.simulcastStarted = true;
                        }
                    }
                },
                onlocalstream: (stream) => {
                    this.onLocalstreamer(stream);
                    if(this.echotest.webrtcStuff.pc.iceConnectionState !== 'completed' && this.echotest.webrtcStuff.pc.iceConnectionState !== 'connected') {


                        console.log('======================================================');
                        console.log(this.spinner);
                        console.log('======================================================');

                        if(this.spinner == null) {
                            var target = document.getElementById('videoright');
                            this.spinner = new Spinner({top:100}).spin(target);
                            this.spinner.spin();
                        } else {
                            this.spinner.spin();
                        }
                    }
                },
                onremotestream: (stream) => {
                    console.log(stream);
                    this.onRemoteStream(stream)
                },
                oncleanup: () => {
                    this.onCleanup()
                }
            })
        },

        start() {
            // this.echotest.send({message: this.streamBody, jsep: this.streamJsep});
            let tmp_body = this.streamBody;
            let tmp_echotest = this.echotest;
            this.echotest.createOffer({
                media: {data: true},
                success: function(jsep) {
                    Janus.debug("Got SDP", jsep);
                    this.streamJsep = jsep;
                    // console.log(this.echotest);
                    tmp_echotest.send({message: tmp_body, jsep: jsep});
                },
                error: function(error) {
                    Janus.error("WebRTC error", error);
                    alert("WebRTC error +" + error.message);
                }
            })
        },
        stop() {
            this.plugin.send({ message: { request: "stop" } })
            this.plugin.hangup()
            this.onCleanup()
        },
        onRemoteStream(stream) {
            if (stream.active) {
                this.stream = stream
            } else {
                this.stream = null
            }
        },
        onLocalstreamer(stream){
            if(stream.active){
                this.streamer = stream
            }else{
                this.streamer = null;
            }
        },
        onCleanup() {
            this.stream = null
            this.status = null
            this.error = null
        },
        onError(message, error='dd') {
            Janus.error(message, error)
            this.error = message + error
        }
      }
    }
</script>
