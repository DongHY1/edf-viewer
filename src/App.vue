<script setup lang="ts">
import { ref } from 'vue';
import Plotly, { Data } from 'plotly.js-dist-min';
import Papa from 'papaparse';
// Reactive state for container and loading status
const container = ref<HTMLElement | null>(null);
const parserInstance = ref<Papa.Parser|null>(null)
const chunkSize = 10000;
let initialRender = true; // 初始渲染标志

interface UpdateProps{
  x:number[][],
  y:number[][]
}
// Function to handle file input change
const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files[0]) {
    const file = target.files[0];
    parseCSVFile(file);
  }
};

// Function to parse the CSV file
const parseCSVFile = (file: File) => {
  let x: number[] = [];
  let y: { [key: string]: number[] } = {}; // Store y-values for each column

  // Initialize y arrays for each key
  const initYArrays = (fields: string[]) => {
    fields.forEach(field => {
      if (field !== 'index' && field !== 'hour' && field !== 'minute' && field !== 'second' && field !== 'hertzs') {
        y[field] = [];
      }
    });
  };
  Papa.parse(file, {
    header: true,
    chunkSize: chunkSize * 1024, // 设置每次读取的块大小
    chunk: (results, parser) => {
      parserInstance.value = parser 
      if (initialRender && results.meta.fields) {
        initYArrays(results.meta.fields);
      }
      parser.pause();
      const chunkData = results.data;
      chunkData.forEach((row: any) => {
        x.push(row['index']);
        for (const key in y) {
          y[key].push(row[key]);
        }
      });

      if (container.value) {
        if (initialRender) {
          // 初始渲染
          let data:Data[] = [];
          for (const key in y) {
            data.push({
              x: x,
              y: y[key],
              name: key,
              type: 'scattergl',
              mode: 'lines+markers'
            });
          }
          Plotly.newPlot(container.value, data, {
            width: 1200,
            height: 800,
            title: 'EDF Viewer'
          },{
            staticPlot:true
          });
          initialRender = false;
        } else {
          // 追加渲染
          let update:UpdateProps = {
            x: [],
            y: []
          };
          let indices = [];
          let i = 0;
          for (const key in y) {
            update.x.push(x);
            update.y.push(y[key]);
            indices.push(i);
            i++;
          }
          Plotly.extendTraces(container.value, update, indices);
        }
        x = [];
        for (const key in y) {
          y[key] = [];
        }
      }

       // 延时一下再继续解析
      setTimeout(() => parser.resume(), 0);
    },
    complete: (results) => {
      console.log('All chunks processed', results);
    },
    error: (error) => {
      console.error('Error parsing CSV:', error);;
    }
  });
};
</script>

<template>
  <div>
    <input type="file" @change="handleFileChange" accept=".csv" />
    <div ref="container"></div>
  </div>
</template>