<script setup>
import { ref, onMounted } from "vue";
import * as d3 from "d3";
import { getCalendarChart } from "@/api/mypage.js";
import axios from "axios";
import { useauthControllerStore } from "@/stores/authController";

const userStore = useauthControllerStore();
const user_access_token = userStore.myAccessToken;

// 차트 사이즈
const width = 1100;
const cellSize = 35;
const height = (cellSize + 10) * 7;

// 잔디의 단위날짜 설정
const formatDate = d3.timeFormat("%x");
const formatDay = (i) => "SMTWTFS"[i];
const formatMonth = d3.timeFormat("%b");

// 주의 시작날짜를 일요일로 설정
const timeWeek = d3.timeSunday;

// Day i를 7까지 반복해서 세기
const countDay = (i) => i % 7;

// data를 받아서 가공
const today = new Date();

const getData = ref([]);

const data = ref(null);

// 색 정하는 함수
function coloring(d) {
  var c = d3.color("rgba(81, 60, 58, 1)");
  c = d3.hsl(c);
  if (d.grade) {
    c.opacity = 0.5 + 0.1 * d.grade;
    c + "";

    return d3.color(c);
  } else {
    return d3.color("rgba(81, 60, 58, 0.2)");
  }
}

// getYear의 value인 year가 변할 때마다 그에 맞춰 데이터 업데이트
function updateData(year, getYears) {
  // 각 연도 별 모든 날짜가 담긴 datesArray 생성
  const startDate = new Date(`${year}-1-1`);
  const endDate = new Date(`${year}-12-31`);
  const datesArray = d3.timeDays(startDate, endDate);

  // 상반기 하반기를 구분할 7월 1일 데이터
  // var july = new Date(`July 1, ${year}`);

  // data에 365일간의 날짜와 임의로 0으로 설정한 value가 포함된 object를 push
  data.value = new Array();
  datesArray.forEach((date) => {
    var obj = new Object();
    obj.createdAt = date;
    obj.grade = 0;
    data.value.push(obj);
  });

  // 현재 선택한 년도에 맞는 userdata 기록 가져오기
  const targetYearArray = getYears.value.find(
    ([year]) => year === getYear.value
  );

  const targetYearData = targetYearArray[1];

  // data에 유저의 기록이 있는 날짜에 grade 추가
  targetYearData.forEach((d) => {
    const n = d3.timeDay.count(d3.timeYear(d.createdAt), d.createdAt);
    data.value[n].grade = d.grade;
  });
}

// 왜인지 모르게 selectYear 함수를 html에서 작동시키면 proxy값이 달라져서 함수를 조금 바꿔서 다시 만듬
function reUpdateData(year, getYears) {
  // 각 연도 별 모든 날짜가 담긴 datesArray 생성
  const startDate = new Date(`${year}-1-1`);
  const endDate = new Date(`${year}-12-31`);
  const datesArray = d3.timeDays(startDate, endDate);

  // 상반기 하반기를 구분할 7월 1일 데이터
  // var july = new Date(`July 1, ${year}`);

  // data에 365일간의 날짜와 임의로 0으로 설정한 value가 포함된 object를 push
  data.value = new Array();
  datesArray.forEach((date) => {
    var obj = new Object();
    obj.createdAt = date;
    obj.grade = 0;
    data.value.push(obj);
  });

  // 현재 선택한 년도에 맞는 userdata 기록 가져오기
  const targetYearArray = getYears.find(([year]) => year === getYear.value);

  const targetYearData = targetYearArray[1];

  // data에 유저의 기록이 있는 날짜에 grade 추가
  targetYearData.forEach((d) => {
    const n = d3.timeDay.count(d3.timeYear(d.createdAt), d.createdAt);
    data.value[n].grade = d.grade;
  });
}

// 차트를 보여줄 svg 파일 생성
function makesvg() {
  var svg = d3
    .select(svgRef.value)
    .attr("width", width)
    .attr("height", height * 2)
    .attr("viewBox", [0, 0, width, height * 2 - 100])
    .attr("style", "max-width: 100%; height: auto; font: 25px sans-serif;")
    .attr("preserveAspectRatio", "xMaxYMax");
  return svg;
}

// svg 파일 초기화
function resetGrass() {
  var svg = d3.select(svgRef.value);

  return svg;
}

function drawGrass(data, svg, year) {
  var july = new Date(`July 1, ${year}`);

  // 해당 연도의 상반기, 하반기 데이터 가져오고 나누기
  const firstHalf = d3.filter(data.value, (d) => d.createdAt.getMonth() < 6);
  const secondHalf = d3.filter(data.value, (d) => d.createdAt.getMonth() >= 6);
  var years = [
    ["상반기", firstHalf],
    ["하반기", secondHalf],
  ];

  const yearGrass = svg
    .selectAll("g")
    .data(years)
    .join("g")
    .attr("transform", (d, i) => `translate(100.5 ,${height * i - 32})`);

  yearGrass
    .append("text")
    .attr("x", -5)
    .attr("y", -5)
    .attr("font-weight", "bold")
    .attr("text-anchor", "end")
    .text(([key]) => key);

  yearGrass
    .append("g")
    .attr("text-anchor", "end")
    .selectAll()
    .data(d3.range(7))
    .join("text")
    .attr("x", -5)
    .attr("y", (i) => (countDay(i) + 0.5) * cellSize)
    .attr("dy", "0.31em")
    .text(formatDay);

  yearGrass
    .append("g")
    .selectAll()
    .data(([, values]) => values)
    .join("rect")
    .attr("width", cellSize - 1)
    .attr("height", cellSize - 1)
    .attr("stroke", "rgba(250,250,250, 0.4)")
    .attr("x", function (d) {
      if (d.createdAt.getMonth() < 6) {
        return (
          timeWeek.count(d3.timeYear(d.createdAt), d.createdAt) * cellSize + 0.5
        );
      } else {
        return timeWeek.count(july, d.createdAt) * cellSize + 0.5;
      }
    })
    .attr("y", (d) => countDay(d.createdAt.getDay()) * cellSize + 0.5)
    .attr("fill", function (d) {
      if (d) {
        return coloring(d);
      } else {
        return d3.color("rgba(0, 0, 150, 1)");
      }
    })
    .attr("stroke", "rgba(0, 0, 0, 0.3)")
    .append("title")
    .text(function (d) {
      return `${formatDate(d.createdAt)} 학습량: ${d.grade}`;
    });

  const month = yearGrass
    .append("g")
    .selectAll()
    .data(([, values]) =>
      d3.timeMonths(d3.timeMonth(values[0].createdAt), values.at(-1).createdAt)
    )

    .join("g");

  month
    .filter((d, i) => i)
    .append("path")
    .attr("fill", "none")
    .attr("stroke", "rgba(0, 0, 0, 0.4)")
    .attr("stroke-width", 3)
    .attr("stroke-padding", "auto")
    .attr("d", pathMonth);

  month
    .append("text")
    .attr("x", function (d) {
      if (d.getMonth() < 6) {
        return timeWeek.count(d3.timeYear(d), timeWeek.ceil(d)) * cellSize + 2;
      } else {
        return timeWeek.count(july, timeWeek.ceil(d)) * cellSize + 2;
      }
    })
    .attr("y", -5)
    .text(formatMonth);
}

function drawGrassAgain(data, svg, july) {
  // 해당 연도의 상반기, 하반기 데이터 가져오고 나누기
  const firstHalf = d3.filter(data.value, (d) => d.createdAt.getMonth() < 6);
  const secondHalf = d3.filter(data.value, (d) => d.createdAt.getMonth() >= 6);
  var years = [
    ["상반기", firstHalf],
    ["하반기", secondHalf],
  ];

  svg.selectAll("*").remove();

  const yearGrass = svg
    .selectAll("g")
    .data(years)
    .join("g")
    .attr("transform", (d, i) => `translate(100.5 ,${height * i - 32})`);

  yearGrass
    .append("text")
    .attr("x", -5)
    .attr("y", -5)
    .attr("font-weight", "bold")
    .attr("text-anchor", "end")
    .text(([key]) => key);

  yearGrass
    .append("g")
    .attr("text-anchor", "end")
    .selectAll()
    .data(d3.range(7))
    .join("text")
    .attr("x", -5)
    .attr("y", (i) => (countDay(i) + 0.5) * cellSize)
    .attr("dy", "0.31em")
    .text(formatDay);

  yearGrass
    .append("g")
    .selectAll()
    .data(([, values]) => values)
    .join("rect")
    .attr("width", cellSize - 1)
    .attr("height", cellSize - 1)
    .attr("stroke", "rgba(250,250,250, 0.4)")
    .attr("x", function (d) {
      if (d.createdAt.getMonth() < 6) {
        return (
          timeWeek.count(d3.timeYear(d.createdAt), d.createdAt) * cellSize + 0.5
        );
      } else {
        return timeWeek.count(july, d.createdAt) * cellSize + 0.5;
      }
    })
    .attr("y", (d) => countDay(d.createdAt.getDay()) * cellSize + 0.5)
    .attr("fill", function (d) {
      if (d) {
        return coloring(d);
      } else {
        return d3.color("rgba(0, 0, 150, 1)");
      }
    })
    .attr("stroke", "rgba(0, 0, 0, 0.3)")
    .append("title")
    .text(function (d) {
      return `${formatDate(d.createdAt)} 학습량: ${d.grade}`;
    });

  const month = yearGrass
    .append("g")
    .selectAll()
    .data(([, values]) =>
      d3.timeMonths(d3.timeMonth(values[0].createdAt), values.at(-1).createdAt)
    )

    .join("g");

  month
    .filter((d, i) => i)
    .append("path")
    .attr("fill", "none")
    .attr("stroke", "rgba(0, 0, 0, 0.4)")
    .attr("stroke-width", 3)
    .attr("stroke-padding", "auto")
    .attr("d", pathMonth);

  month
    .append("text")
    .attr("x", function (d) {
      if (d.getMonth() < 6) {
        return timeWeek.count(d3.timeYear(d), timeWeek.ceil(d)) * cellSize + 2;
      } else {
        return timeWeek.count(july, timeWeek.ceil(d)) * cellSize + 2;
      }
    })
    .attr("y", -5)
    .text(formatMonth);
}

// 각 달의 왼쪽에 얇은 흰 선 만드는 함수
function pathMonth(t) {
  var july = new Date(`${t.getFullYear()}-7-1`);

  const d = countDay(t.getDay());
  const w = timeWeek.count(d3.timeYear(t), t);
  const h = timeWeek.count(d3.timeYear(t), july);

  if (w >= h) {
    return `${
      d === 0
        ? `M${(w - h) * cellSize},0`
        : `M${(w - h + 1) * cellSize},0V${d * cellSize}H${(w - h) * cellSize}`
    }V${7 * cellSize}`;
  } else {
    return `${
      d === 0
        ? `M${w * cellSize},0`
        : `M${(w + 1) * cellSize},0V${d * cellSize}H${w * cellSize}`
    }V${7 * cellSize}`;
  }
}

var svgRef = ref(null); // ref로 svg 엘리먼트 선언

// 초기 설정을 2024로 반응형 변수 생성
const getYear = ref(today.getFullYear());

const getYears = ref();

const userYears = ref();

onMounted(async () => {
  try {
    await axios
      .get("https://i10a609.p.ssafy.io/api/mypage/history", {
        headers: {
          Authorization: `Bearer ${user_access_token}`,
          "Content-Type": "application/json",
        },
      })
      .then(function (response) {
        if (response.data.data == 0) {
          getData.value = [{ createdAt: today, grade: 0 }];
        } else {
          getData.value = response.data.data;
        }
      })
      .catch((error) => console.log(error));

    // 각 data의 date를 Date 형식으로 변환
    getData.value.forEach((d) => {
      d.createdAt = new Date(d.createdAt);
    });

    // 받은 data를 연도별 최신순으로 구분
    getYears.value = d3
      .groups(getData.value, (d) => d.createdAt.getFullYear())
      .reverse();

    // 데이터가 있는 연도들만 담은 userYears 배열 생성
    userYears.value = getYears.value.map(([year]) => year);

    var year = getYear.value;
    updateData(year, getYears);

    var svg = makesvg();
    drawGrass(data, svg, year);
  } catch (error) {
    console.log(error);
  }
});

const dropdownOpen = ref(false);

const toggleDropdown = () => {
  dropdownOpen.value = !dropdownOpen.value;
};

const selectYear = (year, getYears) => {
  dropdownOpen.value = false;
  getYear.value = year;
  reUpdateData(year, getYears);
  var svg = resetGrass();
  var july = new Date(`${year}-7-1`);

  drawGrassAgain(data, svg, july);
};
</script>

<template>
  <div class="grass">
    <h2 class="text-center">일일 학습 경과</h2>
    <div class="grass-component">
      <div class="dropdown">
        <button @click="toggleDropdown" class="toggle-button">
          {{ getYear }}
        </button>

        <div v-if="dropdownOpen" class="dropdown-content">
          <div
            v-for="year in userYears"
            :key="year"
            @click="selectYear(year, getYears)"
          >
            {{ year }}
          </div>
        </div>
      </div>
      <div>
        <svg ref="svgRef"></svg>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.grass {
  width: 80vh;
}
.dropdown {
  margin-left: 12px;
}

.toggle-button {
  background-color: brown;
  color: white;
  padding-top: 4px;
  padding-inline: 4px;
  text-align: center;
  border: none;
  cursor: pointer;
}

.dropdown-content {
  position: absolute;
  background-color: #f9f9f9;
  min-width: 160px;
  box-shadow: 0px 8px 16px 0px rgba(0, 0, 0, 0.2);
  z-index: 1;
}

.dropdown-content div {
  padding: 12px 16px;
  margin: 0;
  text-decoration: none;
  color: #333;
  display: block;
  cursor: pointer;
}

.dropdown-content.show {
  display: block;
}
</style>
