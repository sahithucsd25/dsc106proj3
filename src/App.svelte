<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import * as topojson from 'topojson-client';
  import { Legend } from '../src/helpers';

  let svgElement;
  let legend;
  let incomeData = new Map();
  let domain = [];

  const stateCodes = {
  "Alabama": "AL",
  "Alaska": "AK",
  "Arizona": "AZ",
  "Arkansas": "AR",
  "California": "CA",
  "Colorado": "CO",
  "Connecticut": "CT",
  "Delaware": "DE",
  "District of Columbia": "DC",
  "Florida": "FL",
  "Georgia": "GA",
  "Hawaii": "HI",
  "Idaho": "ID",
  "Illinois": "IL",
  "Indiana": "IN",
  "Iowa": "IA",
  "Kansas": "KS",
  "Kentucky": "KY",
  "Louisiana": "LA",
  "Maine": "ME",
  "Maryland": "MD",
  "Massachusetts": "MA",
  "Michigan": "MI",
  "Minnesota": "MN",
  "Mississippi": "MS",
  "Missouri": "MO",
  "Montana": "MT",
  "Nebraska": "NE",
  "Nevada": "NV",
  "New Hampshire": "NH",
  "New Jersey": "NJ",
  "New Mexico": "NM",
  "New York": "NY",
  "North Carolina": "NC",
  "North Dakota": "ND",
  "Ohio": "OH",
  "Oklahoma": "OK",
  "Oregon": "OR",
  "Pennsylvania": "PA",
  "Puerto Rico": "PR",
  "Rhode Island": "RI",
  "South Carolina": "SC",
  "South Dakota": "SD",
  "Tennessee": "TN",
  "Texas": "TX",
  "Utah": "UT",
  "Vermont": "VT",
  "Virginia": "VA",
  "Washington": "WA",
  "West Virginia": "WV",
  "Wisconsin": "WI",
  "Wyoming": "WY"
} //not being used rn, would be if we can get labels to show on zoom

  onMount(async () => {
    const us = await d3.json("/states-albers-10m.json");
    const csvData = await d3.csv("/census_cleaned.csv");
    csvData.filter(d => d.Category === "Households").forEach(d => {
      incomeData.set(d.State, d.median_income);
    });
    const asArray = [...incomeData].map(([name, value]) => (Number(value)));
    domain = d3.extent(asArray);
    initializeChart(us);
    createLegend();
  });

  function initializeChart(us) {
    const width = 1175;
    const height = 810;
    const colorScale = d3.scaleLinear()
      .domain(domain) //d3.extent(Array.from(incomeData.values()))
      .range(["lightgreen", "darkgreen"]); // Updated color scale from light green to dark green

    const zoom = d3.zoom()
    .scaleExtent([1, 8])
    .on("zoom", zoomed);

    const svg = d3.select(svgElement)
        .attr("viewBox", `0 0 ${width} ${height}`)
        .attr("preserveAspectRatio", "xMidYMid meet")
        .attr("width", width)
        .attr("height", height)
        .style("display", "block")
        .style("margin", "auto")
        .style("overflow", "hidden")
        .on('click', reset)

    const path = d3.geoPath();
    const g = svg.append("g");

    let states = g.selectAll("path")
      .data(topojson.feature(us, us.objects.states).features)
      .join("path")
      .on('click', clicked)
      .attr('d', path)
      .attr("fill", d => {
        const income = incomeData.get(d.properties.name);
        return income ? colorScale(income) : "#cccccc";
      })
      .attr("d", path)
      .append("title")
      .text(d => `${d.properties.name}: $${Number(incomeData.get(d.properties.name)).toLocaleString() || 'No data'}`).exit()

    // const labels = g.selectAll('.label')
    //   .data(topojson.feature(us, us.objects.states).features)
    //     .enter()
    //   ;

    g.append("path")
      .attr("fill", "none")
      .attr("stroke", "white")
      .attr("stroke-linejoin", "round")
      .attr("d", path(topojson.mesh(us, us.objects.states, (a, b) => a !== b)));

    states = states.enter()
      .append('text')
        .attr("class", "label")
        .attr("transform", d => "translate(" + path.centroid(d) + ")")
        .text(d => `${stateCodes[d.properties.name]}\n $${Number(incomeData.get(d.properties.name)).toLocaleString()}`)
    const labels = g.selectAll('.label')
      .data(topojson.feature(us, us.objects.states).features)
        .enter()


    
    svg.call(zoom);

    function clicked(event, d) {
      const [[x0, y0], [x1, y1]] = path.bounds(d);
      event.stopPropagation();
      console.log(g.selectAll('label'))
      // data = incomeData.get(d.properties.name);
      console.log(incomeData);
      console.log(d.properties.name)
      const data = incomeData.get(d.properties.name);
      console.log(data);
      svg.transition().duration(750).call(
        zoom.transform,
        d3.zoomIdentity
          .translate(width / 2, height / 2)
          .scale(Math.min(8, 0.9 / Math.max((x1 - x0) / width, (y1 - y0) / height)))
          .translate(-(x0 + x1) / 2, -(y0 + y1) / 2),
        d3.pointer(event, svg.node())
      );
    }

    function reset() {
      console.log('hi')
      svg.transition().duration(750).call(
      zoom.transform,
      d3.zoomIdentity,
      d3.zoomTransform(svg.node()).invert([width / 2, height / 2])
    );
    }

    function zoomed(event) {
      const {transform} = event;
      g.attr("transform", transform);
      g.attr("stroke-width", 1 / transform.k);
    }
  }

  function createLegend() {
    const legendContainer = d3.select('#legend-container');
    const legend = Legend(d3.scaleLinear(domain, ['lightgreen', 'darkgreen']), { title: 'Median Income (USD)', ticks: 5, tickFormat: (d) => '$' + d.toLocaleString() });
      legendContainer
        .append(() => legend);
  }


  
  
</script>

<main>
  <div class='main'>
    <h1>How does median income vary across the US?</h1>
    <div class='legend' id='legend-container'/>
    <svelte:element this="svg" bind:this={svgElement} />
  </div>
</main>

