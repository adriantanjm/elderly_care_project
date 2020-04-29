<script>
  import { scaleOrdinal, scaleBand, scaleThreshold, scaleLinear } from "d3-scale";
  import { schemeAccent, schemeOranges } from "d3-scale-chromatic";
  import { plnAreas } from "./planning_areas.js";
  import { data } from "./popdata.js";
  import { elder_facilities } from "./elder_facilities.js";
  import { format, formatPrefix } from "d3-format";
  import Box from "./Box.svelte";
  import { hexGrid } from "./hex_grid.js";
  import {
    Graphic,
    Section,
    PointLayer,
    Rectangle,
    RectangleLayer,
    PolygonLayer,
    DiscreteLegend,
    XAxis,
    YAxis,
    createGeoScales,
    Label,
    Title
  } from "@snlab/florence";
  import DataContainer from "@snlab/florence-datacontainer";

  const padding = { left: 90, bottom: 30, top: 0, right: 10 };

  // map data
  const planningAreas = new DataContainer(plnAreas);
  const myGeoScale = createGeoScales(planningAreas.domain("$geometry"));
  const towns = planningAreas.column("PLN_AREA_N");

  // pop data
  const popdata = new DataContainer(data);

  function filterElderly(row) {
    return [
      "60_to_64",
      "65_to_69",
      "70_to_74",
      "75_to_79",
      "80_to_84",
      "85_to_89",
      "90_and_over"
    ].includes(row.age_group);
  }

  // population per town
  const popTowns = popdata
    .groupBy("planning_area")
    .summarise({ residents_per_town: { Resident: "sum" } });
  // elderly per town
  const elderly = popdata
    .filter(filterElderly)
    .groupBy("planning_area")
    .summarise({ elderly_per_town: { Resident: "sum" } });
  const percentElderly = elderly.map("elderly_per_town", (d, i) => {
    return popTowns.column("residents_per_town")[i] === 0
      ? 0
      : d / popTowns.column("residents_per_town")[i];
  });
  elderly.addColumn("percent_elderly", percentElderly);

  // binning
  const numClass = 5;
  const binnedElderly = elderly.bin({
    groupBy: "percent_elderly",
    method: "EqualInterval",
    numClasses: numClass
  });
  const classBins = binnedElderly.column("bins");
  function binsToThreshold(bins) {
    const thresholds = [];
    for (let index = 1; index < bins.length; index++) {
      const bin = bins[index];
      thresholds.push(bin[0]);
    }
    return thresholds;
  }
  const classThresholds = binsToThreshold(classBins);
  const myColorscale = scaleThreshold()
    .domain(classThresholds)
    .range(schemeOranges[numClass]);

  // augment thresholds from
  // [0.06530183727034121, 0.13060367454068242, 0.1959055118110236, 0.26120734908136484] to
  // [0, 0.06530183727034121, 0.13060367454068242, 0.1959055118110236, 0.26120734908136484, 0.32650918635170606]
  const boundedThresholds = [
    binnedElderly.domain("bins")[0],
    ...classThresholds,
    binnedElderly.domain("bins")[1]
  ];
  // round to 2dp
  const boundedThresholds2dp = boundedThresholds.map(format(".2f"));

  // KIM's Code- Adding facilities data and adding boxes of different facilities and their quantity per town
  // facilities data-from Kim
  const el_Facilities = new DataContainer(elder_facilities);
  // filter by hovered area, grouped by number of facilities per town- from Kim
  let filteredTown, facilitiesPerTown;
  $: {
    if (hoveredAreaIndex === undefined) {
      filteredTown = el_Facilities;
    } else {
      filteredTown = el_Facilities.filter(
        row => row["PLANNING AREAS"] === towns[hoveredAreaIndex]
      ); // i changed the key here
    }
    facilitiesPerTown = filteredTown
      .groupBy("TYPE OF FACILITIES")
      .summarise({ total_facilities: { TOTAL: "sum" } });
  }
  // set up hover handlers- from Kim
  let mousePosition = undefined;
  let selectedAreainfo;
  $: selectedAreainfo = planningAreas.column("PLN_AREA_N")[hoveredAreaIndex];
  let hoveredAreaIndex = undefined;

  function handleMouseover(event) {
    hoveredAreaIndex = event.key;
    mousePosition = {
      x: event.localCoordinates.x,
      y: event.localCoordinates.y
    };
  }
  function handleMouseout() {
    hoveredAreaIndex = undefined;
    mousePosition = undefined;
  }

  // Sunny's Code- Containing formatPrefix
  // Population Pyramid- from Sunny
  const tickFormatter = formatPrefix(".0", 1e3); // unit scale on x axis, 1x10^3- unit:thousands

  let townFilter, grouped, maleValues, femaleValues, ages;
  $: {
    if (clickedAreaIndex === "") {
      townFilter = popdata;
    } else {
      townFilter = popdata.filter(
        row => row.planning_area === towns[clickedAreaIndex]
      ); // filter out the row which planning_area has the same string as towns[] , the clickedAreaIndex would from the event key
    }

    grouped = townFilter.groupBy("sex");
    // grouped datacontainer has a lot of subcontainers

    [maleValues, femaleValues] = grouped.map("$grouped", group =>
      group.column("Resident")
    ); // group is it’s just an argument to an anonymous function, dont need be group can be anything
    ages = grouped.map("$grouped", group => group.column("age_group"))[0];
  }

  // domainsF
  const valueDomain = popdata.domain("Resident");
  const ageDomain = popdata.domain("age_group");

  // scales
  const scaleXresident = scaleLinear().domain([0, valueDomain[1]]); // 2nd number from valueDomain, the largest number from Resident column
  const scaleYage_group = scaleBand()
    .domain(ageDomain)
    .padding(0.2); // change the bar size

  let clickedAreaIndex = "";
  function handleClick(event) {
    clickedAreaIndex = event.key;
  }
</script>

<head>
  <title>Visualising the Distribution of Elderly Population Across Singapore</title>
</head>

  <div class="title">
    <!-- title/header content goes here -->
    <h1>
     Visualising the Distribution of Elderly Population across Singapore
    </h1>
    <h2>
      With data on elderly care facilities distribution across the Country
    </h2>
    <h3>Dashboard made by Adrian, Kimberly, Sunny</h3>
    <h3>With the help of Jo Hsi</h3>
  </div>



<div class="graph">
  <div class="main-chart">
  <Graphic width="100%" height="100%" viewBox="0 0 1250 600">
    <!-- Sunny's Pyramid Chart -->
    <Title   
        x={200}
        y={15}
        title={clickedAreaIndex === "" ? "Singapore" : towns[clickedAreaIndex]} 
        titleFontFamily="Josefin Sans", sans-serif
      />
      <!-- male chart -->
      <Section
        x1={0}
        x2={205}
        y1={30}
        y2={550}
        scaleX= {scaleXresident}
        scaleY= {scaleYage_group}
        padding={{ top: 0, bottom: 30, left: 0, right: 0}}
        flipX
        flipY
      >
        <RectangleLayer 
          x1={0}
          x2={maleValues}
          y1={ages}
          y2={({scaleY}) => ages.map(age_group => scaleY(age_group) + scaleY.bandwidth())}
          fill={'steelblue'}
        />
        <Title title="Male" 
        titleFontFamily="Josefin Sans", sans-serif/>
        <XAxis labelFormat={tickFormatter} baseLine={false} 
        labelRotate=-40 labelFontSize=8 />
      </Section>
      <!-- female chart -->
      <Section
        x1={205} 
        x2={410}
        y1={30}
        y2={550}
        scaleX= {scaleXresident}
        scaleY= {scaleYage_group}
        padding={{ top: 0, bottom: 30, left: 0, right: 0}}
        flipY
      >
        <RectangleLayer 
          x1={0}
          x2={femaleValues}
          y1={ages}
          y2={({scaleY}) => ages.map(age_group => scaleY(age_group) + scaleY.bandwidth())}
          fill={'crimson'}
        />
        <Title title="Female" 
        titleFontFamily="Josefin Sans", sans-serif/>
        <XAxis labelFormat={tickFormatter} baseLine={false} 
        labelRotate=-40 labelFontSize=8/>
        <YAxis flip={true} labelFontSize=8 labelColor=rgb(10, 96, 132);/>
      </Section> 
    <!-- </Section> -->
    
    <!-- Adrian's Main map and legend  -->
    <Section
        x1={350}
        x2={1025}
        y1={10}
        y2={610}
      {padding}
      {...myGeoScale}
      flipY>
      {#each planningAreas.rows() as row, i (row.$key)}
      <PolygonLayer 
          geometry={planningAreas.column('$geometry')}
          stroke={'white'}
          strokeWidth={1} 
          fill={elderly.column('percent_elderly').map(myColorscale)}
          onMouseover={handleMouseover}
          onMouseout={handleMouseout}
          onClick={handleClick}
        />
        {/each}
      {#if mousePosition !== undefined}
      <Rectangle
      x1={mousePosition.x + 1200}
      x2={mousePosition.x + 30000}
      y1={mousePosition.y}
      y2={mousePosition.y + 3000}
      fill={'white'}
      opacity={0.7}
      stroke={'white'}
      />
      <Label
      x={mousePosition.x + 1700}
      y={mousePosition.y + 1500}
      anchorPoint={'l'}
      fill={'rgb(10, 96, 132)'}
      fontFamily={'Josefin Sans'}
      text={`${selectedAreainfo}: ${format(".2%")(elderly.column('percent_elderly')[hoveredAreaIndex])}`}
      />
{/if}
    </Section>
    <Section
        x1={850}
        x2={1300}
        y1={20}
        y2={500}
        {padding}
      >

      <DiscreteLegend
      title={'Percentage of Elderly'}
      fill={scaleLinear().domain(boundedThresholds2dp).range(schemeOranges[numClass])}
      labels={boundedThresholds2dp}
      labelCount={numClass}
      orient={'vertical'}
      height={50}
      titleX={() => 1100}
      titleY={() => 30}
      hjust={'center'}
      flip
      />
    </Section> 

  </Graphic>

  <!-- Kim's Box svelte for Number and Type of Facilities -->
<div class= "style_box">
  <Box>
    <span slot="name">
      Home care
    </span>
    <span slot="address">
      No. of<br>
      facilities
      {facilitiesPerTown.column('total_facilities')[0]}
    </span>
  </Box>
  <Box>
    <span slot="name">
      Day care
    </span>
    <span slot="address">
      No. of<br>
      facilities
      {facilitiesPerTown.column('total_facilities')[1]}
    </span>
  </Box>
  <Box>
    <span slot="name">
      Stay in
    </span>
    <span slot="address">
      No. of<br>
      facilities
      {facilitiesPerTown.column('total_facilities')[2]}
    </span>
  </Box>
</div>
</div>
</div>





<div class="explanation">
<p class="texthead">Elderly Care and its Relevance to Urban Planning</p>
<p class="textbody1">
Like many Asian societies, Singapore is an ageing society with low birth-replacement rate. As we can see on the population pyramid in this slide, a significantly large cohort of Singaporeans will progress into their twilight years, many are bound to experience physical deterioration and cognitive decline. Therefore, it is important that we consider these impending demographic transformations that will require new forms of resources. 
However, what kinds of elderly care facilities do we need? And we are providing enough elderly care facilities to ageing towns? These are questions which bugged many urban planners in Asia, specifically in ageing societies like Japan and Singapore. 
</p>

<p class="textbody2">
Therefore, this visualization is trying to understand whether Singapore's elderly care services resource allocation has met the current progressively ageing population. The equal interval map is built based on the percentage of elderly per planning area in order to provide an overview of the elderly population distribution through the island. The population pyramid at the left is showing the detailed population breakdown for each planning area.
<span class="clicked">Please click </span> on the map to find out the details for each town. The interactive box indicator at the right is based on three primary elderly care services: day-care, home care, and stay-in facilities.
<span class="clicked">Please hover over  </span>to find out the exact number of services for each planning area.   
<span class="clicked">The translucent box</span> indicates the percentage of elderly living in the planning area.    
</p>


</div>

<style>
  .clicked {
    font-family: "Josefin Sans", sans-serif;
    font-weight: 700;
    text-align: left;
    font-size: 16px;
    text-align: justify;
    line-height: 1.5;
    color: rgb(10, 96, 132);
  }

  h1,
  h2,
  h3 {
    font-family: "Josefin Sans", sans-serif;
    font-weight: 200;
    text-align: center;
    color: rgb(10, 96, 132);
  }

  h1 {
    font-size: 30px;
    font-weight: bold;
  }

  h2 {
    font-size: 20px;
  }

  h3 {
    font-size: 14px;
  }

  p {
    font-family: "Josefin Sans", sans-serif;
    font-weight: 200;
    text-align: center;
    font-size: 12px;
    color: rgb(5, 10, 52);
  }

  .texthead {
    font-family: "Josefin Sans", sans-serif;
    font-weight: bold;
    text-align: left;
    font-size: 20px;
    color: rgb(10, 96, 132);
  }

  .textbody1 {
    font-family: "Josefin Sans", sans-serif;
    font-weight: 100;
    text-align: left;
    font-size: 16px;
    text-align: justify;
    line-height: 1.5;
    color: rgb(0, 0, 0);

    top: 110%;
    left: 3%;
    bottom: 10%;

    width: 1350px;
    height: 100px;
  }
  .textbody2 {
    font-family: "Josefin Sans", sans-serif;
    font-weight: 100;
    text-align: left;
    font-size: 16px;
    text-align: justify;
    line-height: 1.5;
    color: rgb(0, 0, 0);

    top: 110%;
    left: 3%;
    bottom: 10%;

    width: 1350px;
    height: 150px;
  }

  div.explanation {
    position: absolute;
    top: 110%;
    left: 3%;
    bottom: 10%;

    width: 1350px;
    height: 150px;
  }

  /* set up box-style- from Kim */
  .style_box {
    display: flex;
    flex-direction: row;
    column-gap: 20px;
    position: absolute;
    top: 85%;
    left: 75%;
    width: 10%;
    height: 23%;
  }

  /* .hover-instruction {
                                                                                                                                    position: absolute;
                                                                                                                                    top: 35px;
                                                                                                                                    right: 265px;
                                                                                                                                    width: 100px;
                                                                                                                                    height: 100px;
                                                                                                                                  } */

  /* div.absolute3 {
                                                                                                                                    position: absolute;
                                                                                                                                    top: 700px;
                                                                                                                                    left: 50px;
                                                                                                                                    width: 500px;
                                                                                                                                    height: 100px;
                                                                                                                                  }

                                                                                                                                  div.absolute5 {
                                                                                                                                    position: absolute;
                                                                                                                                    top: 750px;
                                                                                                                                    left: 50px;
                                                                                                                                    width: 500px;
                                                                                                                                    height: 500px;
                                                                                                                                  } */
</style>