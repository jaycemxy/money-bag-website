<template>
<Layout>
  <header class="header">
    <select v-model="type" class="type">
      <option v-for="(t, index) in typeList" :key="index" :value="t.value">{{t.name}}</option>
    </select>
    <!-- <Icon name="triangle"/> -->
    <TabBar class-prefix="interval" :bars="intervalList" :c-bar.sync="interval"/>
  </header>

  <div class="chart">
    <div class="caption">
      <span v-if="interval==='week'">本周</span>
      <span v-else-if="interval==='month'">本月</span>
      <span v-else>今年</span>
    </div>
    <div class="info">
      <div class="total">总支出: ￥{{total}}</div>
      <div class="average">平均值: ￥{{average}}</div>
    </div>
    <div id="figure"></div>
  </div>

  <div class="ranking">
    <div class="caption">
      <span>支出排行榜</span>
    </div>
    <ul class="tag-list" v-if="targetRecords.length > 0">
      <li class="tag-item" v-for="(item,index) in this.groupByTag()" :key="index">
        <div class="tag-info">
          <div class="icon">
            <Icon :name="item.tag.name"/>
          </div>
          <span>{{item.tag.value}}</span>
          <span>{{item.percentage}}%</span>
        </div>
        <div>{{item.amount}}</div>
      </li>
    </ul>
    <NoRecord v-else/>
  </div>

</Layout>
</template>

<script lang="ts">
import Vue from 'vue';
import {Component, Watch} from 'vue-property-decorator';
import Layout from '@/components/Layout.vue';
import Icon from '@/components/Icon.vue';
import echarts from 'echarts';
import TabBar from '@/components/TabBar.vue';
import NoRecord from '@/components/NoRecord.vue';
import dayjs from 'dayjs';
import clone from '@/lib/clone';
import {twoDecimalPlaces} from '@/lib/decimal';

type Group = {
  tag: TagItem;
  amount: number;
  percentage: number;
}

@Component({
  components: {Layout, Icon, TabBar, NoRecord}
})
export default class Chart extends Vue{
  type: '-' | '+' = '-';
  interval: 'week'|'month'|'year' = 'week';

  typeList:TabBarItem[] = [
    {name: '支出', value: '-'},
    {name: '收入', value: '+'}
  ]
  intervalList:TabBarItem[] = [
    {name: '周', value: 'week'},
    {name: '月', value: 'month'},
    {name: '年', value: 'year'}
  ]

  get targetRecords():RecordItem[] {
    const now = dayjs();
    return clone<RecordItem[]>(this.$store.state.recordList)
      .filter(item => item.type === this.type)
      .filter(item => dayjs(item.createAt).isSame(now, this.interval));
  }

  get total() {
    const amounts = [...this.groupByInterval().values()];
    let sum = 0;
    for(let i = 0; i < amounts.length; i++){
      sum += amounts[i];
    }
    return sum;
  }
  get average() {
    const now = dayjs();
    switch(this.interval) {
      case 'week':
        return twoDecimalPlaces(this.total / (now.day()+1) );
      case 'month':
        return twoDecimalPlaces(this.total / now.date());
      case 'year':
        return twoDecimalPlaces(this.total / (now.month()+1) );
      default:
        return 0;
    }
  }
  groupByInterval(): Map<string, number> {
    let result = new Map<string, number>();
    switch (this.interval) {
      case 'week':
        result = this.groupByWeek(this.targetRecords);
        break;
      case 'month':
        result = this.groupByMonth(this.targetRecords);
        break;
      case 'year':
        result = this.groupByYear(this.targetRecords);
        break;
    }
    return result;
  }
  groupByTag(): Group[] {
    const tags: string[] = [];
    let result: Group[] = [];
    let r: RecordItem;
    for (r of this.targetRecords) {
      const index = tags.indexOf(r.tag.name)
      if (index < 0) {
        tags.push(r.tag.name);
        result.push({tag: r.tag, amount: r.amount, percentage: 0});
      } else {
        result[index].amount += r.amount;
      }
    }
    for (let i = 0; i < result.length; i++) {
      result[i].percentage = twoDecimalPlaces(result[i].amount * 100 / this.total);
    }
    result = result.sort((b, a) => a.percentage - b.percentage);
    return result;
  }
  get days() {
    const [year, month] = [dayjs().year(), dayjs().month()];
    const d = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];
    if ((year % 4 === 0 && year % 100 !== 0) || (year % 100 === 0 && year % 400 === 0)) {
      if (month === 1) {
        return 29;
      } else {
        return d[month];
      }
    } else {
      return d[month];
    }
  }
  groupByWeek(records: RecordItem[]): Map<string, number> {
    const keys = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'];
    const result = new Map<string, number>();
    let i: number;
    // 初始化
    for (i = 0; i < keys.length; i++) {
      result.set(keys[i], 0);
    }
    let r: RecordItem;
    for (r of records) {
      const key = keys[dayjs(r.createAt).day()];
      const amount = result.get(key) as number;
      result.set(key, amount + r.amount);
    }
    return result;
  }
  groupByMonth(records: RecordItem[]): Map<string, number> {
    const keys: string[] = [];
    const result = new Map<string, number>();
    let i: number;
    // 初始化
    for (i = 1; i < this.days; i++) {
      keys.push(i.toString());
    }
    for (i = 0; i < keys.length; i++) {
      result.set(keys[i], 0);
    }
    let r: RecordItem;
    for (r of records) {
      const key = dayjs(r.createAt).date().toString();
      const amount = result.get(key) as number;
      result.set(key, amount + r.amount);
    }
    return result;
  }
  groupByYear(records: RecordItem[]): Map<string, number> {
    const keys = ['一月', '二月', '三月', '四月', '五月', '六月', '七月', '八月', '九月', '十月', '十一月', '十二月'];
    const result = new Map<string, number>();
    let i: number;
    // 初始化
    for (i = 0; i < keys.length; i++) {
      result.set(keys[i], 0);
    }
    let r: RecordItem;
    for (r of records) {
      const key = keys[dayjs(r.createAt).month()];
      const amount = result.get(key) as number;
      result.set(key, amount + r.amount);
    }
    return result;
  }
  toArray(value: number, length: number): number[] {
    const result: number[] = [];
    for (let i = 0; i < length; i++) {
      result.push(value);
    }
    return result;
  }

  draw(data: Map<string, number>) {
    // 提取变量
    const x = [...data.keys()];
    const y = [...data.values()];
    const figure = echarts.init(document.getElementById('figure') as HTMLDivElement);
    figure.setOption({
      grid: {
        top: '5%',
        bottom: '10%'
      },
      xAxis: {
        data: x,
        axisTick: {
          interval: 0,
          lineStyle: {opacity: 0}
        },
        axisLabel: {
          interval: 0,
          fontSize: 8,
          color: '#999'
        }
      },
      yAxis: {
        axisLine: {
          lineStyle: {opacity: 0}
        },
        splitLine: {
          lineStyle: {opacity: 0}
        },
        axisLabel: undefined,
        axisTick: undefined,
      },
      series: [
        {
          type: 'line',
          data: y,
        },
        {
          name: '平均线',
          type: 'line',
          data: this.toArray(this.average, x.length),
          symbol: 'none',
          lineStyle: {
            type: 'dashed',
            color: '#999',
            width: 1,
            opacity: 0.5
          }
        }, 
        {
          name: '最大值',
          type: 'line',
          data: this.toArray(Math.max(...y), x.length),
          symbol: 'none',
          lineStyle: {
            color: '#999',
            width: 1,
            opacity: 0.5
          }
        }]
      });
    }

    mounted(){
      this.draw(this.groupByInterval());
    }

    @Watch('type')
    onTypeChange() {
      this.draw(this.groupByInterval());
    }
    @Watch('interval')
    onIntervalChange() {
      this.draw(this.groupByInterval());
    }

}
</script>

<style lang="scss" scoped>
.header {
  position: relative;
  background: #ffda47;
  padding: 4px 0;
  .type{
    font-size: 20px;
    padding: 4px 8px;
  }
  ::v-deep {
    .interval-tab-bar {
      margin: 8px 16px;
      display: flex;
      justify-content: center;
      .interval-tab-bar-item {
        width: 33.3333%;
        border: 1px solid #333;
        font-size: 14px;
        &:first-child {
          border-radius: 4px 0 0 4px;
        }
        &:last-child {
          border-radius: 0 4px 4px 0;
        }
        &.selected {
          background: #333;
          color: #ffda47;
        }
      }
    }
  }
  .icon {
    position: absolute;
    top: 0;
    left: 50%;
    transform: translateX(50%);
    margin: 5px;
    padding: 5px;
  }
}
.chart {
  border-bottom: 1px solid #ddd;
  .caption {
    display: flex;
    flex-direction: column;
    text-align: left;
    span {
      color: #333;
      border-bottom: 1px solid #ddd;
      padding: 8px 16px;
    }
  }
  .info {
    text-align: left;
    color: #999;
    font-size: 14px;
    .total {
      padding: 6px 6px;
    }
    .average {
      padding: 0 6px;
      margin-bottom: 16px;
    }
  }
  #figure {
    width: 100%;
    height: 150px;
  }
}
.ranking {
  .caption {
    text-align: left;
    font-size: 14px;
    padding: 6px 16px;
  }
  .tag-list {
    padding: 6px 16px;
    .tag-item {
      font-size: 14px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 12px 0;
      box-shadow: inset 0 -1px 1px -1px rgba(0, 0, 0, 0.1);
      .tag-info {
        display: flex;
        align-items: center;
        .icon {
          width: 32px;
          height: 32px;
          background: #f5f5f5;
          border-radius: 50%;
          display: flex;
          justify-content: center;
          align-items: center;
          padding: 2px;
        }
        span {
          margin: 0 8px;
        }
      }
    }
  }
}
</style>