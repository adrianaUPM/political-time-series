# Entrega 3 — Minería de Datos Complejos
## Apuntes y guía de implementación: series temporales

**Base:** TEMA_3-1.pdf (20 páginas), TEMA_3-2.pdf (10 páginas), notebooks Sunspots AR/MA/ARMA y RestaurantVisitors ARIMA/SARIMAX, y enunciado de la Entrega 3.

La teoría se sintetiza a partir de estos materiales. Las recomendaciones de validación, los ejemplos políticos y las plantillas de código son adaptaciones para la práctica, no resultados obtenidos con tus datos. No se dispone de la hipótesis textual ni de los indicadores definitivos de la Entrega 2: deben recuperarse sin modificarlos arbitrariamente.

## 1. Qué exige la entrega

- Recuperar la hipótesis de la Entrega 2.
- Explorar la evolución temporal de sus indicadores.
- Aplicar al menos una técnica temporal vista en clase (por ejemplo, SARIMAX, suavizado exponencial o RNN). Los materiales aportados desarrollan la familia ARIMA/SARIMAX, por lo que será el foco de esta guía.
- Evaluar el ajuste y la precisión (MAE, RMSE cuando proceda).
- Identificar tendencias, picos y estacionalidad.
- Explicitar si la evidencia apoya o refuta la hipótesis; si es insuficiente, justificarlo.
- Incluir al menos dos gráficos interpretativos.
- Entregar `notebooks/`, `results/` y `analysis.md`.

## 2. Flujo de trabajo recomendado

```text
Hipótesis e indicadores de Entrega 2
  → construir series regulares y documentar ausencias
  → dividir cronológicamente los datos
  → explorar tendencia, estacionalidad y estacionariedad
  → ACF/PACF y elección de candidatos
  → entrenar ARIMA/SARIMA/SARIMAX
  → seleccionar con AIC/BIC y validación temporal
  → diagnosticar residuos y evaluar en test con MAE/RMSE
  → interpretar patrones y contrastar hipótesis
  → guardar gráficos, tablas y redactar analysis.md
```

## 3. Fundamentos teóricos

### 3.1 Componentes de una serie temporal

Una serie temporal es una sucesión ordenada de observaciones $X_t$. La frecuencia indica la separación entre observaciones (día, mes, trimestre…).

- Tendencia: cambio persistente del nivel a largo plazo.
- Estacionalidad: patrón repetitivo de periodo conocido y fijo.
- Ciclo: oscilación cuya duración no tiene necesariamente un periodo fijo.
- Residuo: variación no explicada por las otras componentes.
- Lag o retardo $k$: separación de $k$ observaciones.

Una descomposición aditiva puede escribirse como $X_t=T_t+S_t+R_t$. En una multiplicativa, $X_t=T_t S_t R_t$; requiere valores positivos. El notebook de restaurantes utiliza `seasonal_decompose`.

### 3.2 Estacionariedad (Tema 3-1, p. 13)

Una serie débilmente estacionaria tiene media y varianza constantes y covarianza que depende del retardo, no de la fecha absoluta. AR, MA y ARMA se emplean sobre series estacionarias. No confundir estacionariedad con estacionalidad.

Prueba Dickey-Fuller aumentada (ADF), presente en ambos notebooks:

- H0: existe una raíz unitaria.
- H1: no existe raíz unitaria (según la especificación del contraste).
- p < 0.05: se rechaza H0.
- p >= 0.05: no hay evidencia suficiente para rechazar H0; no demuestra por sí solo que la serie sea no estacionaria.

Complementar con gráficos, cambios de varianza y conocimiento del dominio. ADF no demuestra por sí solo todas las propiedades de estacionariedad.

### 3.3 Integración y diferenciación (Tema 3-2, pp. 4–6)

Cuando hay raíz unitaria, se puede diferenciar:

$$\Delta X_t=X_t-X_{t-1}$$

$$\Delta^2 X_t=\Delta X_t-\Delta X_{t-1}$$

Ejemplo: 20, 23, 27, 30 → diferencias 3, 4, 3. Si son porcentajes, las diferencias se expresan en puntos porcentuales.

`d` es el número de diferencias ordinarias. `D` es el número de diferencias estacionales: $X_t-X_{t-s}$. Escoger el mínimo necesario: diferenciar en exceso pierde observaciones y puede introducir ruido. No diferenciar manualmente para después volver a diferenciar dentro de ARIMA con el mismo `d`.

### 3.4 Autocorrelación (Tema 3-1, pp. 5–6, 16, 19)

- ACF: correlación entre $X_t$ y $X_{t-k}$; orienta el orden MA (`q`).
- PACF: correlación parcial entre esos valores eliminando la influencia lineal de los retardos intermedios; orienta AR (`p`).

| Modelo ideal | ACF | PACF |
|---|---|---|
| AR(p) | decae gradualmente | corte tras p |
| MA(q) | corte tras q | decae gradualmente |
| ARMA(p,q) | decae | decae |
| Raíz unitaria posible | decaimiento muy lento | frecuentemente primer lag alto |
| Estacional | picos en múltiplos de s | posibles picos estacionales |

Los patrones son orientativos, no una regla exacta en muestras finitas. Las bandas de confianza ayudan a localizar retardos relevantes. El notebook Sunspots propone 10 retardos como referencia general y 2m para periodicidad m (24 para anual en datos mensuales); usa 36 en su ejemplo. Reducir si hay pocos datos; PACF exige menos de la mitad de observaciones.

### 3.5 Familia de modelos (Tema 3-1, pp. 13–20; Tema 3-2, pp. 6–10)

**AR(p)**: utiliza valores anteriores.

$$X_t=c+\sum_{i=1}^{p}\alpha_i X_{t-i}+\varepsilon_t$$

Ejemplo AR(1): con c=5, alfa=0.8 y último valor 25, predicción = 5 + 0.8·25 = 25.

**MA(q)**: utiliza innovaciones o errores anteriores, no es una media móvil de las observaciones.

$$X_t=c+\sum_{i=1}^{q}\varphi_i\varepsilon_{t-i}+\varepsilon_t$$

Ejemplo MA(1): c=20, phi=0.5, último error=4 → predicción 22.

**ARMA(p,q)**: combina ambos para series estacionarias.

$$X_t=c+\sum_{i=1}^{p}\alpha_i X_{t-i}+\sum_{i=1}^{q}\varphi_i\varepsilon_{t-i}+\varepsilon_t$$

**ARIMA(p,d,q)**: aplica ARMA a la serie diferenciada d veces; p y q se orientan con ACF/PACF de la serie estacionarizada. En Python, `ARIMA(y, order=(p,d,q))` trabaja sobre niveles y gestiona la diferenciación internamente.

**ARIMAX**: incorpora regresores externos, con coeficientes beta. Ejemplos de clase: ARX, MAX, ARMAX y ARIMAX. En statsmodels, SARIMAX admite `exog` y estacionalidad desactivada.

**SARIMA(p,d,q)(P,D,Q,s)**: incorpora dinámica estacional.

| Parámetro | Significado |
|---|---|
| p | retardos AR ordinarios |
| d | diferencias ordinarias |
| q | errores MA ordinarios |
| P | orden AR estacional |
| D | diferencias estacionales |
| Q | orden MA estacional |
| s | periodo en observaciones |

Ejemplo de la diapositiva: SARIMA(1,1,1)(1,0,2,12). Tiene AR y MA ordinarios de orden 1, una diferencia ordinaria, periodo 12, AR estacional de orden 1 y MA estacional de orden 2 (retardos 12 y 24), sin diferencia estacional. La implementación multiplicativa de statsmodels puede incluir términos cruzados, que no aparecen en el esquema simplificado de las diapositivas.

**SARIMAX**: SARIMA más variables externas (X). Permite estudiar asociaciones ajustando por la dependencia temporal. Para pronosticar con exógenas se necesitan sus valores futuros o escenarios explícitos.

| Situación | Modelo candidato | Python |
|---|---|---|
| Estacionaria, dependencia de valores | AR(p) | ARIMA(y, order=(p,0,0)) |
| Estacionaria, dependencia de errores | MA(q) | ARIMA(y, order=(0,0,q)) |
| Estacionaria, ambas | ARMA | ARIMA(y, order=(p,0,q)) |
| No estacionaria | ARIMA | ARIMA(y, order=(p,d,q)) |
| Estacionalidad justificada | SARIMA | SARIMAX(y, order=(p,d,q), seasonal_order=(P,D,Q,s)) |
| Variables externas | SARIMAX | SARIMAX(y, exog=X, order=..., seasonal_order=...) |

No introducir estacionalidad solo porque los datos sean mensuales. Para estacionalidad anual mensual, s=12, si existe evidencia y suficientes ciclos. Para trimestrales, s=4; para diarios con patrón semanal, s=7.

## 4. Selección y evaluación del modelo

### 4.1 Verosimilitud, AIC y BIC (Tema 3-1, pp. 7–12)

- Log-verosimilitud (LLF): mayor es mejor en modelos comparables, pero no penaliza complejidad.
- $AIC=-2\log L+2k$: menor es mejor; equilibrio ajuste y complejidad.
- $BIC=-2\log L+k\log n$: menor es mejor; penaliza más con muestras grandes.
- k: parámetros; n: tamaño muestral; L: verosimilitud.

Comparar sobre la misma serie y muestra con verosimilitudes compatibles. No seleccionar por ajuste al pasado solamente: más parámetros pueden sobreajustar. AIC/BIC no son medidas directas del error de predicción.

### 4.2 Likelihood Ratio Test (LLR de los notebooks)

Para modelos anidados y bajo las condiciones habituales del contraste:

$$LR=2(LLF_{complejo}-LLF_{simple})$$

Se compara con una chi-cuadrado con grados de libertad iguales a la diferencia de parámetros libres. p pequeño indica evidencia a favor del modelo complejo. No aplicar a modelos no anidados ni a ajustes no convergentes.

```python
from scipy.stats import chi2

def llr(simple, complejo):
    df = len(complejo.params) - len(simple.params)
    lr = 2 * (complejo.llf - simple.llf)
    if df <= 0 or lr < 0:
        raise ValueError('Revisar orden, convergencia y anidamiento')
    return chi2.sf(lr, df)
```

### 4.3 Errores de predicción

$$MAE=\frac1n\sum |y_t-\hat y_t|$$
$$MSE=\frac1n\sum (y_t-\hat y_t)^2$$
$$RMSE=\sqrt{MSE}$$

Menor es mejor. RMSE penaliza más los errores grandes. MAE y RMSE conservan las unidades del indicador (puntos porcentuales si el CIS está en escala 0–100).

Ejemplo ilustrativo: reales [20, 22, 24], predicciones [21, 20, 24] → errores absolutos [1,2,0], MAE=1, MSE=5/3, RMSE≈1.29.

### 4.4 Diagnóstico de residuos

Se espera que las innovaciones se aproximen a ruido blanco: media cercana a cero, varianza estable y ausencia de autocorrelación. Revisar gráfico, ACF y Ljung-Box. H0 de Ljung-Box: ausencia de autocorrelación conjunta hasta el lag evaluado. p>=0.05 no prueba independencia. ADF sobre residuos, como se hace en Sunspots, no basta para demostrar ruido blanco.

## 5. Qué enseñan los notebooks del profesor

### 5.1 Sunspots — AR, MA y ARMA

- Datos: media mensual de manchas solares, enero 1749–agosto 2017. El notebook menciona el ciclo solar aproximado de 11 años.
- Preprocesamiento: convertir `Date` en índice temporal, renombrar a `value`, eliminar columna auxiliar y asignar frecuencia mensual.
- ADF, división 90/10, ACF y PACF con 36 retardos.
- ARIMA(1,0,0), búsqueda AR de orden 0 a 35, comparación gráfica AR(6), AR(12), AR(18).
- MA(1), búsqueda MA de 1 a 26, comparación con LLF/AIC/BIC y LLR.
- ARMA(18,18) frente a ARMA(1,1); predicciones y ADF sobre residuos.
- Ajuste final con todos los datos y previsión de 10 pasos.
- Búsqueda descendente p,q hasta 18 con parada temprana por empeoramiento de AIC y BIC.

El notebook comenta que a partir de 18 la mejora deja de ser significativa según BIC para AR y LLR para MA. No trasladar ese orden a otra serie. La parada temprana es heurística, no garantiza óptimo global; exigir simultáneamente menor AIC y BIC también puede descartar alternativas interesantes. La llamada LLR de la comparación ARMA final invierte el orden simple/complejo y fija grados de libertad sin calcularlos: utilizar la versión revisada anterior si se cumplen las condiciones.

### 5.2 RestaurantVisitors — SARIMA y SARIMAX

- Datos diarios de visitantes (`rest1` a `rest4`, `total`), variable externa `holiday`.
- Conversión de fechas, frecuencia diaria, ADF, descomposición, ACF (24) y PACF (25).
- División cronológica 95/5 y búsqueda AR/MA.
- SARIMA(0,0,0)(1,0,1,7): periodicidad semanal.
- SARIMAX con el mismo orden y `exog=holiday`: permite incorporar festivos.
- Comparación adicional SARIMAX(0,0,3)(1,0,1,7), predicciones de 24 pasos y gráficos reales frente a previstos.

Trasladar esta lógica, no sus parámetros: en política una variable externa podría ser un evento electoral conocido o un indicador de discurso. Usar `steps=len(test)` en lugar de fijar 24. Los notebooks hacen parte del diagnóstico antes de dividir y usan test al comparar modelos; para una evaluación final independiente conviene separar validación y test. No suprimir avisos de convergencia de forma indiscriminada.

## 6. Adaptación a indicadores políticos y CIS (propuesta)

Si la práctica trata de discurso parlamentario, diversidad léxica, sentimiento o polarización y apoyo electoral, utilizar los indicadores realmente calculados en Entrega 2. Ejemplo de hipótesis ilustrativa, NO sustituto de la original: «Existe una asociación temporal entre el indicador lingüístico y la estimación de voto del partido».

| Fecha mensual | Partido | Indicador lingüístico | Estimación CIS | Número de palabras |
|---|---|---:|---:|---:|
| 2020-09 | PSOE | NA | 31.5 | NA |
| 2020-09 | PP | NA | 18.1 | NA |
| 2020-09 | Unidas Podemos | NA | 12.2 | NA |

Los valores CIS proceden de la tabla mostrada anteriormente. NA significa dato no proporcionado, no cero. No mezclar voto directo sobre censo con estimación sobre voto válido. Para tendencias de apoyo electoral, escoger una definición homogénea. No tratar encuestas como resultados electorales reales.

Decisiones recomendadas:

- Una serie por partido e indicador, frecuencia mensual si hay datos suficientes. Si hay pocas observaciones, valorar trimestre, sin inventar datos.
- Unificar denominaciones y documentar coaliciones, escisiones y cambios metodológicos del CIS.
- Conservar cobertura de discursos y encuestas; ausencia de actividad no implica valor cero de un indicador lingüístico.
- TTR depende de longitud textual: comparar con tamaños similares o utilizar el indicador robusto ya definido en Entrega 2. No cambiar la métrica sin explicarlo.
- Alinear mes de publicación o recogida de encuesta según objetivo; evitar información publicada después de la fecha de predicción.
- Si se estudia asociación con voto, usar voto como y e indicador lingüístico retardado como X; si la hipótesis solo aborda evolución del discurso, modelar el indicador directamente.
- Los porcentajes de partidos no son independientes (composición del voto); modelos por partido no garantizan suma 100 ni valores en [0,100]. Documentar esta limitación.

## 7. Plantilla de notebook en Python

Los siguientes bloques son una adaptación metodológica ejecutable tras sustituir el CSV, columnas y parámetros por los de Entrega 2. Se asume un CSV mensual, una fila por partido y mes, con valores numéricos en escala documentada. No se han ejecutado con tus datos reales.

### 7.1 Carga y preparación

```python
from pathlib import Path
import itertools
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from statsmodels.tsa.stattools import adfuller
from statsmodels.graphics.tsaplots import plot_acf, plot_pacf
from statsmodels.tsa.seasonal import seasonal_decompose
from statsmodels.tsa.statespace.sarimax import SARIMAX
from statsmodels.stats.diagnostic import acorr_ljungbox
from sklearn.metrics import mean_absolute_error, mean_squared_error

root = Path.cwd()
if root.name == 'notebooks':
    root = root.parent
out = root / 'results'
out.mkdir(exist_ok=True)

df = pd.read_csv(root / 'data' / 'indicadores.csv')
df['fecha'] = pd.to_datetime(df['fecha']).dt.to_period('M').dt.to_timestamp()
partido = 'PSOE'  # adaptar
col = 'estimacion_cis'  # o indicador de Entrega 2
serie = df.loc[df['partido'].eq(partido)].copy()
assert not serie['fecha'].duplicated().any(), 'Resolver duplicados justificadamente'
y = pd.to_numeric(serie.set_index('fecha')[col], errors='raise').sort_index().asfreq('MS')
print(y.isna().sum(), 'meses ausentes')
assert y.notna().all(), 'Resolver ausencias sin inventar observaciones'
assert len(y) >= 40, 'Plantilla pensada para una serie suficientemente larga; adaptar si es corta'
```

No utilizar `dropna()` indiscriminadamente después de `asfreq`: puede eliminar meses y alterar la separación entre observaciones. En presencia de huecos, estudiar cobertura, usar un modelo de espacio de estados que admita y ausente o una agregación justificada; no interpolar todo el conjunto antes de dividir. Para regresores externos deben resolverse valores ausentes antes del ajuste.

### 7.2 División temporal y exploración

```python
n = len(y)
a, b = int(n*.70), int(n*.85)
train, val, test = y.iloc[:a], y.iloc[a:b], y.iloc[b:]
print(train.index.min(), train.index.max(), val.index.max(), test.index.max())

ax = train.plot(figsize=(11,4), label='Observado')
train.rolling(6, min_periods=6).mean().plot(ax=ax, label='Media móvil 6 meses')
ax.set(title=f'Evolución de {col}: {partido}', ylabel=col)
ax.legend()
plt.tight_layout(); plt.savefig(out/'01_evolucion.png', dpi=160); plt.show()

# Exploración estacional solo si existe periodicidad plausible y al menos dos ciclos.
if len(train) >= 24:
    dec = seasonal_decompose(train, model='additive', period=12)
    fig = dec.plot(); fig.tight_layout()
    fig.savefig(out/'02_descomposicion.png', dpi=160); plt.show()
```

La media móvil descriptiva no es el modelo MA(q). La ventana de 6 meses es ilustrativa. La descomposición presupone periodo 12, no lo demuestra. Los datos de test no se usan para elegir parámetros.

### 7.3 ADF y ACF/PACF

```python
def diagnostico_adf(s):
    estadistico, p, lags, nobs, criticos, *_ = adfuller(s.dropna(), autolag='AIC')
    return {'ADF': estadistico, 'p': p, 'lags': lags, 'n': nobs}

print('Niveles:', diagnostico_adf(train))
print('Primera diferencia:', diagnostico_adf(train.diff()))
d = 0 if diagnostico_adf(train)['p'] < .05 else 1  # candidato; revisar gráficos
z = train if d == 0 else train.diff().dropna()
lag = min(24, len(z)//2 - 1)
fig, axes = plt.subplots(2, 1, figsize=(10,7))
plot_acf(z, lags=lag, zero=False, ax=axes[0])
plot_pacf(z, lags=lag, zero=False, method='ols', ax=axes[1])
fig.tight_layout(); fig.savefig(out/'03_acf_pacf.png', dpi=160); plt.show()
```

Si primera diferencia sigue sin ser estacionaria, investigar tendencia determinista, rupturas, estacionalidad y posible d=2. No automatizar la decisión solo con p-valor. Para candidatos estacionales, examinar también la diferencia estacional y ACF en múltiplos de s.

### 7.4 Comparar modelos y un baseline

Usamos SARIMAX sin componente estacional para representar ARIMA y con componente estacional para SARIMA. La rejilla pequeña evita órdenes excesivos. Ajustar su alcance al diagnóstico anterior. El ejemplo mantiene el mismo d y tendencia para los candidatos; los criterios de información con diferenciaciones distintas requieren especial cuidado.

```python
def metricas(real, pred):
    assert real.index.equals(pred.index), 'Fechas desalineadas'
    return {'MAE': mean_absolute_error(real, pred),
            'RMSE': np.sqrt(mean_squared_error(real, pred))}

estacionales = [(0,0,0,0)]
# Activar únicamente si el análisis justifica estacionalidad anual y hay ciclos suficientes.
usar_estacional = False
if usar_estacional and len(train) >= 36:
    estacionales += [(1,0,0,12), (0,0,1,12), (1,0,1,12)]

filas, fallos = [], []
for p, q, est in itertools.product(range(3), range(3), estacionales):
    orden = (p,d,q)
    try:
        fit = SARIMAX(train, order=orden, seasonal_order=est,
                      trend='c' if d == 0 else 'n').fit(disp=False)
        if not fit.mle_retvals.get('converged', True):
            raise RuntimeError('No converge')
        pred = fit.forecast(steps=len(val))
        filas.append({'order': orden, 'seasonal_order': est,
                      'AIC': fit.aic, 'BIC': fit.bic, 'LLF': fit.llf,
                      **metricas(val, pred)})
    except Exception as exc:
        fallos.append({'order': orden, 'seasonal_order': est, 'error': str(exc)})

ranking = pd.DataFrame(filas).sort_values(['RMSE','BIC']) if filas else pd.DataFrame()
pd.DataFrame(fallos).to_csv(out/'fallos_modelos.csv', index=False)
assert not ranking.empty, 'Ningún candidato válido'
ranking.to_csv(out/'comparativa_validacion.csv', index=False)
print(ranking.head())
baseline_val = pd.Series(train.iloc[-1], index=val.index)
print('Baseline validación:', metricas(val, baseline_val))
```

Seleccionar por RMSE en validación, teniendo en cuenta BIC, diagnóstico y parsimonia. Si el modelo no supera al baseline, reconocerlo; si los resultados son similares, preferir el más simple. Para una serie corta, considerar validación de origen rodante en lugar de una rejilla amplia. Si hay estacionalidad, comparar también con baseline estacional (valor del mismo mes del año anterior, sin utilizar valores futuros).

### 7.5 Evaluación final y diagnóstico

```python
mejor = ranking.iloc[0]
trainval = pd.concat([train, val])
fit = SARIMAX(trainval, order=mejor['order'],
              seasonal_order=mejor['seasonal_order'],
              trend='c' if d == 0 else 'n').fit(disp=False)
assert fit.mle_retvals.get('converged', True), 'Revisar convergencia'
f = fit.get_forecast(steps=len(test))
pred = f.predicted_mean
ci = f.conf_int(alpha=.05)
base = pd.Series(trainval.iloc[-1], index=test.index)
res = pd.DataFrame([{'modelo':'SARIMAX', **metricas(test,pred)},
                    {'modelo':'Naive', **metricas(test,base)}])
res.to_csv(out/'metricas_test.csv', index=False)
pd.DataFrame({'real':test,'prediccion':pred,'inferior':ci.iloc[:,0],
              'superior':ci.iloc[:,1]}).to_csv(out/'predicciones_test.csv')
print(res)

ax = trainval.tail(24).plot(figsize=(11,4), label='Entrenamiento + validación')
test.plot(ax=ax, label='Real test'); pred.plot(ax=ax, label='Predicción')
ax.fill_between(test.index, ci.iloc[:,0].to_numpy(), ci.iloc[:,1].to_numpy(),
                alpha=.2, label='Intervalo predictivo 95%')
ax.legend(); ax.set_title('Predicción frente a valores reales')
plt.tight_layout(); plt.savefig(out/'04_prediccion_test.png', dpi=160); plt.show()

resid = pd.Series(fit.filter_results.standardized_forecasts_error[0],
                  index=trainval.index).iloc[fit.loglikelihood_burn:].dropna()
p, _, q = mejor['order']; P, _, Q, _ = mejor['seasonal_order']
lag = min(12, len(resid)//5)
if lag > p+q+P+Q:
    print(acorr_ljungbox(resid, lags=[lag], model_df=p+q+P+Q, return_df=True))
fig = fit.plot_diagnostics(figsize=(10,7))
fig.tight_layout(); fig.savefig(out/'05_residuos.png', dpi=160); plt.show()
(out/'modelo_summary.txt').write_text(str(fit.summary()), encoding='utf-8')
```

El intervalo es condicional al modelo y sus supuestos; no equivale al intervalo muestral del CIS. El horizonte es una predicción multipaso desde el final de train+validación. Si se quiere evaluar predicción de un solo mes, aplicar origen rodante de manera consistente también al baseline.

## 8. SARIMAX con variable exógena: contrastar una asociación

Adaptación del ejemplo `holiday` del notebook de restaurantes. Supongamos y=estimación CIS y X=indicador lingüístico del mes anterior (hipótesis ilustrativa).

```python
# serie contiene el partido seleccionado y las fechas ya normalizadas.
X = (serie.set_index('fecha')[['indicador_linguistico']]
     .sort_index().asfreq('MS').shift(1).rename(
        columns={'indicador_linguistico':'indicador_lag1'}))
alineado = pd.concat([y.rename('y'), X], axis=1).iloc[1:]
assert alineado.notna().all().all(), 'Revisar cobertura de X e y'
# Mantener las fechas de corte del protocolo anterior.
tr = alineado.loc[:train.index[-1]]
va = alineado.loc[val.index]
te = alineado.loc[test.index]
# Orden de ejemplo: seleccionar en validación, no en test.
orden, est = (1,d,0), (0,0,0,0)
fit_x = SARIMAX(tr['y'], exog=tr[['indicador_lag1']], order=orden,
                seasonal_order=est, trend='c' if d == 0 else 'n').fit(disp=False)
fit_0 = SARIMAX(tr['y'], order=orden, seasonal_order=est,
                trend='c' if d == 0 else 'n').fit(disp=False)
for nombre, modelo in [('sin X',fit_0),('con X',fit_x)]:
    if not modelo.mle_retvals.get('converged', True):
        raise RuntimeError(f'{nombre} no converge')
print(metricas(va.y, fit_0.forecast(len(va))))
print(metricas(va.y, fit_x.forecast(len(va), exog=va[['indicador_lag1']])))
print(fit_x.params, fit_x.pvalues, fit_x.conf_int())
```

**Interpretación y cautelas:**

- Beta positivo: asociación positiva en la especificación ajustada; negativo: asociación inversa. Revisar cómo la integración y las transformaciones afectan la interpretación.
- Comparar sin X y con X sobre exactamente las mismas fechas. Para aislar el aporte de X usar el mismo orden; también puede compararse el mejor candidato de cada familia mediante validación independiente.
- La predicción multipaso anterior con X observada es una evaluación condicional: los valores de X de meses posteriores al origen pueden no conocerse en ese momento. Para pronóstico real usar solo X disponible en el origen, escenarios de X o evaluación rodante a un paso, respetando fechas de publicación. Un lag de un mes no elimina automáticamente toda fuga temporal.
- Tras fijar la especificación, reajustar sobre train+validación y evaluar una sola vez en test bajo el protocolo elegido. Para análisis de asociación, reportar beta, intervalo y sensibilidad a retardos/controles.
- Tendencias comunes pueden producir correlaciones espurias; revisar estacionariedad, cambios de régimen y especificación. Una asociación temporal no demuestra causalidad. No elegir retardos según el p-valor mínimo sin reconocer comparaciones múltiples.

## 9. Técnicas opcionales

Estas extensiones son propuestas adicionales para cubrir el enunciado, no desarrollos centrales de los dos PDFs.

### 9.1 Picos y anomalías

Usar errores de predicción de un paso o residuos estandarizados. Por ejemplo, marcar |z|>3 como candidato a anomalía (umbral heurístico). Verificar cobertura y contexto: elecciones, crisis, cambios de metodología. Un pico no implica error de datos ni causalidad.

```python
candidatos = resid[resid.abs() > 3].rename('residuo_estandarizado')
candidatos.to_csv(out/'anomalias_candidatas.csv')
```

### 9.2 Comparación de periodos

Definir cortes motivados por la hipótesis antes de observar resultados. Comparar media, mediana, dispersión, tendencia y cobertura. Una prueba t convencional puede ser inadecuada por autocorrelación; considerar regresión con errores temporales o intervalos mediante bootstrap por bloques. Si los cambios coinciden con un evento, hablar de coincidencia temporal, no de causalidad automáticamente.

### 9.3 Predicción futura

Una vez cerrada la evaluación, reajustar el modelo elegido sobre todos los datos, pronosticar un horizonte justificado (p. ej. tres meses), guardar intervalos y explicar incertidumbre. Con X, suministrar escenarios futuros; no inventarlos como observaciones reales.

## 10. Cómo interpretar y contrastar la hipótesis

| Evidencia | Qué permite afirmar | Qué no permite afirmar |
|---|---|---|
| Tendencia creciente del indicador | Aumento descriptivo en el periodo | Causa del aumento |
| ACF alta | Dependencia temporal | Relación causal entre variables |
| Menor AIC/BIC | Mejor equilibrio ajuste-complejidad entre candidatos | Menor error futuro garantizado |
| MAE/RMSE menor que naive | Mejora predictiva en la muestra evaluada | Hipótesis sustantiva confirmada |
| Beta con signo esperado e intervalo que excluye cero | Evidencia de asociación bajo el modelo | Causalidad demostrada |
| Beta incierto o resultados inestables | Evidencia insuficiente | Ausencia demostrada de relación |

Ejemplo de redacción con huecos: «La hipótesis de Entrega 2 sostiene [texto exacto]. El indicador [nombre] presenta [patrón] entre [fechas]. El modelo [orden] obtuvo MAE=[valor] y RMSE=[valor] frente al baseline [valores]. El análisis de [coeficiente/tendencia/comparación] muestra [resultado e incertidumbre]. Por tanto, la evidencia [apoya/refuta/no permite concluir sobre] la hipótesis, dentro de las limitaciones de [cobertura, dependencia, factores omitidos]».

No inventar resultados. El desempeño predictivo y el contraste sustantivo son dos cuestiones diferentes. Si la hipótesis predice aumento de polarización, evaluar la dirección del indicador; si predice asociación con voto, evaluar dicha asociación controlando la estructura temporal.

## 11. Estructura recomendada de entrega

```text
entrega3/
├── data/                         # opcional: datos o instrucciones de acceso
├── notebooks/
│   ├── 01_exploracion.ipynb
│   ├── 02_modelado_temporal.ipynb
│   └── 03_evaluacion_hipotesis.ipynb
├── results/
│   ├── 01_evolucion.png
│   ├── 02_descomposicion.png
│   ├── 03_acf_pacf.png
│   ├── 04_prediccion_test.png
│   ├── 05_residuos.png
│   ├── comparativa_validacion.csv
│   ├── metricas_test.csv
│   ├── predicciones_test.csv
│   └── modelo_summary.txt
└── analysis.md
```

Se pueden unir los notebooks si resulta más sencillo. Registrar versión de paquetes (`pip freeze` o requirements), fuentes y transformaciones. Ejecutar de principio a fin antes de entregar.

## 12. Plantilla de analysis.md

```markdown
# Entrega 3 — Análisis temporal
## 1. Hipótesis de Entrega 2
[Copiar literalmente y definir qué evidencia se espera observar.]
## 2. Datos e indicadores
[Fuentes, fechas, frecuencia, unidades, cobertura, ausencias, partidos.]
## 3. Técnicas aplicadas
[ADF, ACF/PACF, diferenciación, candidatos y justificación de órdenes.]
[División cronológica, baseline, AIC/BIC, métricas y diagnóstico.]
## 4. Resultados y visualizaciones
![Evolución](results/01_evolucion.png)
[Interpretar tendencia, picos y posibles cambios.]
![Predicción](results/04_prediccion_test.png)
[Interpretar errores, intervalos y diferencias frente al baseline.]
[Tabla real de métricas y, si corresponde, coeficientes e intervalos.]
## 5. Discusión de la hipótesis
[Apoyada, refutada o evidencia insuficiente; explicar por qué.]
## 6. Limitaciones y posibles mejoras
[Datos faltantes, longitud, cambios estructurales, variables omitidas,
incertidumbre de encuestas, ausencia de inferencia causal, etc.]
```

## 13. Checklist final

- [ ] Hipótesis textual recuperada y criterio de contraste definido.
- [ ] Indicadores de Entrega 2 conservados y unidades documentadas.
- [ ] Serie ordenada, frecuencia regular, duplicados y ausencias revisados.
- [ ] Gráfico temporal interpretado.
- [ ] Estacionariedad y ACF/PACF justificadas.
- [ ] ARIMA/SARIMAX ajustado con parámetros razonados.
- [ ] Separación cronológica y comparación contra baseline.
- [ ] AIC/BIC, MAE/RMSE y diagnóstico de residuos documentados.
- [ ] Al menos dos gráficos interpretados en analysis.md.
- [ ] Conclusión explícita sobre hipótesis, sin confundir asociación y causalidad.
- [ ] Limitaciones y mejoras, notebooks ejecutables y resultados guardados.

## 14. Referencias de los materiales

- `TEMA_3-1.pdf`: modelado y autocorrelación (pp. 5–6), selección y criterios (pp. 7–12), estacionariedad y AR/MA/ARMA (pp. 13–20).
- `TEMA_3-2.pdf`: integración (pp. 4–5), ARIMA (p. 6), exógenas (p. 7), estacionalidad y SARIMA (pp. 8–9), SARIMAX (p. 10). Se usan páginas físicas del PDF, no numeración impresa.
- `N2_Series_Temporales_V2_Sunspots_AR_MA_ARMA.ipynb`: preprocesamiento, ADF, ACF/PACF, ajuste y selección AR/MA/ARMA, LLR y previsión.
- `N2_Series_Temporales_V2_RestaurantVisitors_ARIMA_SARIMAX.ipynb`: preprocesamiento, descomposición, selección, estacionalidad semanal y regresor holiday.
- Enunciado de Entrega 3 proporcionado en imagen.

Bibliografía recogida en Tema 3-1: Brockwell y Davis (2016), Box et al. (2015), Nielsen (2019).
