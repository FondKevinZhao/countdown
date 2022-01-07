<template>
  <div class="hello-pomodoro">
    <!-- 上 -->
    <div class="top-part">
      <div class="title">
        <div class="title-en">Wizard Timer</div>
        <div class="title-ch">之番茄工作法</div>
      </div>
      <div class="countdown" :class="{ 'break-time': isBreakTime }">
        {{ countDownList }}
      </div>
      <el-progress
        :text-inside="true"
        :show-text="false"
        :stroke-width="18"
        :percentage="percentage"
        status="success"
        class="progress"
      ></el-progress>
      <div class="tip-text">
        请在 {{ moment(startTime).format("HH:mm") }} -
        {{ moment(startTime + workTime * 60 * 1000).format("HH:mm") }}
        完成，加油
      </div>
    </div>

    <!-- 中 -->
    <div class="middle-part">
      <el-input
        type="textarea"
        v-model="textarea"
        @input="inputOnInputHandle"
        :rows="1"
        placeholder="请输入20个字符以内的任务标题"
      ></el-input>
      <div class="start-break-stop">
        <el-button
          type="primary"
          class="work"
          :disabled="!textarea || !workTime || !!interval"
          @click="starWorkTimeClickHandle"
          >开始工作</el-button
        >
        <el-button
          type="danger"
          @click="timePause"
          :disabled="interval === null"
          >暂停/开始</el-button
        >
        <el-button type="danger" @click="stopTaskHandle" :disabled="!interval"
          >停止</el-button
        >
      </div>
    </div>

    <!-- 下 -->
    <div class="bottom-part">
      <div class="setting">
        <el-button type="primary" @click="settingBtn">设置</el-button>
        <div v-show="isSetting" class="setting-content">
          <div class="input-suffix work-time">
            工作时长
            <el-input
              v-model="willWorkTime"
              @change="changeWorkTimeHandle"
              @input="inputWorkTimeHandle"
            ></el-input>
          </div>
          <div class="input-suffix">
            休息时长
            <el-input v-model="breakTime" @change="changeBreakTime"></el-input>
          </div>
        </div>
      </div>
      <div class="history">
        <div class="history-btn">
          <el-button type="primary" @click="historyBtn">任务记录</el-button>
          <el-button type="danger" @click="clearAllHistoryHandle"
            >清空记录</el-button
          >
        </div>
        <ul v-show="showHistory">
          <li class="stop-clicked" v-for="item in historyList" :key="item.id">
            任务：{{ item.task }}, 时间：{{ item.finishTime }}，停止：{{
              item.stop
            }}, 是否完成：{{ item.done }}
          </li>
        </ul>
        <div v-if="!this.historyList.length && showHistory" class="no-task">
          暂无任务记录
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import moment from "moment";
export default {
  data() {
    return {
      // input: "",
      textarea: "", // 任务标题输入框内容
      workTime: 25,
      willWorkTime: 25,
      breakTime: 5, // 休息时间
      countDownList: "00:00", // 显示时间
      // actEndTime: 0,
      handleTime: 0, // 获得秒数
      interval: null, // 定时器名字
      isBreak: false, // 是否休息
      isExpand: false, // 是否显示番茄倒计时方法(家里的版本没写)
      isSetting: false, // 是否显示设置内容
      showHistory: false, // 是否显示任务记录内容
      isBreakTime: false, // 是否显示灰色时间
      percentage: 100, // 进度条
      startTime: Date.now(),
      historyList: JSON.parse(localStorage.getItem("historyList")) || [],
    };
  },
  mounted() {
    // this.countDown(this.endTime);
    // const y = moment(this.breakTime).format("mm:ss");
    // console.log("y", y);
    window.addEventListener("beforeunload", () => this.stopTaskHandle());
  },

  beforeDestroy() {
    window.removeEventListener("beforeunload", () => this.stopTaskHandle());
  },

  computed: {
    endTime() {
      return Date.now() + this.workTime * 60 * 1000; // 问题就出在这个计算属性上,这个计算属性,因为计算属性,workTime不更新,所以不会更新
    },
  },
  methods: {
    moment,
    timeFormat(param) {
      return param < 10 ? "0" + param : param;
    },

    countDown(endTime) {
      let timeObj = {
        min: "00",
        sec: "00",
      };
      let initPercentage = this.percentage;
      let allTime = 0;
      console.log();
      if (this.isBreakTime) {
        allTime = this.breakTime;
      } else {
        allTime = this.workTime;
      }
      const everyTimes = (initPercentage / (allTime * 60 * 1000)) * 16.67;
      this.interval = setInterval(() => {
        // 当前毫秒数
        let newTime = Date.now();
        // 如果时间未结束，对时间进行处理
        if (endTime - newTime > 0) {
          /* 秒 */
          this.handleTime = (endTime - newTime) / 1000;
          // 获取分、秒
          let min = Math.floor(
            ((this.handleTime % (60 * 60 * 24)) % 3600) / 60
          );
          let sec = Math.floor(
            ((this.handleTime % (60 * 60 * 24)) % 3600) % 60
          );
          timeObj = {
            min: this.timeFormat(min),
            sec: this.timeFormat(sec),
          };
          initPercentage = initPercentage - everyTimes;
          if (initPercentage < 0) {
            initPercentage = 0;
          }
          this.percentage = +initPercentage.toFixed(2);
        } else {
          clearInterval(this.interval);
          this.interval = null;
          console.log("倒计时完成");
          // 倒计时走完，全部设置为"0"
          timeObj = {
            min: "00",
            sec: "00",
          };
          this.percentage = 100;
          if (this.isBreakTime === false) {
            this.textarea = "休息一下";
            this.isBreakTime = true;
            const historyList = this.historyList;
            historyList[historyList.length - 1].done = true;
            historyList[historyList.length - 1].isClear = true;
            this.historyList = historyList;
            localStorage.setItem("historyList", JSON.stringify(historyList));
            this.countDown(this.breakTime * 60 * 1000 + Date.now());
            this.workTimeNotification();
          } else {
            // console.log("休息结束");
            this.textarea = "";
            this.restTimeNotification();
          }
        }
        this.countDownList = timeObj.min + ":" + timeObj.sec;
      }, 16.67);
    },

    settingBtn() {
      this.isSetting = !this.isSetting;
    },

    historyBtn() {
      this.showHistory = !this.showHistory;
    },

    /* 停止 */
    timePause() {
      if (this.interval) {
        clearInterval(this.interval);
        this.interval = 0;
        this.workTime =
          (this.workTime * 60 * 1000 - (Date.now() - this.startTime)) /
          60 /
          1000;
      } else {
        this.startTime = Date.now();
        this.countDown(Date.now() + this.workTime * 60 * 1000);
      }
    },

    // textarea 的onchange事件
    inputOnInputHandle(val) {
      this.textarea = val.trim();
    },
    // 工作时长input框
    changeWorkTimeHandle() {
      if (this.willWorkTime <= 0) this.willWorkTime = 1;
      if (this.willWorkTime > 61) this.willWorkTime = 60;
      this.workTime = this.willWorkTime;

      let reg = new RegExp("^[0-9]*$");
      reg.test(this.willWorkTime)
        ? this.$message.success("工作时长设置成功")
        : this.$message.warning("请输入60以内的数字");
    },

    inputWorkTimeHandle(val) {
      if (val.length > 3) {
        this.$message.info("请输入0-9之间的数字");
        this.willWorkTime = this.willWorkTime.slice(0, 2);
      }
    },
    // 开始工作
    starWorkTimeClickHandle() {
      const localHistoryList =
        JSON.parse(localStorage.getItem("historyList")) || [];
      if (localHistoryList.length >= 4) {
        return this.$message.warning("请清空记录再开始");
      }
      this.startTime = Date.now();
      this.isBreakTime = false;
      clearInterval(this.interval);
      this.workTime = this.willWorkTime;
      this.countDown(Date.now() + this.workTime * 60 * 1000);
      const historyList = [
        ...this.historyList,
        {
          id: Date.now(),
          task: this.textarea,
          finishTime:
            moment(this.startTime).format("HH:mm") +
            "-" +
            moment(this.startTime + this.workTime * 60 * 1000).format("HH:mm"),
          stop: moment(this.startTime + this.workTime * 60 * 1000).format(
            "HH:mm"
          ),
          done: false,
          isClear: false,
        },
      ];
      localStorage.setItem("historyList", JSON.stringify(historyList));
      this.historyList = historyList;
    },

    /* 休息时间 */
    changeBreakTime(val) {
      if (val <= 0) val = 1;
      if (val > 61) val = 60;
      this.breakTime = val;
      let reg = new RegExp("^[0-9]*$");
      reg.test(val)
        ? this.$message.success("休息时长设置成功~")
        : this.$message.warning("请输入60以内的数字");
    },
    // 清空记录
    clearAllHistoryHandle() {
      let historyList = JSON.parse(localStorage.getItem("historyList"));
      historyList = historyList.filter((item) => !item.isClear);
      localStorage.setItem("historyList", JSON.stringify(historyList));
      // 里面的数组赋值给外面的数组，以达到更新视图的效果
      this.historyList = historyList;
    },
    // 停止任务
    stopTaskHandle() {
      const historyList = this.historyList;
      historyList[historyList.length - 1].isClear = true;
      historyList[historyList.length - 1].stop = moment(Date.now()).format(
        "YY-MM-DD HH:mm:ss"
      );
      this.historyList = historyList;
      localStorage.setItem("historyList", JSON.stringify(historyList));
      clearInterval(this.interval);
      this.countDownList = "00:00";
      this.percentage = 100;
      this.interval = null;
      this.textarea = "";
    },

    /* 工作时间结束后，打开workTimeNotification */
    workTimeNotification() {
      this.$notify({
        title: "番茄钟完成",
        message: "恭喜您完成当前番茄钟！下面进入休息时间",
        type: "success",
        duration: 10000,
      });
    },
    /* 休息间结束后，打开restTimeNotification */
    restTimeNotification() {
      this.$notify({
        title: "休息完成",
        message: "休息时间总是很短暂，让我们开始工作吧~",
        type: "success",
        duration: 0,
      });
    },
  },
};
</script>

<style scoped>
.hello-pomodoro {
  width: 600px;
  height: 100%;
  margin: 0 auto;
  border: 1px solid skyblue;
}
.top-part {
  margin-top: 30px;
  text-align: center;
}
.title {
  width: 100%;
  display: flex;
  justify-content: center;
}

.title {
  display: block;
}
.title .title-en {
  display: block;
  font-size: 30px;
}
.countdown {
  font-size: 80px;
  color: #409eff;
}
.tip-text {
  font-size: 20px;
  margin: 10px 0;
}

/* middle */

.middle-part >>> .el-textarea__inner {
  text-align: center;
  font-size: 20px;
}

.setting-content {
  margin-left: 15px;
}
.input-suffix {
  display: flex;
  align-items: center;
  width: 200px;
  margin: 10px 0;
}
.input-suffix .el-input {
  width: 100px;
  margin-left: 20px;
}

ul li {
  padding: 3px;
  padding-left: 10px;
  margin: 5px;
  background-color: #0dbc79;
  color: #fff;
}

ul {
  margin-left: 6px !important;
}

ul,
li {
  list-style: none;
  padding: 0;
  margin: 0;
}

.el-button {
  margin: 10px;
}

/* 休息样式 */
.break-time {
  color: #ccc;
}

/* progress */
.top-part >>> .el-progress-bar__inner {
  transition: none;
}

.no-task {
  font-size: 18px;
  font-weight: 700;
  margin-left: 10px;
}
</style>