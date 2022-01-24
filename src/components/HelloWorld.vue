<template>
  <div class="hello-pomodoro">
    <!-- 上 -->
    <div class="top-part">
      <div class="title">
        <div class="title-en">Countdown Timer</div>
        <div class="title-ch">之番茄工作法</div>
      </div>
      <div class="countdown" :class="{ 'break-time': isBreakTime }">
        {{ countDownTimer }}
      </div>
      <el-progress
        :text-inside="true"
        :show-text="false"
        :stroke-width="18"
        :percentage="percentage"
        status="success"
        class="progress"
      ></el-progress>
    </div>

    <!-- 中 -->
    <div class="middle-part">
      <el-input
        type="textarea"
        v-model="taskName"
        @input="inputHandle"
        :rows="1"
        :disabled="isInput"
        placeholder="请输入20个字符以内的任务标题"
      />
      <div class="start-break-stop">
        <el-button
          type="primary"
          class="work"
          :disabled="!taskName || !workTime || percentage !== 100"
          @click="
            () => {
              settingValid();
              startWorkTime();
            }
          "
          >开始工作</el-button
        >
        <el-button
          type="success"
          @click="timePause"
          :disabled="interval === null"
          >暂停/继续</el-button
        >
        <el-button type="danger" @click="stopTask" :disabled="interval === null"
          >停止</el-button
        >
      </div>
    </div>

    <!-- 下 -->
    <div class="bottom-part">
      <div class="setting">
        <el-button type="primary" @click="showSetting">设置</el-button>
        <div v-show="isSetting" class="setting-content">
          <div>
            循环开关 <el-switch v-model="settingParams.isLoop"></el-switch>
          </div>
          <div class="input-suffix work-time">
            <span>工作时长</span>
            <el-input-number
              v-model="settingParams.workTime"
              @change="setTimer"
              :min="1"
              :max="60"
            ></el-input-number>
          </div>
          <div class="input-suffix">
            <span>休息时长</span>
            <el-input-number
              v-model="settingParams.breakTime"
              @change="setTimer"
              :min="1"
              :max="60"
            ></el-input-number>
          </div>
        </div>
      </div>
      <div class="history">
        <div class="history-btn">
          <el-button type="primary" @click="showHistoryList"
            >任务记录</el-button
          >
          <el-button type="danger" @click="clearAll">清空记录</el-button>
        </div>
        <ul v-show="showHistory">
          <li class="stop-clicked" v-for="item in historyList" :key="item.id">
            任务：{{ item.task }}, 时间：{{
              item.startTime + "-" + item.endTime
            }}，停止：{{ item.stop }}, 是否完成：{{ item.done }}
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
      taskName: "", // 任务标题输入框内容
      workTime: 0, // 已生效的工作时间
      countDownTimer: "00:00", // 显示时间
      breakTime: 0, // 已生效的休息时间
      handleTime: 0, // 获得秒数
      interval: null, // 定时器名字
      isSetting: false, // 是否显示设置内容
      isInput: false, // 是否禁用输入框
      showHistory: false, // 是否显示任务记录内容
      isBreakTime: false, // 是否显示灰色时间
      percentage: 100, // 进度条
      isLoop: false, // 是否循环
      isLoopText: "",
      historyList: JSON.parse(localStorage.getItem("historyList")) || [],
      settingParams: {
        isLoop: false,
        workTime: 25,
        breakTime: 5,
      },
    };
  },
  mounted() {
    this.refreshPage();
    window.addEventListener("beforeunload", this.beforeUnloadHandle);
  },
  beforeDestroy() {
    this.beforeUnloadHandle(); // 切换侧边栏组件会卸载组件,处理等同于刷新问题
  },
  watch: {
    countDownTimer() {
      document.title =
        this.countDownTimer +
        `${this.isBreakTime ? " 休息倒计时" : " 工作倒计时"}`;
    },
  },
  methods: {
    // momemnt格式化时间
    formatTime(seconds) {
      // +1是因为moment内部处理的时候使用的是向下取整,所以导致在最后一秒的时候,都显示为00:00,导致00:00被显示了1秒的时间,所以为了修正倒计时的显示问题,需要在时间上+1,只是在倒计时显示的时候+1,真正判断时间差并没有变化
      const remainingTime = moment.duration(seconds + 1, "seconds");
      // 将秒转化为分秒
      this.countDownTimer = moment({
        m: remainingTime.minutes(),
        s: remainingTime.seconds(),
      }).format("mm:ss");
    },
    // 处理页面刷新
    refreshPage() {
      const timeStatus = JSON.parse(window.localStorage.getItem("timeStatus"));
      const settingStatus = JSON.parse(
        window.localStorage.getItem("settingStatus")
      );
      // 解决首次打开settingStatus为undefined的情况;
      if (settingStatus) {
        const { isLoop, workTime, breakTime } = settingStatus;
        this.settingParams = settingStatus;
        // 刷新之后强行将设置参数生效;
        this.isLoop = isLoop || false;
        this.workTime = workTime || 25;
        this.breakTime = breakTime || 5;
      }
      // 保留刷新之前的进度条状态;
      if (timeStatus) {
        const {
          isBreakTime,
          percentage,
          taskName,
          countDownTimer,
          handleTime,
        } = timeStatus;
        this.isBreakTime = isBreakTime;
        this.percentage = percentage;
        this.taskName = taskName;
        this.interval = 0;
        this.countDownTimer = countDownTimer;
        this.handleTime = handleTime;
        // 将秒转化为分秒
        // const res = this.formatTime(handleTime);
        // // 刷新之后直接开始继续任务
        // document.title = res;
        // console.log("res", res);
        this.timePause();
        window.localStorage.removeItem("timeStatus");
      }
    },

    // 处理倒计时历史记录
    historyHandle(isDone) {
      const historyList = this.historyList;
      historyList[historyList.length - 1].done = isDone;
      historyList[historyList.length - 1].isClear = true;
      historyList[historyList.length - 1].endTime = moment(Date.now()).format(
        "HH:mm"
      );
      historyList[historyList.length - 1].stop =
        "20" + moment(Date.now()).format("YY/MM/DD HH:mm");
      this.historyList = historyList;
      localStorage.setItem("historyList", JSON.stringify(historyList));
    },

    // 倒计时和进度条的状态停止重置
    resetStatus() {
      clearTimeout(this.interval);
      this.countDownTimer = "00:00";
      this.percentage = 100;
      this.interval = null;
      this.taskName = this.isLoopText;
      this.isInput = false;
    },

    // 处理进度条
    handleProgress() {
      if (this.isBreakTime) {
        this.percentage = (this.handleTime / (this.breakTime * 60)) * 100;
      } else {
        this.percentage = (this.handleTime / (this.workTime * 60)) * 100;
      }
    },

    // 处理倒计时走完
    isCountDownDone() {
      // 倒计时走完
      this.resetStatus();
      if (this.isBreakTime === false) {
        // 说明是从工作时间进入休息时间
        this.formatTime(this.breakTime * 60 - 1);
        // 结束工作时间,需要进入休息时间
        this.taskName = "休息一下";
        this.isBreakTime = true;
        this.historyHandle(true);
        this.countDown(this.breakTime * 60 * 1000 + Date.now());
        this.workTimeNotification("恭喜您完成当前番茄钟！下面进入休息时间");
      } else {
        // 休息走完,需要循环或者停止
        this.isBreakTime = false;
        this.isInput = false;
        this.workTimeNotification("休息时间总是很短暂，让我们开始工作吧");
        // 判断是否循环
        if (this.isLoop) {
          this.taskName = this.isLoopText;
          this.startWorkTime();
          this.formatTime(this.workTime * 60 - 1);
        }
      }
    },

    countDown(endTime) {
      // 处理倒计时和进度条的函数
      const countDownWorkerOrBreakTime = () => {
        // 当前毫秒数
        let newTime = Date.now();
        // 如果时间未结束，对时间进行处理
        if (endTime - newTime > 0) {
          /* 秒 */
          this.handleTime = (endTime - newTime) / 1000;
          this.formatTime((endTime - newTime) / 1000);
          // 执行进度条
          this.handleProgress();
          this.interval = setTimeout(countDownWorkerOrBreakTime, 1000);
        } else {
          // 倒计时走完
          this.isCountDownDone();
        }
      };
      this.interval = setTimeout(countDownWorkerOrBreakTime, 1000);
    },

    // 处理刷新丢失问题
    beforeUnloadHandle() {
      // 判断是否处于读条状态
      if (this.percentage < 100) {
        const timeStatus = {
          percentage: this.percentage,
          isBreakTime: this.isBreakTime,
          taskName: this.taskName,
          countDownTimer: this.countDownTimer,
          handleTime: this.handleTime,
        };
        window.localStorage.setItem("timeStatus", JSON.stringify(timeStatus));
      }
      // 保留用户的设置时间
      window.localStorage.setItem(
        "settingStatus",
        JSON.stringify(this.settingParams)
      );
      localStorage.setItem("historyList", JSON.stringify(this.historyList));
      // 卸载beforeunload事件
      window.removeEventListener("beforeunload", this.beforeUnloadHandle);
    },

    // 是否打开设置按钮
    showSetting() {
      this.isSetting = !this.isSetting;
    },

    // 是否打开历史记录
    showHistoryList() {
      this.showHistory = !this.showHistory;
    },

    // 暂停/继续
    timePause() {
      if (this.interval) {
        clearTimeout(this.interval);
        this.interval = 0;
      } else {
        this.countDown(Date.now() + this.handleTime * 1000);
        const historyList = this.historyList;
        historyList[historyList.length - 1].endTime = moment(
          Date.now() + this.handleTime * 1000
        ).format("HH:mm");
        historyList[historyList.length - 1].stop = moment(
          Date.now() + this.handleTime * 1000
        ).format("YY:MM:DD HH:mm");
        this.historyList = historyList;
      }
    },

    // taskName 的onchange事件
    inputHandle(val) {
      this.taskName = val.trim();
    },

    // 设置时长input框
    setTimer() {
      this.$message.success("时长设置成功");
    },

    // 让设置的参数生效: 只有点击开始工作或者刷新页面才能使设置参数生效;
    settingValid() {
      const { isLoop, workTime, breakTime } = this.settingParams;
      this.isLoop = isLoop;
      this.workTime = workTime;
      this.breakTime = breakTime;
    },

    // 开始工作
    startWorkTime() {
      // 处理点击开始后的时间
      this.formatTime(this.workTime * 60 - 1); // 因为format内部+1,-1修正时间显示
      // 循环时,保留第一次的任务名称;
      // if (this.isLoop) {
      this.isLoopText = this.taskName;
      // }
      const localHistoryList =
        JSON.parse(localStorage.getItem("historyList")) || [];
      if (localHistoryList.length >= 50) {
        return this.$message.warning("请清空记录再开始");
      }
      const startTime = Date.now();
      this.isBreakTime = false;
      this.isInput = true;
      clearTimeout(this.interval);
      this.countDown(startTime + this.workTime * 60 * 1000);
      const historyList = [
        ...this.historyList,
        {
          id: Date.now().toString(16),
          task: this.taskName,
          startTime: moment(startTime).format("HH:mm"),
          endTime: moment(startTime + this.workTime * 60 * 1000).format(
            "HH:mm"
          ),
          stop:
            "20" +
            moment(startTime + this.workTime * 60 * 1000).format(
              "YY/MM/DD HH:mm"
            ),
          done: false,
          isClear: false,
        },
      ];
      localStorage.setItem("historyList", JSON.stringify(historyList));
      this.historyList = historyList;
    },

    // 清空记录
    clearAll() {
      let historyList = JSON.parse(localStorage.getItem("historyList")) || [];
      historyList = historyList.filter((item) => !item.isClear);
      localStorage.setItem("historyList", JSON.stringify(historyList));
      // 里面的数组赋值给外面的数组，以达到更新视图的效果
      this.historyList = historyList;
    },

    // 停止任务
    stopTask() {
      this.historyHandle(false);
      this.resetStatus();
      this.isBreakTime = false;
    },

    // 全局提示
    workTimeNotification(message) {
      this.$notify({
        title: this.isBreakTime ? "番茄钟完成" : "休息完成",
        message,
        type: "success",
        duration: 5000,
      });
    },
  },
};
</script>

<style scoped>
.hello-pomodoro {
  width: 600px;
  height: 100%;
  min-height: 600px;
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

.progress {
  margin: 0 10px 20px;
}

/* middle */
.middle-part >>> .el-textarea__inner {
  margin: 0 10px;
  width: 580px;
  text-align: center;
  font-size: 20px;
}

.setting-content {
  margin-left: 15px;
}
.setting-content >>> .el-input-number .el-input__inner {
  -webkit-appearance: none;
  width: 113px;
  padding-left: 5px;
  text-align: left;
}
.input-suffix {
  display: flex;
  align-items: center;
  width: 260px;
  margin: 10px 0;
}
.input-suffix span {
  margin-right: 5px;
}
.input-suffix .el-input {
  width: 100px;
  margin-left: 20px;
}
>>> .el-input-number__decrease {
  left: 72px;
  border-top-right-radius: 4px;
  border-bottom-right-radius: 4px;
  border-right: 0;
}

>>> .el-input-number__increase {
  left: 32px;
}

ul li {
  padding: 3px;
  padding-left: 10px;
  margin: 5px;
  background-color: #0dbc79;
  color: #fff;
  border-radius: 5px;
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
