<template>
  <div :class="$style.main">
    <div :class="$style.container">
      <SipCalculator @update="onValuesUpdated($event)" :class="$style['sip-calculator']" />
      <div :class="$style.chartContainer">
        <Pie :data="data" :options="chartOptions" />
      </div>
      <div :class="$style.LineChartContainer">
        <Bar :data="lineData" :options="barChartOptions" />
      </div>
    </div>
    <div style="overflow: auto">
      <h2 style="text-align: center">Investment Summary</h2>
      <table>
        <thead>
          <tr>
            <th>{{ investmentTitle }}</th>
            <th>Year</th>
            <th>Return Rate (%)</th>
            <th>Principal</th>
            <th>Estimated Returns</th>
            <th>Total Wealth Generated</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(entry, index) in investedAmount" :key="index">
            <td>{{ formatPrice(investmentValue[index]) }}</td>
            <td>{{ index + 1 }}</td>
            <td>{{ retrunRate }}</td>
            <td>{{ formatPrice(entry) }}</td>
            <td>{{ formatPrice(expectedReturnsValues[index]) }}</td>
            <td>{{ formatPrice(returns[index]) }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup lang="ts">
import SipCalculator from './components/sip-calculator.vue';
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  ArcElement,
  BarElement
} from 'chart.js';
import { ref, type Ref } from 'vue';
import { Pie, Bar } from 'vue-chartjs';
import {
  calculateLump,
  calculateSIP,
  calculateStepUpSIP,
  calculateYearlySIP,
  formatCurrencyValue,
  formatPrice
} from './components/sip-calculator';
import { InvestmentTypes } from './constants';

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  ArcElement,
  BarElement
);

const chartOptions: any = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    tooltip: {
      enabled: false
    },
    legend: {
      display: true,
      position: 'bottom',
      labels: {
        usePointStyle: true,
        pointStyle: 'rectRounded'
      }
    }
  }
};

const barChartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      display: false
    },
    tooltip: {
      callbacks: {
        label: function () {
          return '';
        },
        afterLabel: function (context: any) {
          return [
            `Principal: ${formatCurrencyValue(investedAmount.value[context.dataIndex])}`,
            `Total Wealth: ${formatCurrencyValue(returns.value[context.dataIndex])}`
          ];
        },
        title: function () {
          return '';
        }
      }
    }
  },
  scales: {
    x: {
      display: true,
      title: {
        display: true,
        text: 'Time Period (Years)'
      },
      stacked: true,
      barPercentage: 1.0,
      categoryPercentage: 1.0
    },
    y: {
      display: true,
      title: {
        display: true,
        text: 'Future Value'
      },
      ticks: {
        callback: function (value: any) {
          return formatCurrencyValue(value);
        }
      }
    }
  }
};

const data = ref({
  labels: ['Invested', 'Est. returns'],
  datasets: [
    {
      backgroundColor: ['#dcdcdc ', '#36A2EB'],
      data: [50, 50]
    }
  ]
});

const lineData = ref({
  labels: [],
  datasets: [
    {
      label: 'Est. returns',
      data: []
    }
  ]
});

const investedAmount = ref([]);
const returns = ref([]);
const retrunRate = ref(0);
const investmentValue: Ref<number[]> = ref([] as number[]);
const expectedReturnsValues = ref([] as number[]);
const investmentTitle = ref('Monthly Investment');

const calculateChartData = (
  totalInvestment: number,
  estimatedReturns: number,
  totalReturn: number
) => {
  const investmentAmount = (totalInvestment / totalReturn) * 100;
  const returns = (estimatedReturns / totalReturn) * 100;

  return {
    labels: ['Invested', 'Est. returns'],
    datasets: [
      {
        backgroundColor: ['#f4c430 ', '#36A2EB'],
        data: [investmentAmount, returns]
      }
    ]
  };
};

const calculateLineData = (
  investment: number,
  expectedReturn: number,
  investmentType: string,
  years: number,
  stepup = 0
) => {
  const values: any = [];
  const labels: any = [];
  const investmentValues: any = [];
  expectedReturnsValues.value = [];
  investmentValue.value = [];
  investmentTitle.value =
    investmentType === InvestmentTypes.LUMPSUM ? 'Total Investment' : 'Monthly Investment';

  for (let i = 1; i <= years; i += 1) {
    const calculateReturns: any = () => {
      if (investmentType === InvestmentTypes.SIP) {
        return calculateSIP(i, investment, expectedReturn);
      } else if (investmentType === InvestmentTypes.LUMPSUM) {
        return calculateLump(i, investment, expectedReturn);
      } else if (
        investmentType === InvestmentTypes.STEPUP ||
        investmentType === InvestmentTypes.SWP
      ) {
        return calculateStepUpSIP(i, investment, stepup, expectedReturn);
      } else if (investmentType === InvestmentTypes.YEARLY) {
        return calculateYearlySIP(investment, expectedReturn, i);
      }
    };

    const { totalInvestment, totalReturn, estimatedReturns } = calculateReturns();
    investmentType === InvestmentTypes.SIP
      ? calculateSIP(i, investment, expectedReturn)
      : investmentType === InvestmentTypes.STEPUP || investmentType === InvestmentTypes.SWP
        ? calculateStepUpSIP(i, investment, stepup, expectedReturn)
        : investmentType === InvestmentTypes.LUMPSUM
          ? calculateLump(i, investment, expectedReturn)
          : calculateYearlySIP(investment, expectedReturn, i);

    labels.push(i);
    values.push(totalReturn);
    expectedReturnsValues.value.push(estimatedReturns);
    investmentValues.push(totalInvestment);
    if (investmentType === InvestmentTypes.STEPUP || investmentType === InvestmentTypes.SWP) {
      const yearlyInvestment = investment * Math.pow(1 + stepup / 100, i - 1);
      investmentValue.value.push(yearlyInvestment);
    } else {
      investmentValue.value.push(investment);
    }
  }

  investedAmount.value = investmentValues;
  returns.value = values;
  retrunRate.value = expectedReturn;

  return {
    labels: labels,
    datasets: [
      {
        backgroundColor: '#f4c430 ',
        label: 'Invested Amount',
        data: investmentValues
      },
      {
        backgroundColor: '#36A2EB',
        label: 'Est. returns',
        data: values
      }
    ]
  };
};

const onValuesUpdated = (updatedData: any) => {
  const {
    totalInvestment,
    estimatedReturns,
    totalReturn,
    expectedReturn,
    investment,
    investmentType,
    years,
    stepup
  } = updatedData;

  const fieldsToCheck = [
    totalInvestment,
    estimatedReturns,
    totalReturn,
    expectedReturn,
    investment,
    years
  ];

  for (let i = 0; i < fieldsToCheck.length; i++) {
    const value = fieldsToCheck[i];
    if (isNaN(value) || value === '' || value === 0) {
      data.value = {
        labels: ['Invested', 'Ést. returns'],
        datasets: [
          {
            backgroundColor: ['#dcdcdc ', '#36A2EB'],
            data: [100, 0]
          }
        ]
      };

      lineData.value = {
        labels: [],
        datasets: [
          {
            label: 'Est. returns',
            data: []
          }
        ]
      };

      return;
    }
  }

  data.value = calculateChartData(totalInvestment, estimatedReturns, totalReturn);
  lineData.value = calculateLineData(investment, expectedReturn, investmentType, years, stepup);
};
</script>

<style module>
.main {
  display: flex;
  justify-content: center;
  flex-direction: column;
  align-items: center;
  background-color: rgba(246, 238, 246, 0.234);
}

.container {
  display: flex;
  flex-wrap: wrap;
  gap: 2rem;
  padding: 1rem;
  box-sizing: border-box;
  justify-content: center;
  width: 80%;
}

.chartContainer {
  max-width: 300px;
  height: 280px;
  min-width: 0;
  padding-top: 15%;
}

.LineChartContainer {
  min-width: 0;
  height: 400px;
  position: relative;
}

@media (min-width: 600px) {
  .sip-calculator,
  .chartContainer {
    flex: 1 1 45%;
  }
  .LineChartContainer {
    flex: 1 1 90%;
  }
}

@media (max-width: 1200px) {
  .sip-calculator,
  .chartContainer,
  .LineChartContainer {
    flex: 1 1 100%;
    padding: 0;
  }
  .container {
    width: 100%;
    padding-top: 0%;
  }

  .main {
    align-items: normal;
  }
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen-Sans, Ubuntu, Cantarell,
    'Helvetica Neue', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>

<style>
table {
  width: 100%;
  border-collapse: collapse;
}
th,
td {
  border: 1px solid #ddd; /* Adding border to table cells */
  padding: 8px;
  text-align: right;
}
th {
  background-color: #f2f2f2;
  text-align: center;
}
tr:nth-child(even) {
  background-color: #f9f9f9;
}
</style>
