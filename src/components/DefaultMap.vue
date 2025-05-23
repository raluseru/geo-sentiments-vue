<script setup lang="ts">
// @ts-nocheck
import { onMounted, onBeforeUnmount } from 'vue';
import * as am5 from '@amcharts/amcharts5';
import * as am5map from '@amcharts/amcharts5/map';
import am5geodata_worldLow from '@amcharts/amcharts5-geodata/worldLow';
import Animated from '@amcharts/amcharts5/themes/Animated';
import { csvParse } from 'd3-dsv';


interface SentimentRow {
  Country: string;
  Region: string;
  RandomValue: string;
}

let root!: am5.Root;
let chart!: am5map.MapChart;
let polygonSeries!: am5map.MapPolygonSeries;
let zoomControl!: am5map.ZoomControl;
let heatLegend!: am5.HeatLegend;
let previousPolygon: am5map.MapPolygon | null = null;

onMounted(async () => {

  root = am5.Root.new('chartdiv');
  root.setThemes([ Animated.new(root) ]);

  // Create the map chart

  chart = root.container.children.push(
    am5map.MapChart.new(root, {
      panX: 'translateX',
      panY: 'translateY',
      projection: am5map.geoMercator()
    })
  );

  // Create the polygon series

  polygonSeries = chart.series.push(
    am5map.MapPolygonSeries.new(root, {
      geoJSON: am5geodata_worldLow,
      exclude: ['AQ']
    })
  );

  polygonSeries.mapPolygons.template.setAll({
    tooltipText: '{name}: {value}',
    toggleKey: 'active',
    interactive: true
  });

  // hover/active fills
  polygonSeries.mapPolygons.template.states.create('hover', {
    fill: root.interfaceColors.get('primaryButtonHover')!
  });
  polygonSeries.mapPolygons.template.states.create('active', {
    fill: root.interfaceColors.get('primaryButtonHover')!
  });

  // click to zoom
  polygonSeries.mapPolygons.template.events.on(
    'active',
    (ev: am5.Event<am5map.MapPolygonSeries, am5map.MapPolygon>) => {
      const target = ev.target as am5map.MapPolygon;
      if (previousPolygon && previousPolygon !== target) {
        previousPolygon.set('active', false);
      }
      if (target.get('active')) {
        polygonSeries.zoomToDataItem(target.dataItem!);
      } else {
        chart.goHome();
      }
      previousPolygon = target;
    }
  );

  // Zoom controls

  zoomControl = chart.set('zoomControl', am5map.ZoomControl.new(root, {}));
  zoomControl.homeButton.set('visible', true);
  chart.chartContainer.get('background')!.events.on('click', () => {
    chart.goHome();
  });

  // Heat legend

  heatLegend = chart.children.push(
    am5.HeatLegend.new(root, {
      orientation: 'vertical',
      startColor: am5.color(0x7f1d1d),
      endColor: am5.color(0x15803d),
      startText: 'low',
      endText: 'high',
      stepCount: 5,
      x: am5.p100,
      centerX: am5.p100,
      paddingRight: 20,
      paddingTop: 20,
      paddingBottom: 20
    })
  );

  polygonSeries.set('heatRules', [
    {
      target: polygonSeries.mapPolygons.template,
      dataField: 'value',
      min: heatLegend.get('startColor')!,
      max: heatLegend.get('endColor')!,
      key: 'fill'
    }
  ]);

  heatLegend.startLabel.setAll({
    fontSize: 12,
    fill: heatLegend.get('startColor')!
  });
  heatLegend.endLabel.setAll({
    fontSize: 12,
    fill: heatLegend.get('endColor')!
  });

  // update legend when data is processed
  polygonSeries.events.on('datavalidated', () => {
    heatLegend.set('startValue', polygonSeries.getPrivate('valueLow') as number);
    heatLegend.set('endValue', polygonSeries.getPrivate('valueHigh') as number);
  });

  // hover shows exact value
  polygonSeries.mapPolygons.template.events.on('pointerover', ev => {
    const item = ev.target.dataItem!;
    heatLegend.showValue(item.get('value') as number);
  });
  polygonSeries.mapPolygons.template.events.on('pointerout', () => {
    heatLegend.showValue(undefined);
  });


  // Load & parse CSV with d3-dsv

  const response: Response = await fetch('/geo_sentiments.csv');
  const csvText: string = await response.text();
  const rows: SentimentRow[] = csvParse<SentimentRow>(csvText);

  // Transform to the { name, value } shape amCharts expects
  const chartData: { name: string; value: number }[] = rows.map(row => ({
    name: row.Region,
    value: +row.RandomValue
  }));

  polygonSeries.data.setAll(chartData);
  polygonSeries.appear(1000, 100);
});

onBeforeUnmount(() => {
  root.dispose();
});
</script>

<template>
  <div id="chartdiv" style="width:100%; height:500px;"></div>
</template>
