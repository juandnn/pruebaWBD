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