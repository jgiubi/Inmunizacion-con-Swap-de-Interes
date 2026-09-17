[README.md](https://github.com/user-attachments/files/32358559/README.md)
# Módulo 3 — Inmunización de una Cartera de Renta Fija con Swaps de Tasa

## Descripción

Proyecto de gestión de activos y pasivos (ALM) que diseña una cartera de bonos soberanos argentinos combinada con un Interest Rate Swap (IRS) para inmunizar un pasivo simulado frente a movimientos de la tasa de interés. La inmunización se logra igualando la duración modificada del portafolio de activos (bonos + swap) con la duración del pasivo.

**Contexto:** proyecto de portafolio profesional que conecta los conceptos de renta fija del Módulo 2 (curva de tasas Nelson-Siegel) con derivados de tasa, demostrando técnicas de uso habitual en tesorería de bancos e instituciones aseguradoras.

---

## Relevancia profesional

- Manejo práctico de ALM con derivados, habilidad central en mesas de tesorería y áreas de riesgo financiero
- Vincula la curva de tasas calibrada en el módulo anterior con una aplicación concreta de cobertura
- Comprensión del pricing de swaps plain vanilla mediante DCF
- Evidencia uso de datos de mercado reales (SOFR via Derivative Logic) en lugar de tasas sintéticas

---

## Herramientas

| Herramienta | Uso |
| Python | Valuación del swap, cálculo de duraciones, stress test |
| Excel | Dashboard de inmunización, Solver, gráfico del stress test |

**Librerías Python:** numpy, pandas

---

## Datos utilizados

| Dataset | Fuente | Uso |
| Curva spot de bonos argentinos | Proyecto Módulo 2 (Nelson-Siegel) | Duración de los bonos de la cartera |
| Tasas SOFR a 1, 2 y 3 años (16/09/2026) | Derivative Logic | Curva de descuento del swap |
| Bonos AL29 y AE38 | Mercado secundario ARG | Activos de la cartera |

---

## Metodología

### Fase 1 — Pasivo simulado

Se define un pasivo zero-coupon:
- **Valor futuro:** USD 1.200.000
- **Vencimiento:** 3 años
- **Tasa de descuento:** 7% anual

El pasivo zero-coupon tiene Duración Macaulay = T, lo que simplifica el análisis y aísla el efecto del swap.

### Fase 2 — Cartera de bonos

Se seleccionan dos bonos soberanos del Módulo 2 cuyas duraciones modificadas "rodean" la duración del pasivo:

| Bono | D. Modificada | Peso |
| AL29 | 1.456 | 40% |
| AE38 | 4.330 | 60% |
| **Cartera** | 3.181 |
| **Pasivo** | 2.804 |

Gap inicial: +0.377 → la cartera es más larga que el pasivo → se requiere un **payer swap** (pagar fijo, recibir flotante) para acortar duración.

> Se usa Duración Modificada en todos los pasos, dado que la métrica de inmunización opera sobre sensibilidad al precio (ΔPV ≈ −D_mod · PV · Δr). La Duración Macaulay se usa únicamente como paso intermedio para obtener la Modificada.

### Fase 3 — Valuación del swap

Swap plain vanilla valuado como diferencia entre un bono fijo y un bono flotante:

V_swap = B_fijo − B_flotante

La **par rate** se calcula como la tasa fija que hace V_swap = 0 al inicio del contrato:

par_rate = (1 − FD_n) / sum (FD_i)

| Parámetro | Valor |
| Curva spot (1Y / 2Y / 3Y) | 4.273% / 4.392% / 4.410% |
| Par rate (tasa fija) | 4.407% |
| D. Modificada del swap | 2.754 |

**Supuesto:** `B_flotante = nocional` es válido únicamente al inicio del swap. En una fecha intermedia, el bono flotante no vale el nocional exacto.

### Fase 4 — Determinación del nocional

El nocional del swap se despeja de la ecuación de inmunización de primer orden:

D_cartera · VP_A + D_swap · N_swap = D_pasivo · VP_L

N_swap = (D_pasivo · VP_L − D_cartera · VP_A) / D_swap

| Resultado | Valor |
| Nocional del swap | USD −134.100 |
| D. cartera post-swap | 2.8037 |
| D. pasivo | 2.8037 |
| Gap residual | 0.000000 |

El nocional negativo refleja la posición **payer** (se paga tasa fija), consistente con la necesidad de acortar duración.

### Fase 5 — Stress test

Shock paralelo de tasas en rango −300bps a +300bps. El cambio en el valor del pasivo se calcula por **repricing completo** con la curva shockeada; los activos se aproximan linealmente vía duración modificada.

**Limitación declarada:** se asume un shock paralelo uniforme sobre ambas curvas (SOFR y tasa de descuento del pasivo), lo cual es un supuesto fuerte. En la práctica, ambas curvas pueden moverse de forma no paralela.

**Resultado esperado:** `error_sin` crece proporcionalmente al shock; `error_con` captura únicamente el error de convexidad residual (efecto de segundo orden), que para inmunización de primer orden es estructuralmente positivo para el activo.

---

## Limitaciones del modelo

1. **Inmunización de primer orden:** la igualación de duraciones anula el riesgo de primer orden (duración), pero no el de segundo (convexidad). Un portfolio convexo tiene ventaja asimétrica ante shocks grandes.
2. **Supuesto de curva plana:** los shocks se aplican en paralelo. En la práctica los distintos tramos de la curva se mueven de forma diferente (riesgo de pendiente).
3. **Riesgo de base:** la curva del pasivo (7% en pesos) y la curva del swap (SOFR) son mercados distintos. Su correlación imperfecta genera riesgo residual.
4. **B_flotante = nocional al inicio:** válido únicamente en t=0.
5. **Pesos de cartera fijos:** en la práctica los pesos se rebalancean periódicamente a medida que las duraciones cambian con el tiempo.
