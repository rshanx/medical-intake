<template>
  <div id="hello">
    <div class="row margint" style="width:100%">
        <div class="col-md-2" style="border-right:1px solid #000;height:600px;overflow-y: scroll;margin-left:5px">
            <h4>已注册的人员</h4>
            <input class="form-control" v-model="searchingCode">
            <ul class="list-group margint">
                <li v-for="registered in matchingRegs" class="list-group-item" :class="getClass(registered.identity)">{{registered.code}}
                    <span class="badge" @click="deleteIdentity(registered)" role="button">&times</span>
                </li>
            </ul>
        </div>
        <div class="col-md-9">
            <h2>扫描到的指纹</h2>
            <div class="row">
                <div class="col-md-6">
                    <div class="input-group">
                        <input type="text" class="form-control" placeholder="服刑人员的编号" v-model="code">
                        <span class="input-group-btn">
                            <button class="btn btn-default" type="button" :disabled="!isRegisterable" @click="register()">注册</button>
                        </span>
                    </div>
                </div>
            </div>
            <div class="row">
                <div class="col-md-6" style="margin-top:10px">
                    <div class="input-group">
                        <input type="text" class="form-control" placeholder="请在右侧选择人员身份" v-model="identityShow">
                        <div class="input-group-btn">
                            <button type="button" class="btn btn-default dropdown-toggle" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">注册人员身份
                                <span class="caret"></span>
                            </button>
                            <ul class="dropdown-menu dropdown-menu-right">
                                <li @click="identity = 'medical'"><a>医药人员</a></li>
                                <li @click="identity = 'prison'"><a>服刑人员</a></li>
                                <li role="separator" class="divider"></li>
                                <li @click="identity = 'docter'"><a>管理医生</a></li>
                                <li @click="identity = 'police'"><a>教管民警</a></li>
                            </ul>
                        </div>
                        <!-- /btn-group -->
                    </div>
                    <!-- /input-group -->
                </div>
                <div class="col-md-3" style="top: -60px; position: relative">
                    <video id="video"></video>
                    <button id="startbutton" @click="takepicture()">拍摄</button>
                </div>
                <div class="col-md-3" style="top: -60px; position: relative">
                    <img id="photo" alt="" :src="headPic">
                </div>
            </div>
            <canvas id="canvas" hidden="true" />
            <div class="row  margint">
                <div class="col-md-4" v-for="imgSrc in imgs">
                    <img :src="imgSrc">
                </div>
            </div>
            <div class="row margint ">
                <div class="col-md-2 col-md-offset-6">
                    <button class="btn btn-danger" @click="reset()" v-if="imgs.length > 0">
                        <span class="glyphicon glyphicon-repeat"></span>
                    </button>
                </div>
            </div>
        </div>
    </div>
</div>

</template>

<script>
import _ from "lodash"
var width = 250; // We will scale the photo width to this
var height = 0; // This will be computed based on the input stream

// |streaming| indicates whether or not we're currently streaming
// video from the camera. Obviously, we start at false.

var streaming = false;

var video = null;
var canvas = null;
var photo = null;
var startbutton = null;
export default {
    name: 'finger-collecting',
    data() {
        return {
            imgs: [],
            websocket: '',
            code: "",
            identity: 'prison',
            regs: [],
            headPic: "",
            searchingCode:''
        }
    },
    mounted: function() {
        this.websocket = new WebSocket("ws://localhost:8090/finger");
        this.websocket.onopen = _ => {
            console.info("connected successfully!")
        }

        this.websocket.onmessage = (msg) => {
            console.info("recieved!");
            console.info(msg.data);
            if (this.imgs.length < 3) {
                this.imgs.push(msg.data);
            }
        }

        this.websocket.onerror = function(err){
        	console.info(err)
        }

        this.websocket.onclose = function(ob) {
        	console.info("closing:")
        }

        video = document.getElementById('video');
        canvas = document.getElementById('canvas');
        photo = document.getElementById('photo');
        startbutton = document.getElementById('startbutton');

        navigator.getMedia = (navigator.getUserMedia ||
            navigator.webkitGetUserMedia ||
            navigator.mozGetUserMedia ||
            navigator.msGetUserMedia);

        navigator.getMedia({
                video: true,
                audio: false
            },
            function(stream) {
                if (navigator.mozGetUserMedia) {
                    video.mozSrcObject = stream;
                } else {
                    var vendorURL = window.URL || window.webkitURL;
                    video.src = vendorURL.createObjectURL(stream);
                }
                video.play();
            },
            function(err) {
                console.log("An error occured! " + err);
            }
        );

        video.addEventListener('canplay', function(ev) {
            if (!streaming) {
                height = video.videoHeight / (video.videoWidth / width);

                // Firefox currently has a bug where the height can't be read from
                // the video, so we will make assumptions if this happens.

                if (isNaN(height)) {
                    height = width / (4 / 3);
                }

                video.setAttribute('width', width);
                video.setAttribute('height', height);
                canvas.setAttribute('width', width);
                canvas.setAttribute('height', height);
                streaming = true;
            }
        }, false);
        this.clearphoto();
        this.getRegisterdIdentity();
    },
    methods: {
        register: function() {
            var imgCode = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
                var r = Math.random()*16|0, v = c == 'x' ? r : (r&0x3|0x8);
                return v.toString(16);
            });
            var identityInfo = {
                'code': this.code,
                'identity': this.identity,
                'op': 'reg',
                'imgs': this.imgs,
                'head': this.headPic
            }
            console.info("sending:" + identityInfo)
            console.info(this.websocket.readyState)
            if (this.websocket.readyState == this.websocket.OPEN) {
                this.websocket.send(JSON.stringify(identityInfo))
                this.regs.push(identityInfo)
            }
            this.reset()
        },
        clearphoto: function() {
            this.headPic = ""
        },
        takepicture: function() {
            var context = canvas.getContext('2d');
            if (width && height) {
                canvas.width = width;
                canvas.height = height;
                context.drawImage(video, 0, 0, width, height);
                var data = canvas.toDataURL('image/png');
                this.headPic = data;

            } else {
                this.clearphoto();
            }
        },
        reset: function() {
            this.code = '';
            this.identity = 'prison';
            this.imgs = [];
            this.clearphoto();
        },
        getRegisterdIdentity: function(){
        	console.info("query registered identity")
        	this.$http.get('existing').then((res) =>{
        		this.regs = res.body;
        	})
        },
        deleteIdentity:function(reg){
        	this.regs = _.filter(this.regs, function(r){
        		return reg.code != r.code
        	})
    		this.$http.delete('identity/' + reg.code);
        },
        getClass(identity){
        	if(identity == 'prison'){
        		return 'list-group-item-success'
        	}else if(identity == 'medical'){
        		return 'list-group-item-warning'
        	}else if(identity == 'police'){
        		return 'list-group-item-danger'
        	}else if(identity == 'docter'){
        		return 'list-group-item-info'
        	}


        }
    },
    computed: {
        isRegisterable: function() {
            if ((this.identity == 'prison' || this.identity == 'medical') && this.headPic.length == 0) {
                return false;
            }
            if (this.imgs.length == 3 && this.code.length > 0) {
                return true
            }

            return false
        },
        matchingRegs:function(){
        	if(this.searchingCode){
        		return _.filter(this.regs, (r) =>{
        			return r.code.indexOf(this.searchingCode) >= 0
        		})
        	}
        	return this.regs;
        },
        identityShow:function(){
        	if(this.identity == 'prison'){
        		return '服刑人员'
        	}else if(this.identity == 'police'){
        		return '教管民警'
        	}else if(this.identity == 'medical'){
        		return '医药人员'
        	}else if(this.identity == 'docter'){
        		return '管理医生'
        	}
        }
    }
}
</script>
<style scoped>
.margint {
  margin-top: 10px
}

#video {
  border: 1px solid black;
  box-shadow: 2px 2px 3px black;
  height:200px;
}

#photo {
  border: 1px solid black;
  box-shadow: 2px 2px 3px black;
  height:200px;
}

#canvas {
  display:none;
}

.camera {
  display:inline-block;
}

.output {
  display:inline-block;
}

#startbutton {
  display:block;
  position:relative;
  margin-left:auto;
  margin-right:auto;
  bottom:32px;
  background-color: rgba(0, 150, 0, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.7);
  box-shadow: 0px 0px 1px 2px rgba(0, 0, 0, 0.2);
  font-size: 14px;
  font-family: "Lucida Grande", "Arial", sans-serif;
  color: rgba(255, 255, 255, 1.0);
}

.contentarea {
  font-size: 16px;
  font-family: "Lucida Grande", "Arial", sans-serif;
  width: 760px;
}
</style>
