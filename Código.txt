/**
 * Web App de analítica de datos en Google Apps Script
 * Versión mejorada con limpieza, análisis descriptivo, K-Means,
 * exportación CSV profesional e informe PDF.
 * Archivo: Código.gs
 *
 * Desarrollado por: Ahren Kahi Churta
 */

/* =========================
 * PUNTO DE ENTRADA WEB APP
 * ========================= */

function doGet() {
  return HtmlService
    .createHtmlOutputFromFile('Index')
    .setTitle('Asistente Analítico GAS')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}

/* =========================
 * CONFIGURACIÓN INICIAL
 * ========================= */

function getInitialConfig() {
  return {
    appName: 'Asistente Analítico en Google Apps Script',
    version: '3.0.0',
    author: 'Ahren Kahi Churta',
    processes: [
      'Registrar referencia',
      'Cargar datos',
      'Ejecutar limpieza',
      'Análisis descriptivo',
      'Ver adaptación con K-Means',
      'Informe PDF',
      'Exportar CSV limpio'
    ]
  };
}

/* =========================
 * REFERENCIA PYTHON
 * ========================= */

function saveReferenceContext(functionName, pythonCode) {
  functionName = sanitizeText(functionName);
  pythonCode   = String(pythonCode || '').trim();

  if (!functionName) throw new Error('Debe ingresar el nombre de la función.');
  if (!pythonCode)   throw new Error('Debe pegar el código Python de referencia.');

  const props = PropertiesService.getUserProperties();
  props.setProperty('FUNCTION_NAME',      functionName);
  props.setProperty('PYTHON_CODE',        pythonCode);
  props.setProperty('REFERENCE_SAVED_AT', new Date().toISOString());

  return {
    ok:           true,
    message:      'Referencia registrada correctamente.',
    functionName: functionName,
    pythonLines:  pythonCode.split('\n').length
  };
}

function getReferenceContext() {
  const props = PropertiesService.getUserProperties();
  return {
    functionName: props.getProperty('FUNCTION_NAME')      || '',
    pythonCode:   props.getProperty('PYTHON_CODE')        || '',
    savedAt:      props.getProperty('REFERENCE_SAVED_AT') || ''
  };
}

/* =========================
 * CARGA DE DATOS
 * ========================= */

function loadCsvData(csvText) {
  const rows       = parseCsvText(csvText);
  const normalized = normalizeRows(rows);

  return {
    ok:        true,
    message:   'Datos cargados correctamente.',
    totalRows: normalized.length,
    columns:   normalized.length ? Object.keys(normalized[0]) : [],
    preview:   normalized.slice(0, 10)
  };
}

/* =========================
 * LIMPIEZA
 * ========================= */

function runLimpieza(csvText) {
  const rawRows = parseCsvText(csvText);
  let data      = normalizeRows(rawRows);

  validateRequiredColumns(data, ['Nombres', 'grupo']);

  // Eliminar registros con Nombres nulos o vacíos
  data = data.filter(function(row) {
    const nombre = row.Nombres;
    if (nombre === null || nombre === undefined) return false;
    const val = String(nombre).trim().toLowerCase();
    return val !== '' && val !== 'na' && val !== 'nan' && val !== 'null';
  });

  // Normalizar texto de Nombres
  data = data.map(function(row) {
    const cloned = cloneObject(row);
    cloned.Nombres = String(cloned.Nombres || '')
      .replace(/[^a-zA-Z0-9áéíóúÁÉÍÓÚñÑüÜ ]/g, '')
      .replace(/\s+/g, ' ')
      .trim();
    return cloned;
  });

  // Generar notas simuladas con fórmula ponderada
  data = data.map(function(row) {
    const cloned = cloneObject(row);
    cloned.N1 = round1(clip(randomNormal(3.5, 0.5), 1.0, 5.0));
    cloned.N2 = round1(randomUniform(1.0, 5.0));
    cloned.N3 = round1(randomUniform(4.0, 5.0));
    cloned.NF = round1(cloned.N1 * 0.3 + cloned.N2 * 0.3 + cloned.N3 * 0.4);
    return cloned;
  });

  // Eliminar duplicados por Nombres + grupo
  data = dropDuplicatesByKeys(data, ['Nombres', 'grupo']);

  // Renombrar columnas a nombres finales
  data = data.map(function(row) {
    return {
      Nombres_Completos: row.Nombres,
      Nota_1:            row.N1,
      Nota_2:            row.N2,
      Nota_3:            row.N3,
      Nota_Final:        row.NF,
      Grupos:            row.grupo
    };
  });

  const headers = data.length
    ? Object.keys(data[0])
    : ['Nombres_Completos', 'Nota_1', 'Nota_2', 'Nota_3', 'Nota_Final', 'Grupos'];

  const csvOutput = objectsToCsv(data, headers);

  return {
    ok:        true,
    message:   '✅ Limpieza completada correctamente.',
    shape:     { rows: data.length, columns: headers.length },
    columns:   headers,
    preview:   data.slice(0, 10),
    data:      data,
    csvOutput: csvOutput
  };
}

/* =========================
 * ANÁLISIS DESCRIPTIVO
 * ========================= */

function runDescriptiveAnalysis(csvText) {
  const cleaned = runLimpieza(csvText);
  const data    = cleaned.data || [];

  if (!data.length) {
    return {
      ok:          true,
      message:     'No hay datos suficientes para generar el análisis descriptivo.',
      totalRows:   0,
      stats:       {},
      groupsCount: [],
      chartData:   {}
    };
  }

  const numericFields = ['Nota_1', 'Nota_2', 'Nota_3', 'Nota_Final'];
  const stats = {};

  numericFields.forEach(function(field) {
    const values = data
      .map(function(row)  { return toNumber(row[field]); })
      .filter(function(v) { return !isNaN(v); });
    stats[field] = calculateStats(values);
  });

  const groupsCount = countByField(data, 'Grupos');

  const chartData = {
    histNota1:    buildHistogramData(data, 'Nota_1',    'Nota_1'),
    histNota2:    buildHistogramData(data, 'Nota_2',    'Nota_2'),
    histNotaFinal:buildHistogramData(data, 'Nota_Final','Nota_Final'),
    barGrupos:    buildBarData(groupsCount, 'Grupo', 'Cantidad'),
    scatterN1Nf:  buildScatterData(data, 'Nota_1',  'Nota_Final'),
    scatterN2N3:  buildScatterData(data, 'Nota_2',  'Nota_3')
  };

  return {
    ok:          true,
    message:     'Análisis descriptivo y gráficos generados correctamente.',
    totalRows:   data.length,
    stats:       stats,
    groupsCount: groupsCount,
    chartData:   chartData,
    preview:     data.slice(0, 10)
  };
}

/* =========================
 * VER ADAPTACIÓN — K-MEANS
 * ========================= */

function runKMeansAdaptation(csvText) {
  const cleaned = runLimpieza(csvText);
  const data    = cleaned.data || [];

  if (!data.length) throw new Error('No hay datos suficientes para ejecutar K-Means.');

  const features = data.map(function(row, index) {
    return {
      index:     index,
      nombre:    row.Nombres_Completos,
      grupo:     row.Grupos,
      nota1:     toNumber(row.Nota_1),
      nota2:     toNumber(row.Nota_2),
      nota3:     toNumber(row.Nota_3),
      notaFinal: toNumber(row.Nota_Final)
    };
  }).filter(function(row) {
    return !isNaN(row.nota1) && !isNaN(row.nota2) && !isNaN(row.nota3) && !isNaN(row.notaFinal);
  });

  if (features.length < 3) throw new Error('Se requieren al menos 3 registros para aplicar K-Means.');

  const k      = 3;
  const result = kmeans(features, k, 25);

  const interpreted = result.items.map(function(item) {
    let perfil, interpretacion;

    if (item.cluster === 0) {
      perfil         = 'Rendimiento bajo';
      interpretacion = 'Estudiante con notas relativamente bajas. Se recomienda refuerzo académico y acompañamiento.';
    } else if (item.cluster === 1) {
      perfil         = 'Rendimiento medio';
      interpretacion = 'Estudiante con desempeño intermedio. Puede mejorar con seguimiento y práctica.';
    } else {
      perfil         = 'Rendimiento alto';
      interpretacion = 'Estudiante con buen desempeño. Puede participar en actividades de apoyo o liderazgo académico.';
    }

    return {
      Nombres_Completos: item.nombre,
      Grupos:            item.grupo,
      Nota_1:            item.nota1,
      Nota_2:            item.nota2,
      Nota_3:            item.nota3,
      Nota_Final:        item.notaFinal,
      Cluster:           item.cluster + 1,
      Perfil:            perfil,
      Interpretacion:    interpretacion
    };
  });

  const clusterSummary = summarizeClusters(interpreted);

  const scatterData = [['Nota_Final', 'Cluster']];
  interpreted.forEach(function(row) {
    scatterData.push([Number(row.Nota_Final), Number(row.Cluster)]);
  });

  const barData = [['Cluster', 'Cantidad']];
  clusterSummary.forEach(function(row) {
    barData.push([row.Cluster, row.Cantidad]);
  });

  const headers = [
    'Nombres_Completos', 'Grupos', 'Nota_1', 'Nota_2', 'Nota_3',
    'Nota_Final', 'Cluster', 'Perfil', 'Interpretacion'
  ];

  const csvOutput = objectsToCsv(interpreted, headers);

  return {
    ok:          true,
    message:     'Modelo K-Means ejecutado correctamente.',
    model:       'K-Means',
    k:           k,
    totalRows:   interpreted.length,
    summary:     clusterSummary,
    preview:     interpreted.slice(0, 15),
    chartData:   { scatterClusters: scatterData, barClusters: barData },
    csvOutput:   csvOutput,
    interpretation:
      'K-Means es un modelo de aprendizaje no supervisado. En este caso agrupó a los estudiantes según sus notas, sin usar una etiqueta previa.'
  };
}

/* =========================
 * GENERAR DATOS DEL INFORME
 * (llamado desde cliente antes de construir el PDF)
 * Consolida en un solo objeto todo lo necesario para
 * que jsPDF en el cliente construya el informe sin
 * volver a procesar el CSV.
 * ========================= */

/**
 * Ejecuta todos los pasos de la pipeline sobre el CSV
 * y devuelve un objeto consolidado con:
 *   - cargaDatos
 *   - limpieza
 *   - analisis
 *   - adaptacion
 *   - meta (versión, autor, fecha)
 *
 * Uso desde el cliente:
 *   google.script.run
 *     .withSuccessHandler(buildPdfFromReportData)
 *     .generateReportData(csvText);
 */
function generateReportData(csvText) {
  if (!csvText || !String(csvText).trim()) {
    throw new Error('Debe cargar o pegar un CSV antes de generar el informe.');
  }

  const now = new Date();

  /* ── Paso 1: Carga ── */
  const cargaResult = loadCsvData(csvText);

  /* ── Paso 2: Limpieza ── */
  const limpiezaResult = runLimpieza(csvText);

  /* ── Paso 3: Análisis descriptivo ── */
  const analisisResult = runDescriptiveAnalysis(csvText);

  /* ── Paso 4: K-Means ── */
  const kmeansResult = runKMeansAdaptation(csvText);

  return {
    meta: {
      appName:   'Asistente Analítico en Google Apps Script',
      version:   '3.0.0',
      author:    'Ahren Kahi Churta',
      timestamp: Utilities.formatDate(now, Session.getScriptTimeZone(), 'dd/MM/yyyy HH:mm')
    },
    cargaDatos: {
      totalRows: cargaResult.totalRows,
      columns:   cargaResult.columns
    },
    limpieza: {
      shape:   limpiezaResult.shape,
      columns: limpiezaResult.columns
    },
    analisis: {
      totalRows:   analisisResult.totalRows,
      stats:       analisisResult.stats,
      groupsCount: analisisResult.groupsCount
    },
    adaptacion: {
      totalRows:      kmeansResult.totalRows,
      k:              kmeansResult.k,
      summary:        kmeansResult.summary,
      interpretation: kmeansResult.interpretation
    }
  };
}

/* =========================
 * MODELO K-MEANS
 * ========================= */

function kmeans(data, k, maxIterations) {
  const sorted = data.slice().sort(function(a, b) { return a.notaFinal - b.notaFinal; });

  let centroids = [
    vectorFrom(sorted[0]),
    vectorFrom(sorted[Math.floor(sorted.length / 2)]),
    vectorFrom(sorted[sorted.length - 1])
  ];

  let items = [];

  for (let iter = 0; iter < maxIterations; iter++) {
    items = data.map(function(row) {
      const vector      = vectorFrom(row);
      let bestCluster   = 0;
      let bestDistance  = distance(vector, centroids[0]);

      for (let c = 1; c < k; c++) {
        const d = distance(vector, centroids[c]);
        if (d < bestDistance) { bestDistance = d; bestCluster = c; }
      }

      const cloned   = cloneObject(row);
      cloned.cluster = bestCluster;
      return cloned;
    });

    const newCentroids = [];
    for (let c = 0; c < k; c++) {
      const group = items.filter(function(item) { return item.cluster === c; });
      if (!group.length) {
        newCentroids.push(centroids[c]);
      } else {
        newCentroids.push([
          average(group.map(function(x) { return x.nota1;     })),
          average(group.map(function(x) { return x.nota2;     })),
          average(group.map(function(x) { return x.nota3;     })),
          average(group.map(function(x) { return x.notaFinal; }))
        ]);
      }
    }
    centroids = newCentroids;
  }

  return { items: reorderClustersByPerformance(items), centroids: centroids };
}

function reorderClustersByPerformance(items) {
  const means = {};

  items.forEach(function(item) {
    if (!means[item.cluster]) means[item.cluster] = [];
    means[item.cluster].push(item.notaFinal);
  });

  const order = Object.keys(means)
    .map(function(cluster) {
      return { oldCluster: Number(cluster), mean: average(means[cluster]) };
    })
    .sort(function(a, b) { return a.mean - b.mean; });

  const map = {};
  order.forEach(function(item, index) { map[item.oldCluster] = index; });

  return items.map(function(item) { item.cluster = map[item.cluster]; return item; });
}

function summarizeClusters(rows) {
  const counter = {};

  rows.forEach(function(row) {
    const key = 'Cluster ' + row.Cluster + ' - ' + row.Perfil;
    if (!counter[key]) counter[key] = { Cluster: key, Cantidad: 0, valores: [] };
    counter[key].Cantidad++;
    counter[key].valores.push(Number(row.Nota_Final));
  });

  return Object.keys(counter).map(function(key) {
    const item = counter[key];
    return {
      Cluster:              item.Cluster,
      Cantidad:             item.Cantidad,
      Promedio_Nota_Final:  round2(average(item.valores))
    };
  });
}

function vectorFrom(row) {
  return [Number(row.nota1), Number(row.nota2), Number(row.nota3), Number(row.notaFinal)];
}

function distance(a, b) {
  let sum = 0;
  for (let i = 0; i < a.length; i++) sum += Math.pow(a[i] - b[i], 2);
  return Math.sqrt(sum);
}

function average(values) {
  if (!values.length) return 0;
  return values.reduce(function(acc, v) { return acc + Number(v); }, 0) / values.length;
}

/* =========================
 * UTILIDADES PARA CHARTS
 * ========================= */

function buildHistogramData(data, field, label) {
  const rows = [[label]];
  data.forEach(function(item) {
    const value = toNumber(item[field]);
    if (!isNaN(value)) rows.push([value]);
  });
  return rows;
}

function buildBarData(items, xLabel, yLabel) {
  const rows = [[xLabel, yLabel]];
  items.forEach(function(item) { rows.push([String(item.categoria), Number(item.cantidad)]); });
  return rows;
}

function buildScatterData(data, xField, yField) {
  const rows = [[xField, yField]];
  data.forEach(function(item) {
    const x = toNumber(item[xField]);
    const y = toNumber(item[yField]);
    if (!isNaN(x) && !isNaN(y)) rows.push([x, y]);
  });
  return rows;
}

/* =========================
 * UTILIDADES INTERNAS
 * ========================= */

function parseCsvText(csvText) {
  const text = String(csvText || '').trim();
  if (!text) throw new Error('Debe proporcionar contenido CSV.');
  return Utilities.parseCsv(text);
}

function normalizeRows(rows) {
  if (!rows || !rows.length) return [];

  const headers = rows[0].map(function(h) {
    return String(h || '').replace(/^\uFEFF/, '').trim();
  });

  return rows.slice(1)
    .filter(function(row) {
      return row.some(function(cell) { return String(cell || '').trim() !== ''; });
    })
    .map(function(row) {
      const obj = {};
      headers.forEach(function(header, index) {
        obj[header] = row[index] !== undefined ? String(row[index]).trim() : '';
      });
      return obj;
    });
}

function validateRequiredColumns(data, requiredColumns) {
  if (!data.length) throw new Error('El archivo CSV no contiene registros de datos.');

  const existingColumns = Object.keys(data[0]);
  const missing = requiredColumns.filter(function(col) {
    return existingColumns.indexOf(col) === -1;
  });

  if (missing.length) throw new Error('Faltan columnas requeridas: ' + missing.join(', '));
}

function dropDuplicatesByKeys(data, keys) {
  const seen   = {};
  const result = [];

  data.forEach(function(row) {
    const compositeKey = keys.map(function(key) {
      return String(row[key] || '').trim().toLowerCase();
    }).join('||');

    if (!seen[compositeKey]) { seen[compositeKey] = true; result.push(row); }
  });

  return result;
}

function objectsToCsv(data, headers) {
  const lines = [headers.join(',')];
  data.forEach(function(row) {
    const values = headers.map(function(header) {
      const value = row[header] !== undefined && row[header] !== null ? String(row[header]) : '';
      return escapeCsvValue(value);
    });
    lines.push(values.join(','));
  });
  return lines.join('\n');
}

function csvToObjects(csvText) {
  return normalizeRows(parseCsvText(csvText));
}

function escapeCsvValue(value) {
  if (/[",\n]/.test(value)) return '"' + value.replace(/"/g, '""') + '"';
  return value;
}

function countByField(data, field) {
  const counter = {};
  data.forEach(function(row) {
    const key = String(row[field] || 'SIN_GRUPO').trim() || 'SIN_GRUPO';
    counter[key] = (counter[key] || 0) + 1;
  });
  return Object.keys(counter).sort().map(function(key) {
    return { categoria: key, cantidad: counter[key] };
  });
}

function calculateStats(values) {
  if (!values.length) {
    return { count: 0, min: null, max: null, mean: null, median: null, stdDev: null, q1: null, q3: null };
  }

  const sorted   = values.slice().sort(function(a, b) { return a - b; });
  const count    = sorted.length;
  const min      = sorted[0];
  const max      = sorted[count - 1];
  const mean     = sorted.reduce(function(acc, v) { return acc + v; }, 0) / count;
  const median   = percentile(sorted, 50);
  const q1       = percentile(sorted, 25);
  const q3       = percentile(sorted, 75);
  const variance = sorted.reduce(function(acc, v) { return acc + Math.pow(v - mean, 2); }, 0) / count;

  return {
    count:  count,
    min:    round2(min),
    max:    round2(max),
    mean:   round2(mean),
    median: round2(median),
    stdDev: round2(Math.sqrt(variance)),
    q1:     round2(q1),
    q3:     round2(q3)
  };
}

function percentile(sorted, p) {
  if (!sorted.length) return null;
  const index = (p / 100) * (sorted.length - 1);
  const lower = Math.floor(index);
  const upper = Math.ceil(index);
  if (lower === upper) return sorted[lower];
  return sorted[lower] + (sorted[upper] - sorted[lower]) * (index - lower);
}

/* =========================
 * MATEMÁTICAS / RANDOM
 * ========================= */

function randomUniform(min, max) {
  return min + Math.random() * (max - min);
}

function randomNormal(mean, stdDev) {
  let u = 0, v = 0;
  while (u === 0) u = Math.random();
  while (v === 0) v = Math.random();
  const z = Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v);
  return mean + z * stdDev;
}

function clip(value, min, max)  { return Math.max(min, Math.min(max, value)); }
function round1(value)          { return Math.round(value * 10)  / 10; }
function round2(value)          { return Math.round(value * 100) / 100; }
function toNumber(value)        { const n = Number(value); return isNaN(n) ? NaN : n; }
function cloneObject(obj)       { return JSON.parse(JSON.stringify(obj)); }
function sanitizeText(text)     { return String(text || '').replace(/\s+/g, ' ').trim(); }
