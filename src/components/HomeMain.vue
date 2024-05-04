<template>
    <div class="top-text-container">
      <p>基于深度强化学习的边缘计算任务卸载算法模拟平台</p>
    </div>
    <div class="table-container">
      <el-descriptions v-if="isLoading" title="实时计算结果展示中......">
          <el-descriptions-item label="加权和计算速率："><el-tag size="small">{{ vMax }}</el-tag></el-descriptions-item>
          <el-descriptions-item label="所能达到的最大计算速率："><el-tag size="small">{{ actuallyVMax }}</el-tag></el-descriptions-item>
          <el-descriptions-item label="计算耗时："><el-tag size="small">{{ computingTime }}</el-tag></el-descriptions-item>
          <el-descriptions-item label="卸载决策："><el-tag size="small">{{ offloadingStrategy }}</el-tag></el-descriptions-item>
          <el-descriptions-item label="时间分配："><el-tag size="small">{{ timeAllocation }}</el-tag></el-descriptions-item>
      </el-descriptions>

      <el-progress v-if="isLoading" :percentage="startTime"></el-progress>
      <el-table v-if="showTable" :data="tableData" boder style="width: 100%">
        <el-table-column prop="channel" label="信道增益" align="center"></el-table-column>
        <el-table-column prop="actuallyVMax" label="所能达到的最大计算速率" align="center"></el-table-column>
      </el-table>
      <div class="pagination-container">
        <el-pagination v-if="showTable"
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page="page"
          :page-sizes="[5]"
          :page-size="limit"
          background
          layout="total, sizes, prev, pager, next, jumper"
          :total="total">
        </el-pagination>
        <el-button @click="startSimulation" :loading="isLoading" type="primary" round>开始模拟</el-button>
        <el-button v-if="isLoading" @click="offSimulation" type="danger" round>终止模拟</el-button>
      </div>
    </div>
    <div class="container">  
        <div id="main"></div>  
        <div id="main1"></div>  
    </div>
</template>


<script>
import axios from 'axios';
import * as echarts from "echarts";
export default {
  data() {
    return {
      tableData: [],
      page: 1,
      limit: 5,
      total: 30000,
      webSocket: null,
      showTable: true,
      isLoading: false,
      UUID: "zhs",
      startTime: 0,
      vMax: "-",
      actuallyVMax: "-",
      computingTime: "-",
      offloadingStrategy: "-",
      timeAllocation: "-",
      base: 0,
      timeFrame: [],
      data: [],
      chartCostTime: null,
      data1: [],
      chartRate: null,
    }
  },
  created() {
    let result = '';
    const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    const charactersLength = characters.length;
    for (let i = 0; i < 5; i++) {
      result += characters.charAt(Math.floor(Math.random() * charactersLength));
    }
    this.UUID = result;
    this.setupWebSocket(); // 创建WebSocket连接
  },
  mounted() {
    axios.post('http://127.0.0.1:8888/data/getData')
    .then(res => {
      this.tableData = [...res.data.data.data]
      // this.total = res.data.data.data.length
    })
    this.init();
    this.init1();
  },
  beforeUnmount() {
    this.closeWebSocket(); // 在组件销毁前关闭WebSocket连接
  },
  methods: {
    handleCurrentChange(val) {
      axios.post('http://127.0.0.1:8888/data/getData')
      .then(res => {
        this.tableData = [...res.data.data.data]
        this.page = val
        // this.total = res.data.data.data.length
      })
    },
    handleSizeChange(val) {
      console.log(val)
    },
    setupWebSocket() {
      const name = this.UUID;
      const websocketUrl = `ws://127.0.0.1:8888/springboot/backend/getStrategy/${name}`;
      this.websocket = new WebSocket(websocketUrl); // 创建WebSocket连接
      this.websocket.onopen = this.onWebSocketOpen; // WebSocket连接打开时的处理函数
      this.websocket.onmessage = this.onWebSocketMessage; // 收到WebSocket消息时的处理函数
      this.websocket.onclose = this.onWebSocketClose; // WebSocket连接关闭时的处理函数
    },
    onWebSocketOpen() {
      console.log("链接成功");
    },
    onWebSocketMessage(event) {
      const message = event.data;
      // 处理从服务器接收的消息
      console.log('Received message:', message);
      // 解析 JSON 字符串为 JavaScript 对象
      const jsonObject = JSON.parse(message);

      this.vMax = jsonObject.VMax;
      this.actuallyVMax = jsonObject.actuallyVMax;
      this.computingTime = jsonObject.computingTime;
      this.offloadingStrategy = jsonObject.offloadingStrategy;
      this.timeAllocation = jsonObject.timeAllocation;

      let rate = parseFloat(jsonObject.VMax) / parseFloat(jsonObject.actuallyVMax);
      this.updateCostTimeGraph(jsonObject.computingTime);
      this.updateRateGraph(rate.toFixed(5));
      // 在这里您可以根据消息的内容执行不同的操作，例如更新界面、触发事件等
    },
    closeWebSocket() {
      if (this.websocket) {
        this.websocket.close(); // 关闭WebSocket连接
      }
    },
    startSimulation() {
      // 这里编写点击按钮时的逻辑
      axios.post('http://127.0.0.1:8888/drl/startSimulate', {
        "name" : this.UUID
      })
      .then(res => {  
        console.log(res)
      })
      console.log('开始模拟');
      this.startTime = 0;
      this.showTable = false;
      this.isLoading = true;
      const intervalId = setInterval(() => {
        this.startTime = parseFloat(this.startTime) + 1 / 30000;
        if (this.variable >= 100) {
          clearInterval(intervalId);
        }
      }, 2000);
    },
    offSimulation() {
      // 这里编写点击按钮时的逻辑
      console.log('终止模拟');
      location.reload(true);
      // this.showTable = true;
      // this.isLoading = false;
    },
    //初始化函数
    init() {
      for(var i = 1; i <= 100; i++) {
        this.timeFrame.push(0);
        this.data.push(0);
      }
      // 基于准备好的dom，初始化echarts实例
      this.chartCostTime = echarts.init(document.getElementById("main"));
      // 绘制图表
      let options = {
        xAxis: {
          type: 'category',
          boundaryGap: false,
          data: this.timeFrame,
          name: '时间帧', // 添加 x 轴标题
        },
        yAxis: {
          boundaryGap: [0, '50%'],
          type: 'value',
          name: '计算耗时(s)', // 添加 y 轴标题
          min: 0, // 设置纵轴的最小值为0.9  
          max: 0.05,   // 设置纵轴的最大值为1 
        },
        series: [
          {
            name: '成交',
            type: 'line',
            smooth: true,
            symbol: 'none',
            stack: 'a',
            areaStyle: {
              normal: {}
            },
            data: this.data
          }
        ]
      };
      // 渲染图表
      this.chartCostTime.setOption(options);
    },
    updateCostTimeGraph(costTime) {
      this.base++;
      this.timeFrame.push(this.base);
      this.data.push(costTime);
      this.timeFrame.shift();
      this.data.shift();
      this.chartCostTime.setOption({
        xAxis: {
          data: this.timeFrame
        },
        series: [
          {
            name: '成交',
            data: this.data
          }
        ]
      });
    },
    //初始化函数
    init1() {
      for(var i = 1; i <= 100; i++) {
        this.data1.push(0);
      }
      // 基于准备好的dom，初始化echarts实例
      this.chartRate = echarts.init(document.getElementById("main1"));
      // 绘制图表
      let options = {
        xAxis: {
          type: 'category',
          boundaryGap: false,
          data: this.timeFrame,
          name: '时间帧', // 添加 x 轴标题
        },
        yAxis: {
          boundaryGap: [0, '0%'],
          type: 'value',
          name: '归一化计算速率', // 添加 y 轴标题
          min: 0.95, // 设置纵轴的最小值为0.9  
          max: 1,   // 设置纵轴的最大值为1 
        },
        series: [
          {
            name: '成交',
            type: 'line',
            smooth: true,
            symbol: 'none',
            stack: 'a',
            areaStyle: {
              normal: {}
            },
            data: this.data1
          }
        ]
      };
      // 渲染图表
      this.chartRate.setOption(options);
    },
    updateRateGraph(rate) {
      this.data1.push(rate);
      this.data1.shift();
      this.chartRate.setOption({
        xAxis: {
          data: this.timeFrame
        },
        series: [
          {
            name: '成交',
            data: this.data1
          }
        ]
      });
    },

  }
}
</script>


<style scoped>
.table-container {
  border: 2px solid #ebeef5; /* 增加边框宽度 */
  border-radius: 5px;
  padding: 10px;
}
.pagination-container {
  display: grid;
  grid-template-columns: auto auto; /* 将容器分为两列 */
  align-items: center; /* 垂直居中 */
  margin-top: 10px; /* 设置上边距 */
}

.top-text-container {
  display: flex;
  justify-content: center; /* 水平居中 */
  align-items: center; /* 垂直居中 */
  height: 30px; /* 设置高度 */
}

.top-text-container p {
  font-size: 24px; /* 设置字体大小 */
}

.container {  
  display: flex;  
  justify-content: space-between; /* 可选，如果你希望元素之间有空间 */  
}  
  
#main, #main1 {  
  width: 50%;  
  height: 50vh;  
  box-sizing: border-box; /* 推荐，包含padding和border在宽度计算内 */  
}

</style>