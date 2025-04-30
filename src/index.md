---
toc: false

sql:
  votaciones: "data/votaciones_completo.parquet"
  votos: "data/votos_detalle_completo.parquet"
---


```js
// Create an autocomplete widget for selecting a user by their handle.
const selectedUser = view(createAutocomplete(_.sortBy(searchOptions, d => d.apellidoPaterno), {
  searchFields: ["nombreCompleto"],
  formatFn: d => `${d.nombreCompleto}`,
  defaultValue: diputados[0].handle, // Change this to an actual handle from your data
  label: "Seleccione diputada o diputado",
  description: null//`Lista compuesta por quienes siguen a ${defaultUser}`
}));
```

```js
const imageURL = `https://www.camara.cl/img.aspx?prmID=GRCL${persona1.idDiputado}`
````

```js
html`
<!-- Tarjeta de perfil ----------------------------------------------------->
<div class="card shadow-sm border-0 mb-4" style="max-width:600px">
  <div class="row g-0 align-items-center">
    <!-- Foto + nombre -->
    <div class="col-auto p-3 text-center">
      <img src="${imageURL}"
           alt="Foto de ${persona1.nombreCompleto}"
           class="rounded-circle img-fluid"
           style="width:96px;height:96px;object-fit:cover">
      <div class="fw-semibold mt-2">${persona1.nombreCompleto}</div>
    </div>

    <!-- Detalle de militancia -->
    <div class="col p-3">
      <h6 class="text-uppercase text-muted mb-2">Militancia</h6>
      <ul class="list-group list-group-flush small">
        ${persona1.militancias.map(d => html`
          <li class="list-group-item px-0 border-0">
            <span class="text-muted me-2">
              ${moment(d.fechaInicio).format("D MMM YYYY")} – ${moment(d.fechaTermino).format("D MMM YYYY")}
            </span>
            <strong>${d.nombrePartido}</strong>
          </li>`)}
      </ul>
    </div>
  </div>
</div>`
```


```sql id=[registrosVotos]
WITH data1 as (SELECT *
FROM votos
  LEFT JOIN votaciones ON votos.idVotacion = votaciones.idVotacion
WHERE votos.idDiputado = ${persona1.idDiputado} AND fecha > '2022-03-10')

SELECT count(*) as votaciones,
  SUM(CASE WHEN voto = 'Afirmativo' THEN 1 ELSE 0 END)::Int as Afirmativo,
  SUM(CASE WHEN voto = 'En Contra' THEN 1 ELSE 0 END)::Int as EnContra,
  SUM(CASE WHEN voto = 'Abstención' THEN 1 ELSE 0 END)::Int as Abstención
FROM data1
WHERE voto IN ('Afirmativo', 'En Contra', 'Abstención')
```

```sql id=[rangoFechasVotaciones]
WITH data1 as (SELECT *
FROM votos
  LEFT JOIN votaciones ON votos.idVotacion = votaciones.idVotacion
WHERE votos.idDiputado = ${persona1.idDiputado} AND fecha > '2022-03-10')

SELECT MIN(fecha) as minFecha, MAX(fecha) as maxFecha
FROM data1
```
<!-- Tarjeta de resumen --------------------------------------------------->
<div class="card shadow-sm border-0 mb-4" style="max-width:650px">
  <div class="card-body">
    <p class="mb-2">
      <strong>${persona1.nombreCompleto}</strong> emitió un voto válido
      (Afirmativo, En Contra o Abstención) en
      <strong>${registrosVotos.votaciones}</strong> votaciones entre
      <span class="text-nowrap">${moment(rangoFechasVotaciones.minFecha).format("D MMM YYYY")}</span>
      y
      <span class="text-nowrap">${moment(rangoFechasVotaciones.maxFecha).format("D MMM YYYY")}</span>
    </p>
    <ul class="list-inline mb-3 fs-6">
      <li class="list-inline-item me-3">
        <span class="badge bg-success me-1">${registrosVotos.Afirmativo}</span>Afirmativo</span>
      </li>
      <li class="list-inline-item me-3">
        <span class="badge bg-danger  me-1">${registrosVotos.EnContra}</span>En Contra</span>
      </li>
      <li class="list-inline-item">
        <span class="badge bg-warning text-dark me-1">${registrosVotos.Abstención}</span>Abstención</span>
      </li>
    </ul>
    <p class="small mb-0">
      Para cada diputada o diputado que participó en las mismas votaciones se
      calcula la <strong>similitud</strong> (porcentaje de coincidencias) y su
      complementaria <strong>distancia</strong>; más abajo se muestran los
      resultados.
    </p>
  </div>
</div>

```js
(() => {
  const top10 = _.chain(dataPlot)
    .sortBy(d => d.afinidad)   // asumimos 1 = idéntico
    .reverse()                 // los más altos primero
    .slice(1, 11)              // descartamos la propia persona
    .value();

  return html`
  <h6 class="text-uppercase text-muted mt-4 mb-2">Mayor afinidad</h6>
  <div class="row row-cols-2 row-cols-sm-3 row-cols-md-5 g-3 text-center">
    ${top10.map(d => html`
      <div class="col">
        <img class="rounded-circle mb-1 shadow-sm"
             src="https://www.camara.cl/img.aspx?prmID=GRCL${d.idDiputado}"
             style="width:72px;height:72px;object-fit:cover">
        <div class="small">${d.nombre}</div>
      </div>`)}
  </div>`;
})()
```


```js
(() => {
  const bottom10 = _.chain(dataPlot)
    .sortBy(d => d.afinidad)   // los más bajos primero
    .slice(0, 10)
    .value();

  return html`
  <h6 class="text-uppercase text-muted mt-4 mb-2">Menor afinidad</h6>
  <div class="row row-cols-2 row-cols-sm-3 row-cols-md-5 g-3 text-center">
    ${bottom10.map(d => html`
      <div class="col">
        <img class="rounded-circle mb-1 shadow-sm"
             src="https://www.camara.cl/img.aspx?prmID=GRCL${d.idDiputado}"
             style="width:72px;height:72px;object-fit:cover">
        <div class="small">${d.nombre}</div>
      </div>`)}
  </div>`;
})()
```


<div class="card shadow-sm border-0 mb-4">
  <div class="card-header bg-light fw-semibold">
    Distancia de votación frente a ${persona1.nombre1} ${persona1.apellidoPaterno}
    <div class="text-muted small fw-normal">
      Distancia = % de votaciones compartidas con votos distintos
    </div>
  </div>
  <div class="card-body p-0">
    ${chart}
  </div>
</div>


```js
const chart = (() => {
  // Alto dinámico: 38-40 px por diputada/o + espacio superior
  const barHeight   = 40;
  //const chartHeight = dataPlot.length * barHeight + 60;
  const chartHeight = 800;

  return Plot.plot({
    height: chartHeight,
    width,                       // variable reactiva de Observable
    marginTop: 50,
    marginLeft: 60,
    marginRight: 20,
    grid: true,                  // líneas horizontales de ayuda
    y: {
      reverse: true,
      label: "Distancia",
      tickFormat: d => d3.format(".0%")(d), // 0 %, 20 %…
      ticks: 6
    },

    marks: [
      /* retratos del resto -----------------------------*/
      Plot.image(
        dataPlot.filter(d => d.idDiputado !== persona1.idDiputado),
        Plot.dodgeX(
          {anchor: "middle"},
          {
            y: d => 1 - d.afinidad,   // distancia = 1 - afinidad
            r: 16,
            src: "imageUrl",
            preserveAspectRatio: "xMidYMin slice",
            title: d => `${d.nombre}\nDistancia: ${d3.format(".0%")(1 - d.afinidad)}`,
            tip: true
          }
        )
      ),

      /* contorno + retrato de la persona objetivo ------*/
      Plot.dot(
        dataPlot.filter(d => d.idDiputado === persona1.idDiputado),
        {y: d => 1 - d.afinidad, r: 22, stroke: "dodgerblue", fill: "none", strokeWidth: 3}
      ),
      Plot.image(
        dataPlot.filter(d => d.idDiputado === persona1.idDiputado),
        {y: d => 1 - d.afinidad, r: 18, src: "imageUrl", preserveAspectRatio: "xMidYMin slice"}
      ),
      Plot.text(
        dataPlot.filter(d => d.idDiputado === persona1.idDiputado),
        {y: d => 1 - d.afinidad, text: "nombre", dy: -28, fontSize: 11, textAnchor: "middle"}
      )
    ]
  });
})();
```




```js
//display(dataPlot)
```


```js
const diputados = FileAttachment("./data/diputados_periodo_actual.json").json();
const matrizAfinidad = FileAttachment("./data/affinity_matrix_legislatura_actual.csv").csv();
```

```js
const dataPlot = (() => {
  const dataAfinidad = _.chain(matrizAfinidad)
    .find((d) => d.idDiputado == persona1.idDiputado)
    .map((item, key) => ({
      idDiputado: key,
      afinidad: +item,
      distancia: d3.format(".1%")(1 - +item),
      nombre: getFullName(key),
      imageUrl: `https://www.camara.cl/img.aspx?prmID=GRCL${key}`
    }))
    .filter((d) => d.idDiputado !== "idDiputado" && d.afinidad > 0)
    .value();

  return dataAfinidad;
})()
```

```js
const searchOptions = diputados.map(d => ({
  handle: getFullName(d.idDiputado),
  nombreCompleto: getFullName(d.idDiputado),
  apellidoPaterno:d.apellidoPaterno
}))
```

```js
const persona1  = diputados.find(
  (d) => getFullName(d.idDiputado) == selectedUser
) || diputados[0]
```

```js
//display(persona1)
```

```js
function getFullName(id) {
  const record = dictDiputado[id];
  return (
    record &&
    `${record.nombre1} ${record.apellidoPaterno} (${record.aliasPartido})`
  );
}
```

```js
const dictDiputado = (() => {
    const dict = {};
  const data = diputados
    .map((d) => {
      d.aliasPartido = getPartido(d);
      return d;
    })
    .forEach((d) => (dict[d.idDiputado] = d));

  return dict;


})()


```

```js
function getPartido(record) {
  const d = record.militancias;

  const recordPartido = !d.length
    ? d
    : _.chain(d)
        .sortBy((d) => d.FechaTermino)
        .last()
        .value();

  return recordPartido.aliasPartido;
}
```


```js

function createAutocomplete(items = [], {
  searchFields = ["handle", "displayName", "description"],
  formatFn = d => `${d.handle} - ${d.displayName} (${d.description})`,
  defaultValue = null,
  label = null,
  description = null,
  placeholderText = "Escribe para buscar...",
  fetchAutoSuggest = async (value) => {
    // Simulate fetching filtered results
    return items.filter(d => searchFields.some(field => {
      const fieldValue = d[field];
      return fieldValue && fieldValue.toString().toLowerCase().includes(value.toLowerCase());
    })).slice(0, 10); // Limit results
  }
} = {}) {
  // Create the form container
  const form = document.createElement("form");
  form.style.position = "relative";
  form.style.width = "100%";
  form.style.fontFamily = "sans-serif";
  form.onsubmit = (event) => event.preventDefault();

  // Add label if provided
  if (label) {
    const labelEl = document.createElement("label");
    labelEl.textContent = label;
    labelEl.style.display = "block";
    labelEl.style.marginBottom = "4px";
    labelEl.style.fontWeight = "bold";
    form.appendChild(labelEl);
  }

  // Add description if provided
  if (description) {
    const descEl = document.createElement("div");
    descEl.textContent = description;
    descEl.style.fontSize = "0.9em";
    descEl.style.color = "#555";
    descEl.style.marginBottom = "8px";
    form.appendChild(descEl);
  }

  // Create the input field
  const input = document.createElement("input");
  input.type = "text";
  input.placeholder = placeholderText;
  input.style.width = "100%";
  input.style.boxSizing = "border-box";
  input.style.padding = "8px";
  input.style.border = "1px solid #ccc";
  input.style.borderRadius = "4px";
  input.value = defaultValue || "";
  form.appendChild(input);

  // Create the datalist for suggestions
  const datalist = document.createElement("datalist");
  datalist.id = `autosuggest-results-${Math.random().toString(36).substr(2, 9)}`;
  input.setAttribute("list", datalist.id);
  form.appendChild(datalist);

  // Store the current value
  form.value = defaultValue || "";

  // Handle input change (selection made from the dropdown)
  input.onchange = (event) => {
    const value = event.target.value;

    // Find the selected item and set the handle only
    const selectedItem = items.find(d => formatFn(d) === value);
    if (selectedItem) {
      form.value = selectedItem.handle; // Set the handle as the value
      input.value = selectedItem.handle; // Update input to show only the handle
    } else {
      form.value = value; // Fallback to raw input value
    }

    // ---- close the datalist dropdown on desktop ----
    datalist.innerHTML = "";          // remove every option
    input.removeAttribute("list");    // detach, forces the overlay to disappear
    // re-attach on the next tick so autocomplete keeps working
    setTimeout(() => {
      input.setAttribute("list", datalist.id);
      input.blur();                   // SACA el foco del campo
    }, 0);

    //input.blur(); // Dismiss keyboard (mobile)
    form.dispatchEvent(new CustomEvent("input"));
  };

  // Handle input event for suggestions
  input.oninput = async (event) => {
    const value = event.target.value.trim();
    if (!value) return;

    // Always clear previous suggestions first
    datalist.innerHTML = "";
    if (!value) return;

    // Fetch suggestions
    const results = await fetchAutoSuggest(value);

    // Clear existing suggestions
    datalist.innerHTML = "";

    // Populate datalist with new suggestions
    results.forEach((result) => {
      const formattedResult = formatFn(result);
      const option = document.createElement("option");
      option.value = formattedResult; // Full formatFn output
      datalist.appendChild(option);
    });
  };

  // Clear input on focus
  input.onfocus = () => {
    if (input.value === form.value) input.value = "";
  };


  return form;
}


```

```js
import moment from 'npm:moment'
```