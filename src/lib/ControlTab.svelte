<script lang="ts">
  import _ from "lodash";
  import { getRandomColor } from "../utils";
 
  export let percent: number;
  export let playing: boolean;
  export let play: () => any;
  export let pause: () => any;
  export let startPoint: Point;
  export let lines: Line[];
  export let robotWidth: number;
  export let robotHeight: number;
  export let robotXY: BasePoint;
  export let robotHeading: number;
  export let x: d3.ScaleLinear<number, number, number>;
  export let y: d3.ScaleLinear<number, number, number>;

  let importCode: string;

  function getCode(line: Line) {
    let code = "";
    let points = "";
    line.controlPoints.forEach(controlPoint => {
      points += `new Point(${controlPoint.x}, ${controlPoint.y}), `;
    });
    points += `new Point(${line.endPoint.x}, ${line.endPoint.y}), `;

    points = points.slice(0, -2);
    code = ".addPath("+line.name+", ";
    
    if (line.endPoint.heading == "linear") {
      code += line.endPoint.heading.toUpperCase()+"("+line.endPoint.startDeg+", "+line.endPoint.endDeg+"), ";
    } else if (line.endPoint.heading == "tangential"){
      if (line.endPoint.reverse == true) {
        code += line.endPoint.heading.toUpperCase() + "("+line.endPoint.reverse+"), ";
      } else {
        code += line.endPoint.heading.toUpperCase() + "(), ";
      }
    } else if (line.endPoint.heading == "constant"){
      code += line.endPoint.heading.toUpperCase() + "("+line.endPoint.degrees+"), ";
    }
    
    code += points + ")";
    return code;
  }
  
  
  function parseImportCode() {
    // ChatGPT wrote this function :D
    // Regex pattern to match addPath calls
    const pathPattern = /\.addPath\(\s*(\w+),\s*(\w+)\(([^)]*)\)\s*(?:,\s*new\s+Point\(([\d\.\-]+),\s*([\d\.\-]+)\))?(?:,\s*new\s+Point\(([\d\.\-]+),\s*([\d\.\-]+)\))?(?:,\s*new\s+Point\(([\d\.\-]+),\s*([\d\.\-]+)\))?\s*\)/g;

    let match: RegExpExecArray | null;
    let prevDeg = 0;
    let pathIndex = 0;
    // Extract and iterate over each path
    while ((match = pathPattern.exec(importCode)) !== null) {
      const pathName = match[1];
      const headingPart = match[2];
      const headingParams = match[3];
      const point1X = match[4];
      const point1Y = match[5];
      const point2X = match[6];
      const point2Y = match[7];
      const point3X = match[8];
      const point3Y = match[9];

      // Parse heading parameters
      const headingParamsArr = headingParams.split(',').map(param => param.trim());
      let headingType: "linear" | "constant" | "tangential" = 'constant';
      let startDeg: number | undefined;
      let endDeg: number | undefined;
      let reverse: boolean | undefined;

      if (headingPart.startsWith('LINEAR')) {
          headingType = 'linear';
          startDeg = headingParamsArr[0] ? parseFloat(headingParamsArr[0]) : undefined;
          endDeg = headingParamsArr[1] ? parseFloat(headingParamsArr[1]) : undefined;
      } else if (headingPart.startsWith('CONSTANT')) {
          headingType = 'constant';
          startDeg = headingParamsArr[0] ? parseFloat(headingParamsArr[0]) : undefined;
      } else if (headingPart.startsWith('TANGENTIAL')) {
          headingType = 'tangential';
          reverse = headingParamsArr.length > 0 ? headingParamsArr[0] === 'true' : false;
      }

      // Parse points
      const controlPoint1 = point1X && point1Y ? { x: parseFloat(point1X), y: parseFloat(point1Y) } : undefined;
      const controlPoint2 = point2X && point2Y ? { x: parseFloat(point2X), y: parseFloat(point2Y) } : undefined;
      const endPoint = point3X && point3Y ? { x: parseFloat(point3X), y: parseFloat(point3Y) } :
          controlPoint2 || (controlPoint1 ? { x: controlPoint1.x, y: controlPoint1.y } : { x: 0, y: 0 });

      // Ensure lines array has enough elements
      if (pathIndex >= lines.length) {
          lines.push({
              name: '',
              color: getRandomColor(), // Assuming this is a function or value to assign color
              endPoint: { x: 0, y: 0, heading: 'constant', degrees: 0 }, // Default value
              controlPoints: []
          });
      }

      // Update the line object
      lines[pathIndex].name = pathName;
      lines[pathIndex].color = getRandomColor(); // Update color if needed
      lines[pathIndex].endPoint = {
          x: endPoint.x,
          y: endPoint.y,
          heading: headingType,
          startDeg: headingType === 'linear' ? startDeg : undefined,
          endDeg: headingType === 'linear' ? endDeg : undefined,
          degrees: headingType === 'constant' ? (startDeg || prevDeg) : undefined,
          reverse: headingType === 'tangential' ? reverse : undefined
      } as Point;

      if (controlPoint1) {
          lines[pathIndex].controlPoints[0] = controlPoint1;
      }
      if (controlPoint2) {
          lines[pathIndex].controlPoints[1] = controlPoint2;
      }

      // Update prevDeg for the next iteration
      prevDeg = (lines[pathIndex].endPoint.endDeg ?? lines[pathIndex].endPoint.degrees ?? prevDeg) as number;
      pathIndex += 1;
    }
  }
</script>

<div class="flex-1 flex flex-col justify-start items-center gap-2 h-full">
  <div
    class="flex flex-col justify-start items-start w-full rounded-lg bg-neutral-50 dark:bg-neutral-900 shadow-md p-4 overflow-scroll h-full gap-6"
  >
    <div class="flex flex-col w-full justify-start items-start gap-0.5 text-sm">
      <div class="font-semibold">Canvas Options</div>
      <div class="flex flex-row justify-start items-center gap-2">
        <div class="font-extralight">Robot Width:</div>
        <input
          bind:value={robotWidth}
          type="number"
          class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-16"
          step="1"
        />
        <div class="font-extralight">Robot Height:</div>
        <input
          bind:value={robotHeight}
          type="number"
          class="pl-1.5 rounded-md bg-neutral-100 border-[0.5px] focus:outline-none w-16 dark:bg-neutral-950 dark:border-neutral-700"
          step="1"
        />
      </div>
    </div>

    <div class="flex flex-col w-full justify-start items-start gap-0.5 text-sm">
      <div class="font-semibold">Current Robot Position</div>
      <div class="flex flex-row justify-start items-center gap-2">
        <div class="font-extralight">X:</div>
        <div>{x.invert(robotXY.x).toFixed(3)}</div>
        <div class="font-extralight">Y:</div>
        <div>{y.invert(robotXY.y).toFixed(3)}</div>
        <div class="font-extralight">Heading:</div>
        <div>
          {robotHeading.toFixed(0) === "-0"
            ? "0"
            : -robotHeading.toFixed(0)}&deg;
        </div>
      </div>
    </div>

    <div class="flex flex-col w-full justify-start items-start gap-0.5">
      <div class="font-semibold">Start Point</div>
      <div class="flex flex-row justify-start items-center gap-2">
        <div class="font-extralight">X:</div>
        <input
          bind:value={startPoint.x}
          min="0"
          max="144"
          type="number"
          class="pl-1.5 rounded-md bg-neutral-100 border-[0.5px] focus:outline-none w-28 dark:bg-neutral-950 dark:border-neutral-700"
          step="0.1"
        />
        <div class="font-extralight">Y:</div>
        <input
          bind:value={startPoint.y}
          min="0"
          max="144"
          type="number"
          class="pl-1.5 rounded-md bg-neutral-100 border-[0.5px] focus:outline-none w-28 dark:bg-neutral-950 dark:border-neutral-700"
          step="0.1"
        />
      </div>
    </div>

    {#each lines as line, idx}
      <div class="flex flex-col w-full justify-start items-start gap-1">
        <div class="flex flex-row w-full justify-between">
          <div
            class="font-semibold flex flex-row justify-start items-center gap-2"
          > 
          <div class="font-extralight">Name:</div>  <!-- rice -->
            <input 
              bind:value={line.name}
              type="string"
              class="pl-1.5 rounded-md bg-neutral-100 border-[0.5px] focus:outline-none w-28 dark:bg-neutral-950 dark:border-neutral-700"
            />
            
            <div
              class="size-2.5 rounded-full shadow-md"
              style={`background: ${line.color}`}
            />
          </div>
          <div class="flex flex-row justify-end items-center gap-1">
            {#if line.controlPoints.length < 2}
              <button
                title="Add Control Point"
                on:click={() => {
                  const lastControlPoint = line.controlPoints.length > 0 ? line.controlPoints[line.controlPoints.length - 1] : { x: 72, y: 72 };

                  line.controlPoints = [
                    ...line.controlPoints,
                    {
                      x: (lastControlPoint.x + line.endPoint.x) / 2,
                      y: (lastControlPoint.y + line.endPoint.y) / 2
                    },
                  ];
                }}
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke-width={2}
                  class="size-5 stroke-green-500"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M12 4.5v15m7.5-7.5h-15"
                  />
                </svg>
              </button>
            {/if}
            {#if lines.length > 1}
              <button
                title="Remove Line"
                on:click={() => {
                  let _lns = lines;
                  lines.splice(idx, 1);
                  lines = _lns;
                }}
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke-width={2}
                  class="size-5 stroke-red-500"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M15 12H9m12 0a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z"
                  />
                </svg>
              </button>
            {/if}
          </div>
        </div>
        <div class={`h-[0.75px] w-full`} style={`background: ${line.color}`} />
        <div class="flex flex-col justify-start items-start">
          <div class="font-light">End Point:</div>
          <div class="flex flex-row justify-start items-center gap-2">
            <div class="font-extralight">X:</div>
            <input
              class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
              step="0.1"
              type="number"
              min="0"
              max="144"
              bind:value={line.endPoint.x}
            />
            <div class="font-extralight">Y:</div>
            <input
              class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
              step="0.1"
              min="0"
              max="144"
              type="number"
              bind:value={line.endPoint.y}
            />

            <select
              bind:value={line.endPoint.heading}
              class=" rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28 text-sm"
            >
              <option value="constant">Constant</option>
              <option value="linear">Linear</option>
              <option value="tangential">Tangential</option>
            </select>

            {#if line.endPoint.heading === "linear"}
              <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-14"
                step="1"
                type="number"
                min="-180"
                max="180"
                bind:value={line.endPoint.startDeg}
              />
              <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-14"
                step="1"
                type="number"
                min="-180"
                max="180"
                bind:value={line.endPoint.endDeg}
              />
            {:else if line.endPoint.heading === "constant"}
              <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-14"
                step="1"
                type="number"
                min="-180"
                max="180"
                bind:value={line.endPoint.degrees}
              />
            {:else if line.endPoint.heading === "tangential"}
              <p class="text-sm font-extralight">Reverse:</p>
              <input type="checkbox" bind:checked={line.endPoint.reverse} />
            {/if}
          </div>
        </div>
        {#each line.controlPoints as point, idx1}
          <div class="flex flex-col justify-start items-start">
            <div class="font-light">Control Point {idx1 + 1}:</div>
            <div class="flex flex-row justify-start items-center gap-2">
              <div class="font-extralight">X:</div>
              <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                bind:value={point.x}
                min="0"
                max="144"
              />
              <div class="font-extralight">Y:</div>
              <input
                class="pl-1.5 rounded-md bg-neutral-100 dark:bg-neutral-950 dark:border-neutral-700 border-[0.5px] focus:outline-none w-28"
                step="0.1"
                type="number"
                bind:value={point.y}
                min="0"
                max="144"
              />
              <button
                title="Remove Control Point"
                on:click={() => {
                  let _pts = line.controlPoints;
                  _pts.splice(idx1, 1);
                  line.controlPoints = _pts;
                }}
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke-width={2}
                  class="size-5 stroke-red-500"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M15 12H9m12 0a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z"
                  />
                </svg>
              </button>
            </div>
          </div>
        {/each}
      </div>
    {/each}
    <button
      on:click={() => {
        lines = [
          ...lines,
          {
            endPoint: {
              x: _.random(0, 144),
              y: _.random(0, 144),
              heading: "tangential",
              reverse: false,
            },
            controlPoints: [],
            color: getRandomColor(),
            name: "Line " + (lines.length + 1),
          },
        ];
      }}
      class="font-semibold text-green-500 text-sm flex flex-row justify-start items-center gap-1"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
        viewBox="0 0 24 24"
        stroke-width={2}
        stroke="currentColor"
        class="size-5"
      >
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          d="M12 4.5v15m7.5-7.5h-15"
        />
      </svg>
      <p>Add Line</p>
    </button>
    

    <div class="font-extralight font-semibold flex flex-col w-full gap-0.5 text-sm">Code:</div> <!-- rice -->
    {#each lines as line}
      <div>{getCode(line)}</div>
    {/each}
    
    <div class="font-extralight font-semibold flex flex-col w-full gap-0.5 text-sm">Code Import:</div>  <!-- rice -->
      <input 
        bind:value={importCode}
        
        on:input={parseImportCode}
        type="string"
        class="font-extralight font-semibold flex flex-col w-full gap-0.5 text-sm focus:outline-none dark:bg-neutral-950 dark:border-neutral-700"
      />  
        
    
  </div>
  
  
  <div
    class="w-full bg-neutral-50 dark:bg-neutral-900 rounded-lg p-3 flex flex-row justify-start items-center gap-3 shadow-lg"
  >
  
    <button
      title="Play/Pause"
      on:click={() => {
        if (playing) {
          pause();
        } else {
          play();
        }
      }}
    >
      {#if !playing}
        <svg
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          stroke-width="2"
          stroke="currentColor"
          class="size-6 stroke-green-500"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M5.25 5.653c0-.856.917-1.398 1.667-.986l11.54 6.347a1.125 1.125 0 0 1 0 1.972l-11.54 6.347a1.125 1.125 0 0 1-1.667-.986V5.653Z"
          />
        </svg>
      {:else}
        <svg
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          stroke-width="2"
          stroke="currentColor"
          class="size-6 stroke-green-500"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M15.75 5.25v13.5m-7.5-13.5v13.5"
          />
        </svg>
      {/if}
    </button>
    <input
      bind:value={percent}
      type="range"
      min="0"
      max="100"
      step="0.000001"
      class="w-full appearance-none slider focus:outline-none"
    />
  </div>
</div>
