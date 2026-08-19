# Refactorización DAX y Optimización con Variables — RetailPro

**Autor:** Axel El Hilon
**Proyecto:** Optimización de código DAX e implementación de buenas prácticas  
**Dataset:** Sample Superstore  
**Archivo del modelo:** `sample-superstore.pbix`

---

## 1. Introducción y Propósito

El objetivo de este proyecto es refactorizar un conjunto de medidas DAX heredadas en el modelo de datos de RetailPro. Aunque las medidas originales entregaban resultados numéricos correctos, presentaban ineficiencias de rendimiento y problemas de mantenibilidad debido a cálculos redundantes y falta de uso de variables (`VAR`).

Se aplicaron estándares profesionales de desarrollo DAX para garantizar que el código sea eficiente, legible y escalable.

---

## 2. Análisis de Problemas y Refactorización (_v2)

### Medida 1: `Crecimiento Anual`

* **Código Original (Ineficiente):**
  ```dax
  Crecimiento Anual = 
  DIVIDE(
      SUM(Ventas[Sales]) - CALCULATE(SUM(Ventas[Sales]), PREVIOUSYEAR(dim_fechas[Date])),
      CALCULATE(SUM(Ventas[Sales]), PREVIOUSYEAR(dim_fechas[Date]))
  )
Problema Identificado: Repite la evaluación de SUM(Ventas[Sales]) y la modificación de contexto con CALCULATE(..., PREVIOUSYEAR(...)) dos veces dentro de la misma expresión.

Solución Aplicada:

Fragmento de código
Crecimiento Anual_v2 = 
VAR VentasActuales = SUM(Ventas[Sales])
VAR VentasAnoAnterior = CALCULATE(SUM(Ventas[Sales]), PREVIOUSYEAR(dim_fechas[Date]))
RETURN
    DIVIDE(VentasActuales - VentasAnoAnterior, VentasAnoAnterior)
Impacto: Las expresiones complejas se evalúan una sola vez en memoria y se almacenan en las variables VentasActuales y VentasAnoAnterior, reduciendo la carga sobre el Formula Engine.

### Medida 2: Margen %
* **Código Original (Ineficiente):**

Fragmento de código
Margen % = 
DIVIDE(
    SUM(Ventas[Profit]),
    SUM(Ventas[Sales])
) * 100
Problema Identificado: No abstrae los componentes principales (Ganancia e Ingreso), lo que dificulta su reutilización y legibilidad en medidas compuestas.

Solución Aplicada:

Fragmento de código
Margen %_v2 = 
VAR GananciaTotal = SUM(Ventas[Profit])
VAR IngresoTotal = SUM(Ventas[Sales])
RETURN
    DIVIDE(GananciaTotal, IngresoTotal) * 100
Impacto: Mejora la semántica del código y facilita modificaciones futuras sin alterar la lógica de división.

### Medida 3: Clasificación Rendimiento
Código Original (Ineficiente):

Fragmento de código
Clasificacion Rendimiento = 
IF(
    DIVIDE(SUM(Ventas[Profit]), SUM(Ventas[Sales])) * 100 >= 20,
    "Alto",
    IF(
        DIVIDE(SUM(Ventas[Profit]), SUM(Ventas[Sales])) * 100 >= 10,
        "Medio",
        "Bajo"
    )
)
Problema Identificado: Evalúa el cálculo del porcentaje de margen tres veces a lo largo de la estructura condicional IF anidada.

Solución Aplicada:

Fragmento de código
Clasificacion Rendimiento_v2 = 
VAR MargenPct = [Margen %_v2]
RETURN
    SWITCH(
        TRUE(),
        MargenPct >= 20, "Alto",
        MargenPct >= 10, "Medio",
        "Bajo"
    )
Impacto: Elimina reevaluaciones redundantes reutilizando la variable MargenPct y reemplaza las estructuras IF anidadas por SWITCH(TRUE()).

## 3. Verificación de Resultados
Se construyó una matriz de prueba en Power BI Desktop comparando las versiones originales frente a las versiones refactorizadas _v2.

Conclusión: Los valores entregados por el par de medidas son 100% idénticos, confirmando que la refactorización conservó la lógica de negocio intacta.

## 4. Diagnóstico de Rendimiento y DAX Studio
¿Qué información proporciona Server Timing en DAX Studio?
Server Timing desglosa el tiempo de respuesta de una consulta DAX en dos componentes esenciales:

Formula Engine (FE): Procesa la lógica de negocio, iteraciones y contexto (monohilo).

Storage Engine (SE / VertiPaq): Escanea, filtra y agrega datos directamente desde la memoria (multihilo).

## ¿Por qué es fundamental medir antes de optimizar?
Permite evitar la "optimización prematura". Medir primero identifica con precisión si el cuello de botella se origina en la lógica del código DAX, en relaciones inadecuadas del modelo o en el volumen de datos, enfocando los esfuerzos donde aportan mayor impacto.
