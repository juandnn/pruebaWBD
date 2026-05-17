---

title: Example report
---

<style> .observablehq, .observablehq main, .observablehq-center, .observablehq article { width: 100%; max-width: calc(100%) !important; } .observablehq p, .observablehq h1, .observablehq h2, .observablehq h3, .observablehq ul, .observablehq ol, .observablehq li, .observablehq blockquote, .observablehq table, .observablehq pre { width: 100%; max-width: none !important; box-sizing: border-box; } </style>

---

# ¿Qué está pasando con Discovery Kids entre 2018 y 2019?

En esta página se pretende encontrar Insights sobre la situación de Discovery Kids entre 2018 y 2019. Para ello se va a utilizar la base de datos del panorama de televisión infantil, que para 18 canales diferentes de televisión colombiana muestra de forma diaria su rating (expresado en miles de personas) y el porcentaje de televidentes que obtuvieron del total posible. 

---


```js
const data_dicc = await FileAttachment("data/diccionario_completo.json").json();
const data_panorama = await FileAttachment("data/panorama_tv_infantil.csv").csv();
const data_panorama_mes = await FileAttachment("data/panorama_tv_infantil_agrupado_mes.csv").csv();
const data_total= await FileAttachment("data/totales_discovery_kids.csv").csv();

```
# I. Situación general de la televisión colombiana

Selecciona uno o varios segmentos para compararlos en la misma grafica. Al hacer click en cualquier punto se muestra el valor exacto correspondiente.

```js
const edadesSeleccionadas = view(
  Inputs.checkbox(
    ["Total Niños 4-8 años", "Total Amas de casa 18+", "Total Personas 4+"],
    {
      label: "¿Qué edades quieres mostrar?",
      value: ["Total Niños 4-8 años", "Total Amas de casa 18+", "Total Personas 4+"]
    }
  )
);
edadesSeleccionadas;

```

```js
function graficaTotalesDiscovery(data_total, edadesSeleccionadas) {
  const mesesNombre = ["Ene","Feb","Mar","Abr","May","Jun","Jul","Ago","Sep","Oct","Nov","Dic"];

  const datos = data_total.flatMap(d =>
    edadesSeleccionadas.map(categoria => ({
      fecha: new Date(+d.Año, +d.Mes - 1, 1),
      año: +d.Año,
      mes: +d.Mes,
      categoria,
      valor: +d[categoria]
    }))
  );

  const width = 1500;
  const baseHeight = 260;
  const perCategory = 55;
  const height = baseHeight + edadesSeleccionadas.length * perCategory;
  const margin = { top: 30, right: 180, bottom: 80, left: 85 };

  const COLORS = ["#60a5fa","#34d399","#fb923c","#f472b6","#a78bfa","#facc15"];

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .style("max-width", "100%")
    .style("height", "auto")
    .style("font-family", "sans-serif");

  const x = d3.scaleTime()
    .domain(d3.extent(datos, d => d.fecha))
    .range([margin.left, width - margin.right]);

  const y = d3.scaleLinear()
    .domain([0, d3.max(datos, d => d.valor) * 1.08])
    .nice()
    .range([height - margin.bottom, margin.top]);

  const color = d3.scaleOrdinal()
    .domain(edadesSeleccionadas)
    .range(COLORS);

  // Grid lines
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).ticks(6).tickSize(-(width - margin.left - margin.right)).tickFormat(""))
    .call(g => g.select(".domain").remove())
    .call(g => g.selectAll("line")
      .attr("stroke", "rgba(255,255,255,0.08)")
      .attr("stroke-dasharray", "4,3"));

  // Eje X
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(
      d3.axisBottom(x)
        .ticks(d3.timeMonth.every(2))
        .tickFormat(d => `${mesesNombre[d.getMonth()]} ${d.getFullYear()}`)
    )
    .call(g => g.select(".domain").attr("stroke", "rgba(255,255,255,0.25)"))
    .call(g => g.selectAll("line").attr("stroke", "rgba(255,255,255,0.25)"))
    .selectAll("text")
      .attr("transform", "rotate(-35)")
      .style("text-anchor", "end")
      .style("fill", "white")
      .style("font-size", "11px");

  // Eje Y
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).ticks(6).tickFormat(d => d >= 1000 ? `${(d/1000).toFixed(0)}k` : d))
    .call(g => g.select(".domain").attr("stroke", "rgba(255,255,255,0.25)"))
    .call(g => g.selectAll("line").attr("stroke", "rgba(255,255,255,0.25)"))
    .call(g => g.selectAll("text")
      .style("fill", "white")
      .style("font-size", "11px"));

  // Labels de ejes
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", height - 8)
    .attr("text-anchor", "middle")
    .style("fill", "rgba(255,255,255,0.6)")
    .style("font-size", "12px")
    .text("Mes");

  svg.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -(height / 2))
    .attr("y", 22)
    .attr("text-anchor", "middle")
    .style("fill", "rgba(255,255,255,0.6)")
    .style("font-size", "12px")
    .text("Total espectadores");

  const line = d3.line()
    .x(d => x(d.fecha))
    .y(d => y(d.valor))
    .curve(d3.curveCatmullRom.alpha(0.5));

  const porCategoria = d3.group(datos, d => d.categoria);

  // Área bajo la línea (sutil)
  const area = d3.area()
    .x(d => x(d.fecha))
    .y0(height - margin.bottom)
    .y1(d => y(d.valor))
    .curve(d3.curveCatmullRom.alpha(0.5));

  svg.selectAll(".area")
    .data(porCategoria)
    .join("path")
    .attr("fill", d => color(d[0]))
    .attr("opacity", 0.07)
    .attr("d", d => area(d[1]));

  svg.selectAll(".linea")
    .data(porCategoria)
    .join("path")
    .attr("fill", "none")
    .attr("stroke", d => color(d[0]))
    .attr("stroke-width", 2.2)
    .attr("stroke-linejoin", "round")
    .attr("d", d => line(d[1]));

  // --- Tooltip ---
  const tooltipG = svg.append("g").style("display", "none").style("pointer-events", "none");

  const tooltipW = 200, tooltipH = 76;

  const tooltipRect = tooltipG.append("rect")
    .attr("width", tooltipW)
    .attr("height", tooltipH)
    .attr("rx", 8)
    .attr("fill", "#1e293b")
    .attr("stroke", "rgba(255,255,255,0.12)")
    .attr("stroke-width", 1);

  // Barra de color superior del tooltip
  const tooltipBar = tooltipG.append("rect")
    .attr("width", tooltipW)
    .attr("height", 4)
    .attr("rx", 8)
    .attr("fill", "white");

  const tCategoria = tooltipG.append("text")
    .attr("x", 12).attr("y", 22)
    .style("font-size", "11px")
    .style("font-weight", "600")
    .style("fill", "white");

  const tFecha = tooltipG.append("text")
    .attr("x", 12).attr("y", 42)
    .style("font-size", "11px")
    .style("fill", "rgba(255,255,255,0.6)");

  const tValor = tooltipG.append("text")
    .attr("x", 12).attr("y", 62)
    .style("font-size", "13px")
    .style("font-weight", "600")
    .style("fill", "white");

  // Línea vertical de referencia
  const refLine = svg.append("line")
    .attr("stroke", "rgba(255,255,255,0.2)")
    .attr("stroke-width", 1)
    .attr("stroke-dasharray", "4,3")
    .attr("y1", margin.top)
    .attr("y2", height - margin.bottom)
    .style("display", "none")
    .style("pointer-events", "none");

  let activePoint = null;

  // Puntos
  svg.selectAll(".punto")
    .data(datos)
    .join("circle")
    .attr("class", "punto")
    .attr("cx", d => x(d.fecha))
    .attr("cy", d => y(d.valor))
    .attr("r", 5)
    .attr("fill", d => color(d.categoria))
    .attr("stroke", "rgba(0,0,0,0.4)")
    .attr("stroke-width", 1.5)
    .style("cursor", "pointer")
    .on("mouseenter", function(event, d) {
      d3.select(this).transition().duration(120).attr("r", 8);
    })
    .on("mouseleave", function(event, d) {
      if (activePoint !== this) d3.select(this).transition().duration(120).attr("r", 5);
    })
    .on("click", function(event, d) {
      if (activePoint && activePoint !== this) {
        d3.select(activePoint).transition().duration(120).attr("r", 5);
      }
      activePoint = this;
      d3.select(this).attr("r", 8);

      const cx = x(d.fecha);
      const cy = y(d.valor);
      const flipX = cx + tooltipW + 20 > width - margin.right;
      const tx = flipX ? cx - tooltipW - 14 : cx + 14;
      const ty = Math.max(margin.top, Math.min(cy - tooltipH / 2, height - margin.bottom - tooltipH));

      refLine
        .style("display", null)
        .attr("x1", cx).attr("x2", cx);

      tooltipG
        .style("display", null)
        .attr("transform", `translate(${tx},${ty})`);

      tooltipBar.attr("fill", color(d.categoria));

      tCategoria.text(d.categoria);
      tFecha.text(`${mesesNombre[d.mes - 1]} ${d.año}`);
      tValor.text(d3.format(",.2f")(d.valor));
    });

  // Click fuera para cerrar tooltip
  svg.on("click", function(event) {
    if (!event.target.classList.contains("punto")) {
      tooltipG.style("display", "none");
      refLine.style("display", "none");
      if (activePoint) {
        d3.select(activePoint).transition().duration(120).attr("r", 5);
        activePoint = null;
      }
    }
  });

  // Leyenda
  const legend = svg.append("g")
    .attr("transform", `translate(${width - margin.right + 24},${margin.top})`);

  const legendItems = legend.selectAll("g")
    .data(edadesSeleccionadas)
    .join("g")
    .attr("transform", (_, i) => `translate(0,${i * 28})`);

  legendItems.append("circle")
    .attr("cx", 7).attr("cy", 7).attr("r", 6)
    .attr("fill", d => color(d))
    .attr("opacity", 0.9);

  legendItems.append("text")
    .attr("x", 20).attr("y", 11)
    .style("font-size", "11px")
    .style("fill", "rgba(255,255,255,0.85)")
    .text(d => d);

  return svg.node();
}



```

```js

graficaTotalesDiscovery(data_total, edadesSeleccionadas)

```


# II. heatmap calendario



```js 
const edadesSeleccionadas2 = view(
  Inputs.select(
    ["Niños 4-8 años", "Amas de casa 18+", "Personas 4+"],
    {
      label: "¿Qué edad quieres mostrar?",
      value: "Niños 4-8 años"
    }
  )
);
edadesSeleccionadas2;

```

```js 
const metricaCalendario = view(
  Inputs.radio(["Share", "Rating en Miles"], {
    label: "¿Qué métrica quieres ver?",
    value: "Share"
  })
);
metricaCalendario;

```

```js 

function calendarioHeatmapDiscovery(data_panorama, edadesSeleccionadas2, metricaCalendario) {
  const edadSeleccionada = Array.isArray(edadesSeleccionadas2)
    ? edadesSeleccionadas2[0]
    : edadesSeleccionadas2;
  const anios = [2018, 2019];

  const datos = data_panorama
    .filter(d => d.Canal === "Discovery Kids")
    .map(d => {
      const fecha = new Date(d.Fecha + "T00:00:00");

      return {
        año: +d.Año,
        fecha,
        mes: +d.Mes,
        diaSemana: fecha.getDay(),
        semana: d3.utcWeek.count(d3.utcYear(fecha), fecha),
        edad: edadSeleccionada,
        valor: +d[`${metricaCalendario} ${edadSeleccionada}`]
      };
    })
    .filter(d => anios.includes(d.año))
    .filter(d => !isNaN(d.valor));

  const cellSize = 14;
  const yearWidth = cellSize * 54;
  const yearHeight = cellSize * 7 + 55;
  const margin = { top: 70, right: 30, bottom: 55, left: 45 };
  const yearGap = 44;

  const width = margin.left + margin.right + yearWidth;
  const height = margin.top + margin.bottom + yearHeight * anios.length + yearGap * (anios.length - 1);
  const extent = d3.extent(datos, d => d.valor);
  const colorDomain = extent[0] === extent[1]
    ? [extent[0] - 1, extent[1] + 1]
    : extent;

  const color = d3.scaleSequential()
    .domain(colorDomain)
    .interpolator(d3.interpolateRdYlGn);

  const wrapper = document.createElement("div");

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .style("max-width", "100%")
    .style("height", "auto")
    .style("font-family", "sans-serif");

  const dias = ["Dom", "Lun", "Mar", "Mié", "Jue", "Vie", "Sáb"];
  const meses = ["Ene", "Feb", "Mar", "Abr", "May", "Jun", "Jul", "Ago", "Sep", "Oct", "Nov", "Dic"];

  svg.append("text")
    .attr("x", margin.left)
    .attr("y", 26)
    .attr("font-size", 16)
    .attr("font-weight", "bold")
    .attr("fill", "white")
    .text(edadSeleccionada);

  anios.forEach((año, yearIndex) => {
    const xBase = margin.left;
    const yBase = margin.top + yearIndex * (yearHeight + yearGap);

    svg.append("text")
      .attr("x", xBase)
      .attr("y", yBase - 5)
      .attr("font-size", 14)
      .attr("font-weight", "bold")
      .attr("fill", "white")
      .text(año);

    svg.append("g")
      .selectAll("text")
      .data(dias)
      .join("text")
      .attr("x", xBase - 8)
      .attr("y", (_, i) => yBase + i * cellSize + cellSize * 0.75)
      .attr("text-anchor", "end")
      .attr("font-size", 10)
      .attr("fill", "white")
      .text(d => d);

    const datosAnioEdad = datos.filter(d => d.año === año && d.edad === edadSeleccionada);

    svg.append("g")
      .selectAll("rect")
      .data(datosAnioEdad)
      .join("rect")
      .attr("x", d => xBase + d.semana * cellSize)
      .attr("y", d => yBase + d.diaSemana * cellSize)
      .attr("width", cellSize - 1)
      .attr("height", cellSize - 1)
      .attr("fill", d => color(d.valor))
      .attr("stroke", "white")
      .style("cursor", "pointer")
      .on("click", function(event, d) {
        infoPanel.innerHTML = `
          <strong>Discovery Kids</strong>
          <span>${d.fecha.toISOString().slice(0, 10)} | ${d.edad}</span>
          <span>${metricaCalendario}: ${d3.format(",.2f")(d.valor)}</span>
        `;
      });

    const primerDiaMes = d3.utcMonths(
      new Date(Date.UTC(año, 0, 1)),
      new Date(Date.UTC(año + 1, 0, 1))
    );

    svg.append("g")
      .selectAll("text")
      .data(primerDiaMes)
      .join("text")
      .attr("x", d => {
        const fecha = new Date(d);
        const semana = d3.utcWeek.count(d3.utcYear(fecha), fecha);
        return xBase + semana * cellSize;
      })
      .attr("y", yBase - 8)
      .attr("font-size", 10)
      .attr("fill", "white")
      .text((_, i) => meses[i]);
  });

  const legendWidth = 220;
  const legendHeight = 10;
  const legendId = `legend-gradient-${Math.random().toString(36).slice(2)}`;

  const defs = svg.append("defs");

  const gradient = defs.append("linearGradient")
    .attr("id", legendId);

  gradient.selectAll("stop")
    .data(d3.range(0, 1.01, 0.1))
    .join("stop")
    .attr("offset", d => `${d * 100}%`)
    .attr("stop-color", d => color(
      color.domain()[0] + d * (color.domain()[1] - color.domain()[0])
    ));

  const legendX = margin.left;
  const legendY = height - 25;

  svg.append("rect")
    .attr("x", legendX)
    .attr("y", legendY)
    .attr("width", legendWidth)
    .attr("height", legendHeight)
    .attr("fill", `url(#${legendId})`);

  const legendScale = d3.scaleLinear()
    .domain(color.domain())
    .range([legendX, legendX + legendWidth]);

  svg.append("g")
    .attr("transform", `translate(0,${legendY + legendHeight})`)
    .call(d3.axisBottom(legendScale).ticks(5).tickFormat(d3.format(".2f")))
    .call(g => g.selectAll("text").attr("fill", "white"))
    .call(g => g.selectAll("line, path").attr("stroke", "white"));

  const infoPanel = document.createElement("div");
  infoPanel.style.marginTop = "12px";
  infoPanel.style.padding = "12px 14px";
  infoPanel.style.border = "1px solid rgba(255,255,255,0.28)";
  infoPanel.style.borderRadius = "6px";
  infoPanel.style.color = "white";
  infoPanel.style.fontFamily = "sans-serif";
  infoPanel.style.fontSize = "13px";
  infoPanel.style.display = "grid";
  infoPanel.style.gap = "4px";
  infoPanel.textContent = "Haz click en un día del calendario para ver el detalle.";

  wrapper.append(svg.node(), infoPanel);
  return wrapper;
}
```

```js 

calendarioHeatmapDiscovery(data_panorama, edadesSeleccionadas2, metricaCalendario)
```


# III. Gráfica 3: Mosaic plot por grupo de edad. 

```js
const metricaCalendario2 = view(
  Inputs.radio(["Share", "Rating en Miles"], {
    label: "Métrica",
    value: "Share"
  })
);
```

```js
const edadesSeleccionadas3 = view(
  Inputs.radio(["Niños 4-8 años", "Amas de casa 18+", "Personas 4+"], {
    label: "Segmento de edad",
    value: "Niños 4-8 años"
  })
);
```

```js
const agrupacionMosaic = view(
  Inputs.radio(["semana", "mes", "año", "rango completo"], {
    label: "Agrupar por",
    value: "mes"
  })
);
```

```js
function mosaicPlotCanales(data_panorama, metrica, edad, agrupacion) {
  const columna = `${metrica} ${edad}`;

  function clavePeriodo(fecha) {
    const d = new Date(fecha);
    if (agrupacion === "rango completo") return "2018-2019";
    if (agrupacion === "día") return d3.timeFormat("%Y-%m-%d")(d);
    if (agrupacion === "semana") return `${d.getFullYear()}-S${String(d3.timeFormat("%U")(d)).padStart(2,"0")}`;
    if (agrupacion === "mes") return d3.timeFormat("%Y-%m")(d);
    return `${d.getFullYear()}`;
  }

  const datosBase = data_panorama
    .map(d => ({
      canal: d.Canal,
      fecha: new Date(d.Fecha + "T00:00:00"),
      valor: +d[columna]
    }))
    .filter(d => !isNaN(d.valor))
    .map(d => ({ ...d, periodo: clavePeriodo(d.fecha) }));

  if (datosBase.length === 0) return htl.html`<p style="color:white">Sin datos disponibles.</p>`;

  // Promedios por (periodo, canal)
  const datosAgrupados = Array.from(
    d3.rollup(datosBase, v => d3.mean(v, d => d.valor), d => d.periodo, d => d.canal),
    ([periodo, canales]) => Array.from(canales, ([canal, valor]) => ({ periodo, canal, valor }))
  ).flat();

  const periodos = Array.from(new Set(datosAgrupados.map(d => d.periodo))).sort();
  const canales  = Array.from(new Set(datosAgrupados.map(d => d.canal))).sort();

  // Suma total por periodo (determina el ancho de cada columna)
  const totalPorPeriodo = new Map(
    periodos.map(p => [p, d3.sum(datosAgrupados.filter(d => d.periodo === p), d => d.valor)])
  );
  const totalGlobal = d3.sum(Array.from(totalPorPeriodo.values()));

  const width  = Math.max(2000, periodos.length * 42);
  const height = 700;
  const margin = { top: 52, right: 200, bottom: 88, left: 20 };
  const innerW = width - margin.left - margin.right;
  const innerH = height - margin.top - margin.bottom;

  const xScale = d3.scaleLinear()
    .domain([0, totalGlobal])
    .range([margin.left, margin.left + innerW]);

  const PALETTE = [
    "#60a5fa","#34d399","#fb923c","#f472b6",
    "#a78bfa","#facc15","#38bdf8","#4ade80",
    "#f87171","#e879f9","#fbbf24","#2dd4bf"
  ];
  const color = d3.scaleOrdinal().domain(canales).range(PALETTE);

  const svg = d3.create("svg")
    .attr("width", width).attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .style("max-width", "100%").style("height", "auto")
    .style("font-family", "sans-serif");

  const wrapper = document.createElement("div");

  // Título
  svg.append("text")
    .attr("x", margin.left).attr("y", 30)
    .attr("fill", "white").attr("font-size", 15).attr("font-weight", "600")
    .text(`${metrica} · ${edad} · agrupado por ${agrupacion}`);

  // Rectángulos del mosaico
  let xActual = margin.left;

  periodos.forEach(periodo => {
    const datosPeriodo = datosAgrupados
      .filter(d => d.periodo === periodo)
      .sort((a,b) => d3.descending(a.valor, b.valor));

    const totalPeriodo = totalPorPeriodo.get(periodo);
    const anchoPeriodo = xScale(totalPeriodo) - xScale(0);
    let yActual = margin.top;

    datosPeriodo.forEach(d => {
      const alto = totalPeriodo === 0 ? 0 : innerH * (d.valor / totalPeriodo);

      svg.append("rect")
        .attr("x", xActual + 0.5).attr("y", yActual + 0.5)
        .attr("width",  Math.max(0, anchoPeriodo - 1.5))
        .attr("height", Math.max(0, alto - 1.5))
        .attr("rx", 2)
        .attr("fill", color(d.canal))
        .attr("opacity", 0.88)
        .style("cursor","pointer")
        .on("mouseenter", function() { d3.select(this).attr("opacity", 1); })
        .on("mouseleave", function() { d3.select(this).attr("opacity", 0.88); })
        .on("click", function(event) {
          infoPanel.style.borderLeftColor = color(d.canal);
          infoPanel.innerHTML = `
            <strong>${d.canal}</strong>
            <span>Periodo: ${periodo}</span>
            <span>Promedio: ${d3.format(",.2f")(d.valor)}</span>
            <span>Participación: ${d3.format(".1%")(d.valor / totalPeriodo)}</span>
          `;
          event.stopPropagation();
        });

      // Etiqueta dentro del rectángulo si hay espacio
      if (alto > 16 && anchoPeriodo > 40) {
        svg.append("text")
          .attr("x", xActual + 5).attr("y", yActual + 13)
          .attr("fill","white").attr("font-size", Math.min(10, anchoPeriodo / 6))
          .attr("pointer-events","none")
          .text(d.canal);
      }

      yActual += alto;
    });

    // Etiqueta del periodo (eje X)
    svg.append("text")
      .attr("x", xActual + anchoPeriodo / 2)
      .attr("y", margin.top + innerH + 14)
      .attr("fill","rgba(255,255,255,0.75)").attr("font-size", 10)
      .attr("text-anchor","middle")
      .attr("transform", `rotate(-45,${xActual + anchoPeriodo/2},${margin.top + innerH + 14})`)
      .text(periodo);

    // Separador vertical
    svg.append("line")
      .attr("x1", xActual).attr("x2", xActual)
      .attr("y1", margin.top).attr("y2", margin.top + innerH)
      .attr("stroke","rgba(255,255,255,0.12)").attr("stroke-width",1);

    xActual += anchoPeriodo;
  });

  // Nota de pie
  svg.append("text")
    .attr("x", margin.left + innerW/2).attr("y", height - 10)
    .attr("fill","rgba(255,255,255,0.4)").attr("font-size",10).attr("text-anchor","middle")
    .text("Ancho de columna ∝ suma de promedios del periodo · Alto de celda ∝ participación del canal");

  // Leyenda
  const legend = svg.append("g")
    .attr("transform", `translate(${width - margin.right + 24},${margin.top})`);

  legend.append("text")
    .attr("y", -14).attr("fill","rgba(255,255,255,0.9)")
    .attr("font-size",12).attr("font-weight","700")
    .text("Canal");

  legend.selectAll("g")
    .data(canales).join("g")
    .attr("transform", (_,i) => `translate(0,${i*26})`)
    .call(g => {
      g.append("rect")
        .attr("width",13).attr("height",13).attr("rx",3)
        .attr("fill", d => color(d)).attr("opacity",0.9);
      g.append("text")
        .attr("x",20).attr("y",11)
        .attr("fill","rgba(255,255,255,0.85)").attr("font-size",11)
        .text(d => d);
    });

  const infoPanel = document.createElement("div");
  infoPanel.style.marginTop = "12px";
  infoPanel.style.padding = "12px 14px";
  infoPanel.style.border = "1px solid rgba(255,255,255,0.28)";
  infoPanel.style.borderLeft = "4px solid rgba(255,255,255,0.28)";
  infoPanel.style.borderRadius = "6px";
  infoPanel.style.color = "white";
  infoPanel.style.fontFamily = "sans-serif";
  infoPanel.style.fontSize = "13px";
  infoPanel.style.display = "grid";
  infoPanel.style.gap = "4px";
  infoPanel.textContent = "Haz click en un canal del mosaico para ver el detalle.";

  wrapper.append(svg.node(), infoPanel);
  return wrapper;
}
```

```js
mosaicPlotCanales(
  data_panorama,
  metricaCalendario2,
  edadesSeleccionadas3,
  agrupacionMosaic
)
```
