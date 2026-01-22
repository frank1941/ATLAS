# ATLAS
Archetype & Trend Learning for Assortment Strategy


flowchart TB
  %% =========================
  %% End-to-End: Arquetipos + Tendencia + Forecast (Chamarras/Abrigos)
  %% =========================

  subgraph SOURCES["Fuentes (existentes)"]
    A1["Catálogo (metadata)\n(id_articulo, temporada, precio, etc.)"]
    A2["Ventas (histórico + temporada)\n(id_articulo, semana, unidades, ingresos)"]
    A3["Features visuales extraídas\n(dev.vision_search_poc.ft_chamarras_abrigos_diccv4_features_fm_v6)\natributos + (opcional) embeddings/caption"]
  end

  subgraph JOB0["JOB 0 (opcional): Validación & Normalización de features"]
    J0["task: normalize_features\n- limpieza nulls\n- normalización valores\n- reglas diccionario v4\n- auditoría de calidad"]
    T0["tbl: dev.vision_search_poc.ft_chamarras_abrigos_features_norm_v1"]
    Q0["tbl: dev.vision_search_poc.rpt_features_quality_v1\n(completitud, nulos, outliers, etc.)"]
  end

  subgraph JOB1["JOB 1 (por temporada/mensual): Construcción de Arquetipos"]
    J1a["task: build_archetype_training_set\n- seleccionar atributos core\n- preparar vector híbrido\n(embeddings + atributos)"]
    T1a["tbl: dev.vision_search_poc.ft_archetype_training_set_v1"]
    J1b["task: cluster_archetypes\n- clustering\n- tamaño mínimo\n- estabilidad/merge"]
    T1b["tbl: dev.vision_search_poc.dim_archetype_v1\n(arquetipo_id + atributos dominantes + centro_embedding)"]
    J1c["task: archetype_cards\n- descripción textual\n- top atributos\n- ejemplos visuales (ids)"]
    T1c["tbl: dev.vision_search_poc.rpt_archetype_cards_v1"]
  end

  subgraph JOB2["JOB 2 (semanal/diario): Asignación SKU → Arquetipo"]
    J2a["task: assign_archetype\n- nearest archetype (centroid)\n- distancia\n- flags de confianza"]
    T2a["tbl: dev.vision_search_poc.ft_articulo_archetype_map_v1\n(id_articulo→arquetipo_id, distancia, confianza)"]
  end

  subgraph JOB3["JOB 3 (semanal): Índices (Novelty, Momentum, Preference Shift)"]
    J3a["task: compute_novelty\n- distancia normalizada\n- novelty buckets"]
    T3a["tbl: dev.vision_search_poc.rpt_novelty_index_articulo_v1"]
    J3b["task: compute_momentum\n- ventas tempranas (wk1-3)\n- vs baseline arquetipo histórico"]
    T3b["tbl: dev.vision_search_poc.rpt_momentum_index_articulo_semana_v1"]
    J3c["task: compute_preference_shift\n- share ventas por arquetipo\n- actual vs anterior"]
    T3c["tbl: dev.vision_search_poc.rpt_preference_shift_archetype_semana_v1"]
  end

  subgraph JOB4["JOB 4 (semanal): Trend Score + Señales para Forecast"]
    J4a["task: compute_trend_score\n- combina 3 índices\n- calibración por negocio"]
    T4a["tbl: dev.vision_search_poc.rpt_trend_score_articulo_semana_v1"]
    J4b["task: build_forecast_inputs\n- demanda por arquetipo\n- asignación a SKUs\n- features para modelo"]
    T4b["tbl: dev.vision_search_poc.ft_forecast_inputs_archetype_semana_v1"]
    T4c["tbl: dev.vision_search_poc.ft_forecast_inputs_articulo_semana_v1"]
  end

  subgraph JOB5["JOB 5 (opcional): Pronóstico + Monitoreo"]
    J5a["task: forecast_archetype\n- modelo por arquetipo\n- salida semanal"]
    T5a["tbl: dev.vision_search_poc.prd_forecast_archetype_semana_v1"]
    J5b["task: allocate_to_sku\n- reparto por trend/momentum\n- constraints negocio"]
    T5b["tbl: dev.vision_search_poc.prd_forecast_articulo_semana_v1"]
    J5c["task: monitoring\n- drift de mix por arquetipo\n- performance forecast"]
    T5c["tbl: dev.vision_search_poc.rpt_monitoring_archetype_v1"]
  end

  subgraph CONSUME["Consumo (Dashboards / Apps / Analytics)"]
    D1["Lakeview / BI\n- Cards de arquetipos\n- Tendencias por semana\n- Ranking de SKUs nuevos"]
    D2["Databricks Apps\n- search + explain\n- 'por qué este arquetipo sube'\n- alertas de tendencia"]
  end

  %% Flows
  A3 --> J0 --> T0
  T0 --> J1a --> T1a --> J1b --> T1b --> J1c --> T1c
  T0 --> J2a --> T2a
  A2 --> J3b
  T2a --> J3a --> T3a
  T2a --> J3c --> T3c
  J3b --> T3b
  T3a --> J4a
  T3b --> J4a
  T3c --> J4a --> T4a --> J4b --> T4b --> T4c
  T4b --> J5a --> T5a --> J5b --> T5b --> J5c --> T5c
  T1c --> D1
  T4a --> D1
  T5b --> D1
  T1c --> D2
  T4a --> D2
