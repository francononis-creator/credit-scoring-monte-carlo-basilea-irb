# Scoring Crediticio y Simulación de Capital Económico (Monte Carlo / Vasicek)

Trabajo final de la **Diplomatura en Ciencia de Datos con R y Python (UTN)**.

Modelo de **scoring crediticio interno** y **simulación de capital económico** para una cartera de créditos personales, comparado contra el **capital regulatorio estandarizado** y valorizado en términos económicos. La pregunta central: *¿puede un banco, con herramientas propias, medir su riesgo real mejor que lo que exige el regulador de forma simple?*

## Resumen de resultados

| Concepto | Resultado |
|---|---|
| Cartera analizada | 831 créditos (tras excluir refinanciaciones), mora observada 31,3% |
| Modelo base | Regresión logística WOE/IV de 4 variables, AUC test **0,694** · Gini 0,388 · KS 0,375 |
| Capital económico (Monte Carlo, VaR 99,9% − EL) | **16,48%** de la exposición |
| Capital regulatorio estandarizado | **6,00%** de la exposición |
| Diferencia | El capital económico es **+174,7%** mayor que el regulatorio |

En esta cartera, el enfoque estandarizado **subestimaría** el riesgo real. Esta conclusión depende de los supuestos (ver *Limitaciones*).

## Datos

- Dataset de créditos personales (`credito.csv`): 1.190 filas y 26 columnas, con variable objetivo `moroso` (22,6% de mora en el total). Se carga directamente desde un [repositorio público de datasets de la diplomatura](https://raw.githubusercontent.com/palasatenea66/DATASETS/main/credito.csv).
- **Alcance del análisis:** se excluyó el producto `DS_REFINANCIACION`, quedando **831 créditos** (mora 31,3%). Split estratificado 75/25: 623 para entrenamiento y 208 para prueba.
- **Tratamiento:** el valor centinela −1 del nivel socioeconómico (exclusivo de refinanciación) se separó de la escala real; el único nulo de consultas se imputó con la mediana.
- **Control de data leakage:** se excluyeron `dias_atraso` y `canti_moras`, variables que ya reflejan el incumplimiento, además de los identificadores.

## Metodología

1. **EDA y definición de alcance:** estructura, nulos, balance de la variable objetivo y correlaciones.
2. **Binning y WOE/IV:** agrupación monótona y por similitud de WOE; ranking por valor de información. Las de mayor IV fueron producto (0,58), provincia (0,36), ingresos (0,24) y consultas en 7 días (0,21).
3. **Selección de variables:** *backward stepwise* por AIC y significancia, con chequeo previo de multicolinealidad (VIF < 5). Variables finales: **ingresos, consultas en 7 días, producto y provincia** (todas con p < 0,001).
4. **Benchmark interpretable:** regresión logística sobre variables WOE (AIC 627,7). Incluye un ejemplo de descomposición del score por cliente (contribución de cada variable a la PD).
5. **Modelos challenger (comparativos):** Random Forest, SVM, XGBoost y red neuronal (MLP), con hiperparámetros definidos en una exploración previa.
6. **Intervalos de confianza:** bootstrap de 2.000 remuestreos para AUC y KS.
7. **PD individual** de cada crédito con el benchmark (PD promedio estimada 31,1% vs. mora observada 31,3%).
8. **Monte Carlo (modelo de un factor sistémico, Vasicek):** 100.000 escenarios, LGD 45%, correlación de activos ρ = 0,15, nivel de confianza 99,9%. La exposición se aproxima con el monto otorgado.
9. **Comparación con el capital regulatorio** (ponderador minorista 75% × 8%) y **valorización económica**: escalado a una cartera de referencia de $500 M, costo de oportunidad del capital, política de rechazo, *re-pricing* y análisis de sensibilidad.

## Resultados

### Discriminación (conjunto de prueba, n = 208)

| Modelo | AUC | Gini | KS | AUC IC 95% (bootstrap) |
|---|---|---|---|---|
| SVM | 0,720 | 0,440 | 0,366 | 0,645 – 0,791 |
| Random Forest | 0,696 | 0,393 | 0,347 | 0,617 – 0,767 |
| **Logística WOE/IV (benchmark)** | **0,694** | **0,388** | **0,375** | 0,621 – 0,765 |
| XGBoost | 0,689 | 0,377 | 0,339 | 0,610 – 0,760 |
| Red neuronal (MLP) | 0,659 | 0,318 | 0,274 | 0,580 – 0,735 |

Los intervalos se superponen: **no hay diferencia estadísticamente clara** entre los modelos. Se adopta la logística WOE/IV como benchmark por su interpretabilidad, que es lo que se exige a un modelo de uso regulatorio.

### Simulación de capital (cartera del dataset, EAD $25,5 M)

| Métrica | Valor |
|---|---|
| Pérdida esperada (EL) | $2,91 M (11,38% de la EAD) |
| VaR 99,9% | $7,11 M (27,87%) |
| **Capital económico (VaR − EL)** | **$4,21 M (16,48%)** |
| Capital regulatorio estandarizado | $1,53 M (6,00%) |

Validación del motor: la EL simulada coincide con la teórica (diferencia de 0,009 p.p.) y el error estándar del VaR es de 0,45% (IC 95%: $7,06 M – $7,17 M).

### Valorización económica (cartera escalada a $500 M)

| Concepto | Valor |
|---|---|
| Capital económico | $82,4 M |
| Capital regulatorio | $30,0 M |
| Costo de oportunidad anual (17%) | $14,0 M vs. $5,1 M (diferencia: $8,9 M) |
| Rechazar el 10% de mayor riesgo | Ahorro neto de **+$1,2 M** con costo de fondeo del 35% (cambia de signo por debajo de ~31,5%) |
| *Re-pricing* por riesgo | 69 de 831 créditos (8,3%) están sub-pricéados; ingreso adicional estimado de **$1,18 M** anuales |

**Conclusión de negocio:** rechazar al 10% de mayor riesgo apenas conviene y es sensible al costo de fondeo; el *re-pricing* por riesgo no depende de ese supuesto y mejora el ingreso sin rechazar clientes.

### Sensibilidad del capital económico (cartera de $500 M)

| Supuesto variado | Rango | Capital económico |
|---|---|---|
| LGD (35% – 60%, ρ = 0,15) | 12,9% – 22,1% de la EAD | $64,5 M – $110,6 M |
| ρ (0,03 – 0,25, LGD = 45%) | 7,3% – 21,6% de la EAD | $36,3 M – $107,8 M |

Estimación puntual: $82,4 M. Rango bajo supuestos alternativos plausibles: **$36,3 M – $110,6 M**.

## Limitaciones

- **Muestra chica** (623 para entrenar, 208 para probar): los intervalos de confianza son amplios y el AUC de entrenamiento (0,80) supera al de prueba (0,69), señal de sobreajuste moderado.
- **Poder discriminante moderado** (Gini ≈ 0,39).
- **Supuestos fijos, no estimados:** LGD (45%), ρ (0,15), costo de capital (17%), costo de fondeo (35%) y tasa de interés (65%). La sensibilidad muestra que el capital económico depende mucho de LGD y ρ.
- **Costo de capital:** el 17% solo es comparable con tasas nominales altas si se interpreta como tasa real; por eso se muestra la sensibilidad a distintos niveles.
- **EAD aproximada** con el monto otorgado.
- **La PD se asigna a toda la cartera** con un modelo entrenado sobre el 75% de ella.
- **Horizonte:** se comparan pérdida esperada y margen sobre un horizonte comparable (~1 año).
- La **cartera escalada** a $500 M es un supuesto ilustrativo.

## Cómo reproducir

```bash
git clone https://github.com/francononis-creator/credit-scoring-monte-carlo-basilea-irb.git
cd credit-scoring-monte-carlo-basilea-irb
pip install pandas numpy scipy scikit-learn statsmodels xgboost matplotlib seaborn joblib jupyter
jupyter notebook
```

Abrí el notebook y ejecutá las celdas en orden. El dataset se descarga automáticamente desde su URL pública.

## Tecnologías

Python · pandas · NumPy · SciPy · scikit-learn · statsmodels · XGBoost · matplotlib · seaborn

## Autor

**Franco Nonis** · Data Scientist · [LinkedIn](https://www.linkedin.com/in/franco-nonis-2821543a9/) · francononis14@gmail.com
