<template>
  <div class="container">
    <div class="handler">
      <a v-show="!expandDetail" v-on:click="toggleExpanding">[+] 展開詳細說明</a>
      <a v-show="expandDetail" v-on:click="toggleExpanding">[-] 收起詳細說明</a>
    </div>
    <div v-show="expandDetail" class="alert alert-info">
      <span class="glyphicon glyphicon-pencil" aria-hidden="true"></span>
      <span>
        前置條件：
        <ul>
          <li><a target="_blank" href="http://laws.mol.gov.tw/Chi/FLAW/FLAWDOC03.asp?keyword=&lc1=FL014930%2C+20150701%2C+24&sdate=&edate=&datatype=etype&recordNo=7">勞動 2 字第 0960130677 號 函</a>：原約定月薪給付總額相當於 240 小時者（即「平日每小時工資額」係以月薪總額除以 30 再除以 8 核計者）</li>
          <li>現行勞基法與一例一休的例假日假設為週日，兩例的例假日為週六與週日</li>
          <li>現行勞基法的週六假設為約定不用上班的日子</li>
          <li>一例一休的休息日假設為星期六</li>
          <li>基於以上假設，在計算輪班制度時（例如四班二輪）可能會與實際狀況有誤差</li>
          <li>下面的「額外工資」欄位包含加班費與例假日上班的工資加倍發給</li>
          <li>原有七天假期由於以前的雙週八十四小時工時，所以週休二日的勞工沒有休假，但應映現行法令改為單週工時四十小時，週休二日的勞工也應該要放這七天，詳情請見 <a target="_blank" href="https://youtu.be/4gd2m_73NHE?t=23m13s">有話好說節目的討論</a> 與 <a href="http://law.moj.gov.tw/LawClass/LawSingle.aspx?Pcode=N0030002&FLNO=23" target="_blank">勞基法施行細則 23 條</a></li>
        </ul>
      </span>
    </div>
    <h2>條件設定</h2>
    <p>假設勞工採月薪制，其月薪 <input class="monthly-pay" number type="number" v-model="monthlyPay"> 元，每月總工時為 {{assumingWorkHours}} 計算，平均時薪為 {{hourlyPay.toFixed(2)}} 元。</p>
    <offday-condition
      :reason="reason"
      @reason-changed="reasonChanged">
    </offday-condition>
    <div class="input">
      <label>週一 <input debounce="100" number type="number" min="0" max="24" class="workhours" v-model="workhours[0]"></label>
      <label>週二 <input debounce="100" number type="number" min="0" max="24" class="workhours" v-model="workhours[1]"></label>
      <label>週三 <input debounce="100" number type="number" min="0" max="24" class="workhours" v-model="workhours[2]"></label>
      <label>週四 <input debounce="100" number type="number" min="0" max="24" class="workhours" v-model="workhours[3]"></label>
      <label>週五 <input debounce="100" number type="number" min="0" max="24" class="workhours" v-model="workhours[4]"></label>
      <label>週六 <input debounce="100" number type="number" min="0" max="24" class="workhours" v-model="workhours[5]"></label>
      <label>週日 <input debounce="100" number type="number" min="0" max="24" class="workhours" v-model="workhours[6]"></label>
    </div>
    <div class="row">
      <div class="col-md-4">
        <current-solution
          :solution="currentSolution"
          :regular-pay="regularPay"
          :overtime-pay-winner="mostOvertimePay.current"
          :expand="expandDetail">
        </current-solution>
      </div>
      <div class="col-md-4">
        <one-rest-one-off-solution
          :solution="oneRestOneOffSolution"
          :regular-pay="regularPay"
          :overtime-pay-winner="mostOvertimePay.oneRestOneOff"
          :expand="expandDetail">
        </one-rest-one-off-solution>
      </div>
      <div class="col-md-4">
        <two-off-solution
          :solution="twoOffSolution"
          :regular-pay="regularPay"
          :overtime-pay-winner="mostOvertimePay.twoOff"
          :expand="expandDetail">
        </two-off-solution>
      </div>
    </div>
  </div>
</template>

<script>
import * as solutions from '../lib/solutions';
import OffdayCondition from './OffdayCondition';
import CurrentSolution from './CurrentSolution';
import OneRestOneOffSolution from './OneRestOneOffSolution';
import TwoOffSolution from './TwoOffSolution';

export default {
  components: {
    OffdayCondition,
    CurrentSolution,
    OneRestOneOffSolution,
    TwoOffSolution
  },
  methods: {
    toggleExpanding (evt) {
      this.expandDetail = !this.expandDetail;
    },

    hash () {
      let params = {
        reason: this.reason,
        workhours: this.workhours.join(','),
        monthlyPay: this.monthlyPay
      };

      this.$router.go({
        name: 'workweek',
        query: params
      });
    },

    reasonChanged (reason) {
      this.reason = reason;
    }
  },

  data () {
    const params = this.$route.query;
    let workhours = params.workhours
                    ? params.workhours.split(',').map(h => parseInt(h))
                    : [8, 8, 8, 8, 8, 4, 0];
    let monthlyPay = 36000;
    let reason = 'disaster';
    if (params.regularDayOffWorkReason) {
      params.reason = params.regularDayOffWorkReason;
    }

    if (params.reason &&
       ((params.reason === 'laborAgree' ||
         params.reason === 'laborDisagree' ||
         params.reason === 'disaster'))) {
      reason = params.reason;
    }

    if (params.monthlyPay) {
      monthlyPay = parseFloat(params.monthlyPay);
    }

    return {
      reason: reason,
      daynames: solutions.DAY_NAMES,
      workhours: workhours,
      assumingWorkHours: 240,
      monthlyPay: monthlyPay,
      expandDetail: false,
      regularHoursPerDay: solutions.REGULAR_HOURS_PER_DAY
    };
  },
  computed: {
    hourlyPay () {
      return parseFloat(this.monthlyPay / this.assumingWorkHours);
    },
    regularPay () {
      return this.hourlyPay * 8 * 7;
    },
    currentSolution () {
      return solutions.current(this.workhours, this.hourlyPay, this.reason);
    },
    oneRestOneOffSolution () {
      return solutions.oneRestOneOff(this.workhours, this.hourlyPay, this.reason);
    },
    twoOffSolution () {
      return solutions.twoOff(this.workhours, this.hourlyPay, this.reason);
    },
    mostOvertimePay () {
      let overtimePays = [ this.currentSolution.overtimePay,
                           this.oneRestOneOffSolution.overtimePay,
                           this.twoOffSolution.overtimePay ];
      let max = Math.max(...overtimePays);
      return {
        current: this.currentSolution.overtimePay === max,
        oneRestOneOff: this.oneRestOneOffSolution.overtimePay === max,
        twoOff: this.twoOffSolution.overtimePay === max
      };
    }
  },
  watch: {
    reason (val) {
      this.hash();
    },
    workhours (val) {
      this.hash();
    },
    monthlyPay (val) {
      this.hash();
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
.warning {
  color: red;
  font-weight: bold;
}

.warning a {
  color: red;
  text-decoration: underline;
}

.info {
  color: green;
  font-weight: bold;
}

.alert a {
  text-decoration: underline;
}

.monthly-pay {
  width: 80px;
  text-align: center;
}

.assuming-work-hours {
  width: 50px;
  text-align: center;
}

.workhours {
  width: 50px;
  text-align: center;
  margin-right: 20px;
}

.pro {
  color: green;
  font-weight: bold;
}

.pro:after {
  content: "勝";
  margin-left: 5px;
  color: white;
  background-color: red;
  border: 1px solid red;
  border-radius: 50%;
  padding: 2px;
  transform: rotate(10deg);
  font-weight: normal;
}

.handler a {
  cursor: pointer;
}

</style>
