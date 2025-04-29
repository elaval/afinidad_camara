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

<div class="row">
  <div class="col-3">
  ${persona1.nombreCompleto}<br>
${html`<img width=100 src="${imageURL}" class="rounded-circle">`}

  </div>  
  <div class="col">
  <h4>Militancia</h4>
  ${persona1.militancias.map(d => html`<li>${moment(d.fechaInicio).format("D MMM YYYY")} - ${moment(d.fechaTermino).format("D MMM YYYY")}: ${d.nombrePartido}</li>`)}
  </div>
</div>

```sql id=[registrosVotos]
WITH data1 as (SELECT *
FROM votos
  LEFT JOIN votaciones ON votos.idVotacion = votaciones.idVotacion
WHERE votos.idDiputado = ${persona1.idDiputado} AND fecha > '2022-03-10')

SELECT count(*) as votaciones,
  SUM(CASE WHEN voto = 'Afirmativo' THEN 1 ELSE 0 END)::Int as Afirmativo,
  SUM(CASE WHEN voto = 'En Contra' THEN 1 ELSE 0 END)::Int as EnContra,
  SUM(CASE WHEN voto = 'Abstención' THEN 1 ELSE 0 END)::Int as Abstención,
  SUM(CASE WHEN voto = 'Dispensado' THEN 1 ELSE 0 END)::Int as Dispensado,
  SUM(CASE WHEN voto = 'No Vota' THEN 1 ELSE 0 END)::Int as NoVota,
FROM data1
```

```sql id=[rangoFechasVotaciones]
WITH data1 as (SELECT *
FROM votos
  LEFT JOIN votaciones ON votos.idVotacion = votaciones.idVotacion
WHERE votos.idDiputado = ${persona1.idDiputado} AND fecha > '2022-03-10')

SELECT MIN(fecha) as minFecha, MAX(fecha) as maxFecha
FROM data1
```

<div class="card">
Entre ${moment(rangoFechasVotaciones.minFecha).format("DD MMM YYYY")} y ${moment(rangoFechasVotaciones.maxFecha).format("DD MMM YYYY")} hay ${registrosVotos.votaciones} votaciones con registros de ${persona1.nombre1} ${persona1.apellidoPaterno}
<li> ${registrosVotos.Afirmativo} voto Afirmativo  
<li> ${registrosVotos.EnContra} voto En Contra
<li> ${registrosVotos.Abstención} voto Abstención
<li> ${registrosVotos.Dispensado} voto Dispensado
<li> ${registrosVotos.NoVota} No Vota
</div>

## Diputadas y diputados con mayor afinidad en votaciones

```js
const screenWidth = width
const umbralWidth = 700;
const tileWidth = screenWidth > umbralWidth ? umbralWidth / 10 : screenWidth / 5
```

```js
(() => {
  const top10 = _.chain(dataPlot)
    .sortBy((d) => d.afinidad)
    .reverse()
    .slice(1, 11)
    .value();


  return html`
<div class="row">
${top10.map(
  (d) =>
    html`<div class="col flex-grow-0" style="min-width:${tileWidth}px"><img src="${`https://www.camara.cl/img.aspx?prmID=GRCL${d.idDiputado}`}" class="rounded-circle"><br>${
      d.nombre
    }</div>`
)}
</div>`;
})()
```

## Diputadas y diputados con menor afinidad en votaciones
```js
(() => {
  const bottom10 = _.chain(dataPlot)
    .sortBy((d) => d.afinidad)
    .reverse()
    .slice(-10)
    .value();

  const umbralWidth = 800;

  return html`
<div class="row">
${bottom10.map(
  (d) =>
    html`<div class="col flex-grow-0 small" style="min-width:${tileWidth}px"><img  src="${`https://www.camara.cl/img.aspx?prmID=GRCL${d.idDiputado}`}" class="rounded-circle"><br>${
      d.nombre
    }</div>`
)}
</div>`;
})()
```

<div class="row">

<div class="card col">

${chart}

</div>

</div>

```js
const chart = (() => {
  const chart = Plot.plot({
    height: 780,
    width:screenWidth,
    marginTop: 50,
    y: { reverse: true, label: "Distancia", tickFormat: "%" },
    marks: [
      Plot.image(
        dataPlot.filter((d) => d.idDiputado !== persona1.idDiputado),
        Plot.dodgeX(
          { anchor: "middle" },
          {
            y: (d) => 1 - d["afinidad"],
            r: 15,
            src: "imageUrl",
            preserveAspectRatio: "xMidYMin slice", // try not to clip heads
            tip: true,
            title: (d) => `${d.nombre}\ndistancia: ${d.distancia}`,
            channels: { nombre: "nombre", distancia: "distancia" }
          }
        )
      ),
      Plot.image(
        dataPlot.filter((d) => d.idDiputado == persona1.idDiputado),
        Plot.dodgeX(
          { anchor: "middle" },
          {
            y: (d) => 1 - d["afinidad"],
            r: 20,
            src: "imageUrl",
            preserveAspectRatio: "xMidYMin slice", // try not to clip heads

            tip: false,
            channels: { nombre: "nombre" }
          }
        )
      ),
      Plot.text(
        dataPlot.filter((d) => d.idDiputado == persona1.idDiputado),
        Plot.dodgeX(
          { anchor: "middle" },
          {
            y: (d) => 1 - d["afinidad"],
            text: "nombre",
            dy: -30
          }
        )
      )
    ]
  });
  d3.select(chart).selectAll("image").on("mouseup", onclick);
  return chart;
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

    input.blur(); // Dismiss keyboard (mobile)
    form.dispatchEvent(new CustomEvent("input"));
  };

  // Handle input event for suggestions
  input.oninput = async (event) => {
    const value = event.target.value.trim();
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
    input.value = ""; // Clear the input so the user can type a new search
  };

  return form;
}


```

```js
import moment from 'npm:moment'
```