<template>
  <div id="app">
    <div v-show="showContainer" class="face-capture" id="face-capture">
      <p class="tip">请保持人像在取景框内</p>
      <video
        id="video"
        :width="vwidth"
        :height="vheight"
        playsinline
        webkit-playsinline
      ></video>
      <canvas id="refCanvas" :width="cwidth" :height="cheight"></canvas>
      <img
        class="img-cover"
        src="@/assets/img/face/yuanxingtouming.png"
        alt=""
      />
      <p class="contentp">{{ scanTip }}</p>
    </div>
    <div v-if="!showContainer" class="img-face">
      <img class="imgurl" :src="imgUrl" />
    </div>
  </div>
</template>

<script>
// import HelloWorld from './components/HelloWorld.vue'
import tracking from "@/assets/js/tracking-min.js";

import "@/assets/js/data/face-min.js";

import "@/assets/js/data/eye-min.js";

import "@/assets/js/data/mouth-min.js";

import { faceInfo, conFace } from "@/request/api/my.js";
export default {
  name: "App",
  components: {
    // HelloWorld
  },
  data() {
    return {
      screenSize: {
        width: window.screen.width,
        height: window.screen.height,
      },
      URL: null,
      streamIns: null, // 视频流
      showContainer: true, // 显示
      tracker: null,
      tipFlag: false, // 提示用户已经检测到
      flag: false, // 判断是否已经拍照
      context: null, // canvas上下文
      profile: [], // 轮廓
      removePhotoID: null, // 停止转换图片
      scanTip: "人脸识别中...", // 提示文字
      imgUrl: "",
      canvas: null,
      trackertask: null,
      vwidth: "266",
      vheight: "266",
      cwidth: "266",
      cheight: "266",
      userInfo: {},
      orderData: {},
    };
  },
  mounted() {
    //设置video canvas宽高
    const scale = this.screenSize.width / 375;
    this.vwidth = 266 * scale;
    this.vheight = 266 * scale;
    this.cwidth = 266 * scale;
    this.cheight = 266 * scale;
    this.playVideo();
  },

  methods: {
    playVideo() {
      this.getUserMedia(
        {
          //摄像头拍摄的区域
          video: {
            width: 500,
            height: 500,
            facingMode: "user",
          } /* 前置优先 */,
        },
        this.success,
        this.error
      );
    }, // 访问用户媒体设备

    getUserMedia(constrains, success, error) {
      if (navigator.mediaDevices.getUserMedia) {
        //最新标准API
        navigator.mediaDevices
          .getUserMedia(constrains)
          .then(success)
          .catch(error);
      } else if (navigator.webkitGetUserMedia) {
        //webkit内核浏览器
        navigator.webkitGetUserMedia(constrains).then(success).catch(error);
      } else if (navigator.mozGetUserMedia) {
        //Firefox浏览器
        navagator.mozGetUserMedia(constrains).then(success).catch(error);
      } else if (navigator.getUserMedia) {
        //旧版API
        navigator.getUserMedia(constrains).then(success).catch(error);
      } else {
        this.scanTip = "你的浏览器不支持访问用户媒体设备";
      }

      //        success(stream) {
      //       this.streamIns = stream;
      //       const video = document.getElementById("video");
      //       // webkit内核浏览器
      //       this.URL = window.URL || window.webkitURL;
      //       if ("srcObject" in video) {
      //         video.srcObject = stream;
      //       } else {
      //         video.src = this.URL.createObjectURL(stream);
      //       }
      //       // 苹果手机的系统弹框会阻止js的线程的继续执行 手动0.1秒之后自动执行代码
      //       setTimeout(() => {
      //         video.play();
      //         this.initTracker();// 人脸捕捉
      //       }, 100);
      //     },
      //     error(e) {
      //       this.scanTip = "访问用户媒体失败";
      //     }
    },
    initTracker() {
      this.context = document.getElementById("refCanvas").getContext("2d"); // 画布
      this.canvas = document.getElementById("refCanvas");
      this.tracker = new window.tracking.ObjectTracker("face"); // tracker实例
      this.tracker.setInitialScale(4);
      this.tracker.setStepSize(2); // 设置步长
      this.tracker.setEdgesDensity(0.1);
      try {
        this.trackertask = window.tracking.track("#video", this.tracker); // 开始追踪
      } catch (e) {
        this.scanTip = "访问用户媒体失败，请重试";
      } //开始捕捉方法 一直不停的检测人脸直到检测到人脸
      this.tracker.on("track", (e) => {
        //画布描绘之前清空画布
        this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
        if (e.data.length === 0) {
          this.scanTip = "未检测到人脸";
        } else {
          e.data.forEach((rect) => {
            //设置canvas 方框的颜色大小
            this.context.strokeStyle = "#42e365";
            this.context.lineWidth = 2;
            this.context.strokeRect(rect.x, rect.y, rect.width, rect.height);
          });
          if (!this.tipFlag) {
            this.scanTip = "检测成功，正在拍照，请保持不动2秒";
          } // 1.5秒后拍照，仅拍一次 给用户一个准备时间 // falg 限制一直捕捉人脸，只要拍照之后就停止检测
          if (!this.flag) {
            this.scanTip = "拍照中...";
            this.flag = true;
            this.removePhotoID = setTimeout(() => {
              this.tackPhoto();
              document.getElementById("video").pause();
              this.tipFlag = true;
            }, 1500);
          }
        }
      });
    },
    tackPhoto() {
      // 在画布上面绘制拍到的照片
      this.context.drawImage(
        document.getElementById("video"),
        0,
        0,
        this.vwidth,
        this.vwidth
      ); // 保存为base64格式
      this.imgUrl = this.saveAsPNG(document.getElementById("refCanvas"));
      /** 拿到base64格式图片之后就可以在this.compare方法中去调用后端接口比较了，也可以调用getBlobBydataURI方法转化成文件再去比较
       * 我们项目里有一个设置个人头像的地方，先保存一下用户的图片，然后去拿这个图片的地址和当前拍照图片给后端接口去比较。
       * */ // this.compare(imgUrl) //判断图片大小
      this.imgSize();
      this.faceToTengXun(); // 人脸比对
      this.close();
    },
    imgSize() {
      if (this.imgUrl) {
        // 获取base64图片byte大小
        const equalIndex = this.imgUrl.indexOf("="); // 获取=号下标
        let size;
        if (equalIndex > 0) {
          const str = this.imgUrl.substring(0, equalIndex); // 去除=号
          const strLength = str.length;
          const fileLength = strLength - (strLength / 8) * 2; // 真实的图片byte大小
          size = Math.floor(fileLength / 1024); // 向下取整
          console.log("size", size + "KB");
        } else {
          const strLength = this.imgUrl.length;
          const fileLength = strLength - (strLength / 8) * 2;
          size = Math.floor(fileLength / 1024); // 向下取整
          console.log("size", size + "KB");
        }
        if (size > 1024) {
          // 图片超过1M 按比例压缩
          this.imgUrl = document
            .getElementById("refCanvas")
            .toDataURL("image/png", 1024 / size);
        }
      }
    }, // Base64转文件
    getBlobBydataURI(dataURI, type) {
      var binary = window.atob(dataURI.split(",")[1]);
      var array = [];
      for (var i = 0; i < binary.length; i++) {
        array.push(binary.charCodeAt(i));
      }
      return new Blob([new Uint8Array(array)], {
        type: type,
      });
    }, // compare(url) { //     let blob = this.getBlobBydataURI(url, 'image/png') //     let formData = new FormData() //     formData.append("file", blob, "file_" + Date.parse(new Date()) + ".png") //     // TODO 得到文件后进行人脸识别 // }, // 保存为png,base64格式图片
    saveAsPNG(c) {
      return c.toDataURL("image/png", 0.4);
    },
    close() {
      this.flag = false;
      this.tipFlag = false;
      this.showContainer = false;
      this.context = null;
      this.scanTip = "人脸识别中...";
      clearTimeout(this.removePhotoID);
      if (this.streamIns) {
        this.streamIns.enabled = false;
        this.streamIns.getTracks()[0].stop();
        this.streamIns.getVideoTracks()[0].stop();
      }
      this.streamIns = null;
      this.trackertask.stop();
      this.tracker = null;
    },
  },
};
</script>

<style lang="scss">
.face-capture {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content:center;
}
// .tip {
//   position: fixed;
//   top: 48px;
//   z-index: 5;
//   font-size: 18px;
//   font-family: PingFangSC-Medium, PingFang SC;
//   font-weight: 500;
//   color: #333333;
//   line-height: 25px;
// }

// .face-capture video,

// .face-capture canvas {
//   position: fixed;
//   top: 117.5px;
//   object-fit: cover;
//   z-index: 2;
//   background-repeat: no-repeat;
//   background-size: 100% 100%;
// }

// .face-capture .img-cover {
//   position: fixed;
//   top: 63px;
//   width: 375px;
//   height: 375px;
//   object-fit: cover;
//   z-index: 3;
//   background-repeat: no-repeat;
//   background-size: 100% 100%;
// }

// .face-capture .contentp {
//   position: fixed;
//   top: 438px;
//   font-size: 18px;
//   font-weight: 500;
//   color: #333333;
// }

// .face-capture .rect {
//   border: 2px solid #0aeb08;
//   position: fixed;
//   z-index: 4;
// }

// .img-face {
//   display: flex;
//   flex-direction: column;
//   align-items: center;
//   justify-content: center;
// }

// .img-face .imgurl {
//   position: fixed;
//   top: 117.5px;
//   width: 266px;
//   height: 266px;
//   border-radius: 133px;
// }
</style>
