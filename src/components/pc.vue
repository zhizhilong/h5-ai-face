<template>
  <div>
    <video id="videoCamera-face" preload autoplay loop muted></video>
    <canvas
      id="canvas-face"
      width="500"
      height="500"
      style="position: absolute; top: 0; left: 0"
    ></canvas>
  </div>
</template>

<script>
// import tracking from '@/assets/js/tracking-min.js'
// import '@/assets/js/data/face-min.js'
// export default {
//   name: "PC",
//   data() {
//     return {
//       videoEle: null,
//       trackerTask: null,
//       first: null,
//     };
//   },
//   mounted() {
//     this.openCamera();
//   },
//   methods: {
//     openCamera() {
//       var video = document.getElementById("video");
//       var canvas = document.getElementById("canvas");
//       var context = canvas.getContext("2d");
//       // console.log('-=-=-');
//       var tracker = new window.tracking.ObjectTracker("face"); // 引入第三方 库
//       tracker.setInitialScale(1);
//       tracker.setStepSize(2);
//       tracker.setEdgesDensity(0.1);

//       this.trackerTask = tracking.track("#video", tracker, { camera: true });
//       //-------  指定 canvas 的宽高 ，auto 自动播放
//       let constraints = {
//         video: { width: 500, height: 500 },
//         audio: true,
//       };

//       let promise = navigator.mediaDevices.getUserMedia(constraints); // h5 新的API

//       promise
//         .then(function (MediaStream) {
//           video.srcObject = MediaStream;
//           video.play();
//         })
//         .catch(function (PermissionDeniedError) {
//           console.log(PermissionDeniedError);
//         });
//       //--------------
//       let that = this;
//       that.tracker_fun(tracker, video, context, canvas); //open 摄像头，执行tracker_fun( 传入参数 )
//     },
//     tracker_fun(tracker, video, context, canvas) {
//       let that = this;
//       let set_clear;
//       set_clear = setTimeout(function () {
//         // 每秒 检测人脸 结果
//         tracker.on("track", function (event) {
//           // 视频流
//           // context.clearRect(0, 0, canvas.width, canvas.height);
//           if (!that.first) {
//             // if  --- > else  检测到人脸 success() =>{}
//             event.data.forEach(function (rect) {
//               if (rect.x) {
//                 that.first = rect.x;
//                 video.pause(); // success  将暂停 video
//                 console.log(rect);
//                 console.log(video);
//                 context.drawImage(video, 0, 0, 500, 500); // 绘制到 canvas
//                 context.font = "11px Helvetica";
//                 context.fillText("", 100, 40);
//                 context.strokeStyle = "#a64ceb";
//                 context.strokeRect(rect.x, rect.y, rect.width, rect.height);
//                 var data_url = canvas.toDataURL("image/png"); //base64 request
//                 console.log(data_url);

//                 // video.load();
//                 // that.first = null;
//                 // that.tracker_fun(tracker,video,context,canvas)
//                 return;
//               }
//             });
//           }
//         });
//         clearTimeout(set_clear);
//         console.log("-------");
//       }, 5000);
//     },
//   },
//   destroyed() {
//     // 停止侦测
//     // this.trackerTask.stop();
//     // // 关闭摄像头
//     // window.tracking.closeCamera();
//   },
// };

// 引入trackingjs所需文件
import tracking from "@/assets/js/tracking-min.js";
import "@/assets/js/data/face-min.js";

export default {
  name: "PC",
  data() {
    return {
      trackerTask: null,
      trackering: null,
      mediaStreamTrack: null,
    };
  },
  mounted() {
    this.getCompetence();
  },
  methods: {
    getCompetence() {
      let flag = true;
      const _this = this;
      const video = (this.mediaStreamTrack =
        document.getElementById("videoCamera-face"));
      const canvas = document.getElementById("canvas-face");
      const context = canvas.getContext("2d");

      const tracker = new window.tracking.ObjectTracker("face");
      tracker.setInitialScale(4);
      tracker.setStepSize(2);
      tracker.setEdgesDensity(0.1);
      // 启动摄像头初始化
      this.trackerTask = window.tracking.track("#videoCamera-face", tracker, {
        camera: true,
      });
      tracker.on("track", function (event) {
        // console.log(event);
        context.clearRect(0, 0, canvas.width, canvas.height);
        event.data.forEach(function (rect) {
          context.strokeStyle = "#ff0000";
          context.strokeRect(rect.x, rect.y, rect.width, rect.height);
        });
        if (event.data.length) {
          // 会不停的去检测人脸，所以这里需要做个锁
          if (flag) {
            // 裁剪出人脸并绘制下来
            const canvasUpload = document.getElementById("canvas-face-upload");
            const contextUpload = canvasUpload.getContext("2d");
            contextUpload.drawImage(video, 0, 0, 105, 105);
            flag = false;
            // 人脸的basa64
            const dataURL = canvasUpload.toDataURL("image/jpeg");
            // ajax请求
            _this.$store
              .dispatch("LoginByFaceID", {
                face_img: dataURL.slice(23), // 我们后台需要的basa64不要前缀
              })
              .then((res) => {
                let type = "success";
                // 登录成功跳转到首页
                if (res.data.code === 200) {
                  _this.$router.push({
                    path: "/",
                  });
                }
                // 登录失败重新进行人脸检测
                else {
                  type = "error";
                  setTimeout(() => {
                    flag = true;
                  }, 1500);
                }
                _this.$message({
                  message: res.data.message,
                  type,
                  center: true,
                });
              })
              .catch((error) => {
                console.log(error);
              });
          }
        }
      });
    },
  },
  destroyed() {
    if (!this.mediaStreamTrack) {
      return;
    }
    // 关闭摄像头和侦测
    this.mediaStreamTrack.srcObject.getTracks()[0].stop();
    this.trackerTask.stop();
  },
};
</script>

<style></style>
