<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import * as topojson from 'topojson-client';

  let svgElement;
  let incomeData = new Map();

  onMount(async () => {
    const us = await d3.json("/states-albers-10m.json");
    const csvData = await d3.csv("/census_cleaned.csv");
    csvData.filter(d => d.Category === "Households").forEach(d => {
      incomeData.set(d.State, d.median_income);
    });
    initializeChart(us);
  });

  function initializeChart(us) {
    const width = 975;
    const height = 610;
    const colorScale = d3.scaleLinear()
      .domain([52000, 102000]) //d3.extent(Array.from(incomeData.values()))
      .range(["lightgreen", "darkgreen"]); // Updated color scale from light green to dark green
    console.log(incomeData.values())
    console.log(d3.min(incomeData.values()))

    const svg = d3.select(svgElement)
        .attr("viewBox", `0 0 ${width} ${height}`)
        .attr("preserveAspectRatio", "xMidYMid meet")
        .attr("width", width)
        .attr("height", height)
        .style("display", "block")
        .style("margin", "auto")
        .style("overflow", "visible");

    const path = d3.geoPath();
    const g = svg.append("g");

    g.selectAll("path")
      .data(topojson.feature(us, us.objects.states).features)
      .join("path")
      .attr("fill", d => {
        const income = incomeData.get(d.properties.name);
        return income ? colorScale(income) : "#cccccc";
      })
      .attr("d", path)
      .append("title")
      .text(d => `${d.properties.name}: $${incomeData.get(d.properties.name) || 'No data'}`);

    g.append("path")
      .attr("fill", "none")
      .attr("stroke", "white")
      .attr("stroke-linejoin", "round")
      .attr("d", path(topojson.mesh(us, us.objects.states, (a, b) => a !== b)));
  }
</script>

<svelte:element this="svg" bind:this={svgElement} />
