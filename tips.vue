<template>
  <a-layout style="height: 100vh; padding: 20px;">
    <!-- 主体内容区域 -->
    <a-layout>
      <div class="fixed-tag">欢迎使用调度系统AI助手</div>
      <a-layout-content ref="contentRef" style="flex: 1; overflow-y: auto; padding: 20px;">
        <div v-for="(message, index) in messages" :key="message.id" class="message-box">
          <div :class="['message-content', message.role === 'user' ? 'user-message' : 'assistant-message']">
            <template v-if="message.role === 'user'">
              {{ message.content }}
            </template>
            <template v-else>
              <a-spin v-if="message.isLoading" class="loading-spinner" />
              <div v-else>
                <template v-if="message.chartData">
                  <a-table
                          :dataSource="message.chartData.tableData"
                          :columns="message.chartData.tableColumns"
                          bordered
                          :scroll="{ x: 'max-content' }"
                          style="width: 100%; margin-bottom: 20px;"
                  />
                  <div class="charts-container" v-if="message.chartData?.chartConfig?.locationOption !== null">
                    <h3 style="margin-top: 20px;">数据可视化</h3>
                    <div :id="'barChart-' + message.id" style="width: 100%; height: 300px;"></div>
                    <div class="pie-container">
                      <div :id="'locationChart-' + message.id" style="width: 50%; height: 350px;"></div>
                      <div :id="'legendChart-' + message.id" style="width: 50%; height: 350px;"></div>
                    </div>
                  </div>
                </template>
                <div v-else>
                  {{ message.content || '暂无内容' }}
                </div>
              </div>
            </template>
          </div>
        </div>
      </a-layout-content>

      <!-- 输入框区域 -->
      <a-layout-footer style="padding: 0; background: transparent; margin-left: 1%;">
        <a-radio-group v-model:value="selectedOption" style="margin-bottom: 10px;">
          <a-radio value="智能问答">智能问答</a-radio>
          <a-radio value="数据库查询">数据库查询</a-radio>
          <a-radio value="调度系统">调度系统</a-radio>
        </a-radio-group>
        <div style="display: flex; align-items: center;">
          <a-input
                  v-model:value="inputValue"
                  placeholder="请输入问题"
                  @pressEnter="handleSend"
                  style="border-radius: 20px;"
          >
            <template #suffix>
              <a-button type="primary" @click="handleSend">发送</a-button>
            </template>
          </a-input>
          <a-button @click="handleVoiceInput" style="margin-left: 10px;">语音</a-button>
        </div>
      </a-layout-footer>
    </a-layout>
  </a-layout>
</template>

<script setup>
  import { ref, computed, onMounted, nextTick, onBeforeUnmount } from 'vue';
  import { Spin, Button, List, Radio } from 'ant-design-vue';
  import * as echarts from 'echarts';
import { BarChart } from 'echarts/charts'; // 确保导入BarChart
import { GridComponent, TooltipComponent } from 'echarts/components';

use([CanvasRenderer, BarChart]); // 注册BarChart
  //添加查询逻辑
    import { Input, TableColumnsType, Select } from 'ant-design-vue';
    import { use } from 'echarts/core';
    import { CanvasRenderer } from 'echarts/renderers';
    import { LineChart } from 'echarts/charts';
    import axios from 'axios';
    import { message } from 'ant-design-vue';
    // 注册必须的组件
    use([CanvasRenderer, LineChart, GridComponent, TooltipComponent]);
    const inputValue = ref(''); //绑定数据，出现在输入框的默认栏里
    const viewType = ref('table'); // 决定使用表格展示，还是图表展示数据，默认为表格视图
    const db_id = ref('perpetrator');
    const isSpinning = ref(false); // True为显示查询进度条和叉车前进动画
    const selectedColumn = ref('');
    const dataSource = ref([]);
    const columns = ref([]);
    const chartSize = 20;
    const recognition = ref(null);
    const isRecording = ref(false); // 是否正在录音
    const barChartRef = ref(null);
    const legendChartDiv = ref(null);
    const locationChartDiv = ref(null);
    const contentRef = ref(null);

        // 新增滚动到底部方法
        const scrollToBottom = () => {
          nextTick(() => {
            if (contentRef.value?.$el) {
              const container = contentRef.value.$el;
              container.scrollTop = container.scrollHeight;
            }
          });
        };
// 新增渲染key
const renderKey = ref(0);
// 新增图表实例引用
const barChartInstance = ref(null);
const locationPieInstance = ref(null);
const legendPieInstance = ref(null);

const initLocationPieChart = (totals) => {
    const option = {
          tooltip: {
            trigger: 'item',
            formatter: '{a} <br/>{b} : {c} ({d}%)',
          },
          legend: {
            data: Object.keys(totals),
            orient: 'vertical',
            left: 'left',
            textStyle: {
              color: '#333',
              fontSize: chartSize,
            },
            formatter: (name) => {
              const index = Object.keys(totals).findIndex(label => label === name);
              return `${name}: ${totals[name]}`;
            },
          },
          series: [
            {
              name: 'Total by Location',
              type: 'pie',
              radius: '75%',
              center: ['50%', '50%'],
              label: {
                fontSize: chartSize
              },
              data: Object.keys(totals).map(key => ({
                value: totals[key],
                name: key,
              })),
            },
          ],
        };
    return option;
  };

const initLegendPieChart = (totals) => {
    const option = {
          tooltip: {
            trigger: 'item',
            formatter: '{a} <br/>{b} : {c} ({d}%)',
          },
          legend: {
            data: Object.keys(totals),
            orient: 'vertical',
            right: 'right',
            textStyle: {
              color: '#333',
              fontSize: chartSize,
            },
            formatter: (name) => {
              const index = Object.keys(totals).findIndex(label => label === name);
              return `${name}: ${totals[name]}`;
            },
          },
          series: [
            {
              name: 'Total by Legend',
               label: {
                 fontSize: chartSize
              },
              type: 'pie',
              radius: '75%',
              center: ['50%', '50%'],
              data: Object.keys(totals).map(key => ({
                value: totals[key],
                name: key,
              })),
            },
          ],
        };
    return option;
  };
// 修改后的initBarChart函数
const initBarChart = (xData, seriesData) => {
    const option = {
              legend: {
                data: seriesData.map(series => series.name),
                top: '0%',
                textStyle: {
                  color: '#333',
                  fontSize: 14,
                },
                selectedMode: 'multiple',
              },
              xAxis: {
                type: 'category',
                data: xData,
                axisLabel: {
                  interval: 0,
                },
                axisTick: {
                  alignWithLabel: true,
                },
              },
              yAxis: {
                type: 'value',
              },
              series: seriesData.map(series => ({
                name: series.name,
                type: 'bar',
                data: series.data,
              })),
            }; // 删除了这里的多余逗号
    return option;
};

const handleSend = async () => {
  if (!inputValue.value.trim()) return;

  // 添加用户消息
  const userMessage = {
    role: 'user',
    content: inputValue.value,
    id: messageId++,
    chartData: null
  }
  messages.value.push(userMessage);

  // 添加加载中的助手消息
  const assistantMessage = {
    role: 'assistant',
    content: '',
    isLoading: true,
    id: messageId++,
    chartData: null,
    charts: {
      bar: null,
      location: null,
      legend: null
    }
  }
  messages.value.push(assistantMessage);

  try {
    let response;
    if (selectedOption.value === '数据库查询'){
    response = await onSearch(inputValue.value);
                    // 更新消息数据
                        const updatedMessage = {
                          ...assistantMessage,
                          isLoading: false,
                          // 只有数据库查询需要图表数据
                          chartData: (selectedOption.value === '数据库查询' && response.success) ? {
                            tableData: response.tableData,
                            tableColumns: response.tableColumns,
                            chartConfig: response.chartConfig
                          } : null,
                          content: response.success ?
                            (selectedOption.value === '数据库查询' ? '查询结果如下' : response.content) :
                            response.error
                        }

                        // 替换原消息
                        messages.value.splice(messages.value.length - 1, 1, updatedMessage);
                        // 只有数据库查询需要初始化图表
                        if (selectedOption.value === '数据库查询' && response.success && response.chartConfig?.locationOption !== null) {
                          nextTick(() => {
                            initChartsForMessage(updatedMessage);
                          });
                        }
                    }
    if (selectedOption.value === '智能问答'){
        response = {
                  success: true,
                  content: '这是智能问答的输出：' + inputValue.value + ' 的相关解答'
                };
    }
        if (selectedOption.value === '调度系统'){
            response = {
                      success: true,
                      content: '这是调度系统的输出：' + inputValue.value + ' 的相关解答'
                    };
        }
    resetState();
    scrollToBottom(); // 新增滚动
  } catch (error) {
    console.error('请求失败:', error);
    messages.value.splice(messages.value.length - 1, 1, {
      ...assistantMessage,
      isLoading: false,
      content: '请求失败，请稍后重试'
    });
    scrollToBottom(); // 新增滚动
  }
}

// 初始化图表
const initChartsForMessage = (message) => {
  if (!message.chartData?.chartConfig) return

  destroyChartsForMessage(message)

  const { chartConfig } = message.chartData

  message.charts.bar = echarts.init(document.getElementById(`barChart-${message.id}`))
  message.charts.location = echarts.init(document.getElementById(`locationChart-${message.id}`))
  message.charts.legend = echarts.init(document.getElementById(`legendChart-${message.id}`))

  message.charts.bar.setOption(chartConfig.barOption)
  message.charts.location.setOption(chartConfig.locationOption)
  message.charts.legend.setOption(chartConfig.legendOption)
}

// 销毁图表
const destroyChartsForMessage = (message) => {
  Object.values(message.charts).forEach(instance => {
    if (instance && !instance.isDisposed) {
      instance.dispose()
    }
  })
}
// 查询逻辑
const onSearch = async (question) => {
        let response;

        try {
            // 仅包裹 axios 请求
             response = await axios.post('http://127.0.0.1:5003/axios', {
                question,
                db_id: 'perpetrator'
            });
        } catch (error) {
            return {
                    success: false,
                    error: '服务器连接失败，请检查网络连接'
                  }
        }
// 创建新数组避免引用
        const rawTableData = response.data[1];
        const rawTableColumns = response.data[0];

       tableData.value = response.data[1]
       tableColumns.value = response.data[0]
       // 空结果检测
       if (tableData.value.length === 0 || tableColumns.value.length === 0) {
           console.log('tableData.value.length === 0')
           return {
               success: false,
               error: '查询结果为空，请换个问法'
           };
       }
       selectedColumn.value = tableColumns.value[0]['key'];

       const { xData, seriesData } = prepareDataForSelectedColumn(tableColumns.value, tableData.value, selectedColumn.value);
if (seriesData.length === 0) {
            return {
                    success: true,
                    tableData: rawTableData,
                    tableColumns: rawTableColumns,

                    chartConfig: {
                    barOption: null,
                    locationOption: null,
                    legendOption: null
                    }
                  }
      } else {
          return {
                  success: true,
                    tableData: rawTableData,
                    tableColumns: rawTableColumns,

                  chartConfig: {
                  barOption: initBarChart(xData, seriesData),
                  locationOption: initLocationPieChart(getLocationTotals(tableColumns.value, tableData.value, selectedColumn.value)),
                  legendOption: initLegendPieChart(getLegendTotals(tableColumns, tableData.value))
                  }
                }
      }

}

// 生成各图表配置的函数（示例）
const generateBarOption = (columns, data) => {
  // 实际数据处理逻辑
  return { /*...*/ };
};

// 组件卸载时清理所有图表
onBeforeUnmount(() => {
  messages.value.forEach(msg => {
    if (msg.role === 'assistant') {
      destroyChartsForMessage(msg);
    }
  });
});
   let messageId = 0;


const resetState = () => {
    inputValue.value = ''
};


const prepareDataForSelectedColumn = (columns, dataSource, selectedColumn) => {
    const xData = dataSource.map(item => item[selectedColumn]); // 使用选定列作为X轴数据
    const seriesData = columns.filter(column =>
      typeof dataSource[0][column.dataIndex] === 'number' && !column.dataIndex.includes('ID') && column.dataIndex !== 'Year' && column.dataIndex !== selectedColumn
    ).map(column => ({
      name: column.title,
      data: dataSource.map(item => item[column.dataIndex])
    }));

    return { xData, seriesData };
  };

const getLocationTotals = (columns, dataSource, selectedColumn) => {
    const totals = {};
    dataSource.forEach(item => {
      const value = item[selectedColumn];
      totals[value] = totals[value] || 0;
      columns.filter(column => typeof dataSource[0][column.dataIndex] === 'number' && !column.dataIndex.includes('ID') && column.dataIndex !== 'Year' && column.dataIndex !== selectedColumn).forEach(column => {
        totals[value] += item[column.dataIndex];
      });
    });
    return totals;
  };

  const getLegendTotals = (columns, dataSource) => {
    const totals = {};
    // 过滤掉被选为横轴的列
    const filteredColumns = columns.value.filter(column =>
      typeof dataSource[0][column.dataIndex] === 'number' &&
      !column.dataIndex.includes('ID') &&
      column.dataIndex !== 'Year' &&
      column.dataIndex !== selectedColumn.value // 排除被选为横轴的列
    );

    filteredColumns.forEach(column => {
      totals[column.title] = dataSource.reduce((acc, cur) => acc + cur[column.dataIndex], 0);
    });
    return totals;
  };













const selectedOption = ref('数据库查询');
const messages = ref([]);
const barcharts = ref([]);
const tableData = ref([
  { key: '1', name: '张三', age: 32, score: 85 },
  { key: '2', name: '李四', age: 28, score: 92 },
  { key: '3', name: '王五', age: 45, score: 78 },
]);
const tableColumns = ref([
  { title: '姓名', dataIndex: 'name', key: 'name' },
  { title: '年龄', dataIndex: 'age', key: 'age' },
  { title: '分数', dataIndex: 'score', key: 'score' },
]);

// 图表实例映射表
const chartInstances = new Map();



const generatePieData = () => {
  const ageData = [
    { value: tableData.value.filter(item => item.age < 30).length, name: '30岁以下' },
    { value: tableData.value.filter(item => item.age >= 30 && item.age < 40).length, name: '30-39岁' },
    { value: tableData.value.filter(item => item.age >= 40).length, name: '40岁及以上' }
  ];

  const scoreData = [
    { value: tableData.value.filter(item => item.score >= 90).length, name: '优秀（≥90）' },
    { value: tableData.value.filter(item => item.score >= 80 && item.score < 90).length, name: '良好（80-89）' },
    { value: tableData.value.filter(item => item.score < 80).length, name: '及格（<80）' }
  ];

  return { ageData, scoreData };
};

const renderBarChart = (dom) => {
  if (!dom) return;

  try {
    const chartInstance = echarts.init(dom);
    chartInstances.set(dom.id, chartInstance);

    const names = tableData.value.map(item => item.name);
    const scores = tableData.value.map(item => item.score);

    const option = {
      title: { text: '分数柱状图', left: 'center' },
      tooltip: { trigger: 'axis' },
      xAxis: {
        type: 'category',
        data: names,
        axisLabel: { rotate: 45 }
      },
      yAxis: { type: 'value', name: '分数' },
      series: [{
        name: '分数',
        type: 'bar',
        data: scores,
        itemStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: '#83bff6' },
            { offset: 0.5, color: '#188df0' },
            { offset: 1, color: '#188df0' }
          ])
        }
      }]
    };

    chartInstance.setOption(option);
  } catch (error) {
    console.error('柱状图初始化失败:', error);
  }
};

const renderPieChart = (dom, title, data, colors) => {
  if (!dom) return;

  try {
    const chartInstance = echarts.init(dom);
    chartInstances.set(dom.id, chartInstance);

    const option = {
      title: { text: title, left: 'center' },
      tooltip: { trigger: 'item' },
      color: colors,
      series: [{
        type: 'pie',
        radius: ['40%', '70%'],
        data: data,
        label: {
          formatter: '{b}: {d}%'
        },
        emphasis: {
          itemStyle: {
            shadowBlur: 10,
            shadowOffsetX: 0,
            shadowColor: 'rgba(0, 0, 0, 0.5)'
          }
        }
      }]
    };

    chartInstance.setOption(option);
  } catch (error) {
    console.error(`${title}饼图初始化失败:`, error);
  }
};
// 空的语音输入处理函数
const handleVoiceInput = () => {
// 语音输入功能待实现
};
</script>

<style scoped>
  .message-box {
    margin-bottom: 20px;
  }
  .fixed-tag {
    font-size: 30px;
  }

  .message-content {
    padding: 15px 25px;
    border-radius: 20px;
    position: relative;
    min-height: 40px;
    transition: all 0.3s ease;
    font-size: 20px;
  }

  .user-message {
    background-color: #1890ff;
    color: white;
    margin-left: auto;
    margin-right: 10%;
    width: fit-content;
    display: flex;
  }

.assistant-message {
  background-color: #ffffff; /* 背景改为白色 */
  color: #333333; /* 文字颜色改为深色 */
  margin-right: auto;
  margin-left: 1%;
  border: 1px solid #e8e8e8; /* 添加边框以增强视觉效果 */
  max-width: 99%;
}
.other-message{
    display: flex;
    width: fit-content;
}
.database-message{

}
  .loading-spinner {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 40px;
  }
/* 确保图表容器在不同屏幕尺寸下表现良好 */
.charts-container {
  display: flex;
  flex-direction: column;
  gap: 20px;
  width: 100%;
}

.bar-chart {
  width: 100%;
  height: 500px;
}

.pie-container {
  display: flex;
  gap: 20px;
  width: 100%;
}

.pie-chart {
  width: 50%; /* 每个饼图占据一半宽度 */
  height: 350px;
}

/* 在小屏幕设备上调整布局 */
@media (max-width: 768px) {
  .pie-container {
    flex-direction: column; /* 将饼图堆叠显示 */
  }

  .pie-chart {
    width: 100%; /* 单独一行时宽度占满 */
    height: 300px; /* 调整高度以适应小屏幕 */
  }
}

  .chart-item {
    background: #fff;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    padding: 15px;
  }

  /* 滚动条样式 */
  ::-webkit-scrollbar {
    width: 8px;
  }

  ::-webkit-scrollbar-track {
    background: #f1f1f1;
    border-radius: 4px;
  }

  ::-webkit-scrollbar-thumb {
    background: #888;
    border-radius: 4px;
  }

  ::-webkit-scrollbar-thumb:hover {
    background: #555;
  }

  .ant-btn-block {
    margin-bottom: 20px;
  }

  /* 单选按钮样式 */
  .ant-radio-wrapper {
    margin-right: 20px;
  }
</style>