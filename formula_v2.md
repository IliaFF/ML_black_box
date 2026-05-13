<!-- Description: итоговая формула модели V2 (44 params, вырожденные члены исключены) с LaTeX-уравнениями и значениями параметров.
     Related: [[formula.md]], [[scripts/tmp_cv_engineered.py]], [[results/model_u_v2_final_params.npy]] -->

# Итоговая формула (Модель V2)

**Параметров:** 44 (3 вырождены → используется 41 эффективно)  
**Область применения:** все признаки $|x_i| < 9$ (active zone)  
**Файл параметров:** `results/model_u_v2_final_params.npy`

Модель V2 расширяет K4 двумя реальными улучшениями:
1. **$g_\theta(\theta \geq 0) \neq 0$** — добавлен гауссиан для положительных $\theta$ (был ≡0)
2. **Два гауссиана** в $C_{\text{neg}}$ (был ноль)
3. **Три гауссиана** в $C_{\text{pos}}$ (был один)

---

## Полная формула

$$
\boxed{y = \Bigl(F(\mu)\sin(k_1\theta) + H(\mu) + g_\theta(\theta)\cos\!\bigl(k_2\mu + \tfrac{tc^2}{10}\bigr) + \mu\cdot tc\cdot C(\theta)\Bigr)\cdot G_{\text{main}}}
$$

---

## Константы

$$
k_1 = 2.500, \qquad k_2 = 1.800
$$

---

## Гейт-функция

$$
G_{\text{clip}}(x) = \operatorname{clip}(10 - |x|,\; 0,\; 1)
$$

$$
G_{\text{main}} = G_{\text{clip}}(\mu)\cdot G_{\text{clip}}(\theta)\cdot G_{\text{clip}}(db)\cdot\prod_{i=1}^{9} G_{\text{clip}}(g_i)
$$

где $g_i$ — девять нефизических gate-признаков (`scint_alpha`, `pixel_jitter`, `cathode_temp`, `anode_noise`, `beam_phase`, `shield_rho`, `veto_count`, `gain_offset`, `trigger_rate`).

> В active zone все $|x_i| < 9$ → $G_{\text{main}} = 1$.

---

## Компонента $F(\mu)$ — чётный резонанс Брейта–Вигнера

$$
F(\mu) = \frac{A_{bw}\,\mu^2}{\bigl(|\mu| - a_{bw}\bigr)^2 + g_W^2}
$$

| Параметр | Значение |
|----------|----------|
| $A_{bw}$ | 100.653  |
| $a_{bw}$ | 9.664    |
| $g_W$    | 1.444    |

---

## Компонента $H(\mu)$ — нечётный двойной резонанс

$$
H(\mu) = A_H\left(\frac{1}{(\mu - m_H)^2 + g_H^2} - \frac{1}{(\mu + m_H)^2 + g_H^2}\right)
$$

| Параметр | Значение |
|----------|----------|
| $A_H$    | 269.798  |
| $m_H$    | 9.290    |
| $g_H$    | 1.286    |

---

## Компонента $g_\theta(\theta)$ — θ-модулятор фазы

В V2 **ненулевая с обеих сторон**:

$$
g_\theta(\theta) =
\begin{cases}
\dfrac{A_T\,\theta\,(\theta - t_z)}{(\theta - t_H)^2 + g_T^2} + B_T\,\theta\,e^{-e_T(\theta - c_T)^2} & \theta < 0 \\[12pt]
B_T^+\,\theta\,e^{-e_T^+(\theta - c_T^+)^2} & \theta \geq 0
\end{cases}
$$

**Параметры $g_\theta^-$ ($\theta < 0$):**

| Параметр | Значение  |
|----------|-----------|
| $A_T$    | 350.534   |
| $t_z$    | −4.042    |
| $t_H$    | −11.032   |
| $g_T$    | 1.822     |
| $B_T$    | −10.554   |
| $e_T$    | 0.0615    |
| $c_T$    | −1.775    |

**Параметры $g_\theta^+$ ($\theta \geq 0$):**

| Параметр  | Значение |
|-----------|----------|
| $B_T^+$   | 27.617   |
| $e_T^+$   | 1.401    |
| $c_T^+$   | −0.434   |

---

## Компонента $C(\theta)$ — коэффициент при $\mu \cdot tc$

$$
C(\theta) =
\begin{cases}
C_{0n} + A_n\,\sinh\!\left(\dfrac{k_2\,|\theta|}{n_n}\right) + D_n\,\theta\,e^{-e_n(\theta-c_n)^2} + D_{n2}\,\theta\,e^{-e_{n2}(\theta-c_{n2})^2} & \theta < 0 \\[12pt]
C_{0p} + A_c\,\sinh\!\left(\dfrac{k_2\,\theta}{n_c}\right) + B_c\,\theta\,e^{-c_c\,\theta^2} + D_p\,\theta\,e^{-e_p(\theta-c_p)^2} + D_{p2}\,\theta\,e^{-e_{p2}(\theta-c_{p2})^2} & \theta \geq 0
\end{cases}
$$

**$C_{\text{neg}}$ ($\theta < 0$):**

| Параметр  | Значение  |
|-----------|-----------|
| $C_{0n}$  | 0.3015    |
| $A_n$     | 0.4794    |
| $n_n$     | 3.986     |
| $D_n$     | −0.0719   |
| $e_n$     | 1.557     |
| $c_n$     | −6.096    |
| $D_{n2}$  | 0.3527    |
| $e_{n2}$  | 0.1564    |
| $c_{n2}$  | −0.257    |

**$C_{\text{pos}}$ ($\theta \geq 0$):**

| Параметр  | Значение  |
|-----------|-----------|
| $C_{0p}$  | 0.2155    |
| $A_c$     | 0.3770    |
| $n_c$     | 3.731     |
| $B_c$     | 0.8663    |
| $c_c$     | 0.2685    |
| $D_p$     | −0.6828   |
| $e_p$     | 1.719     |
| $c_p$     | 0.530     |
| $D_{p2}$  | −0.0389   |
| $e_{p2}$  | 3.521     |
| $c_{p2}$  | 5.738     |

---

## Отличия от K4

| Компонента | K4 | V2 |
|------------|----|----|
| $g_\theta(\theta \geq 0)$ | ≡ 0 | Гауссиан ($B_T^+, e_T^+, c_T^+$) |
| $C_{\text{neg}}$ Гауссианы | 0 | 2 (+6 params) |
| $C_{\text{pos}}$ Гауссианы | 1 | 3 (+6 params) |
| Структура $F, H, k_1, k_2$ | одинакова | одинакова |

Три параметра V2 вырождены и исключены из документации:
- $\alpha_F \approx 5\times10^{-5}$ → множитель $(1+\alpha_F tc^2) \approx 1$
- $\beta_{tc} \approx 10$ → аргумент косинуса $tc^2/\beta_{tc} = tc^2/10$ как в K4
- $A_T^+ \approx -2\times10^{-5}$ → рациональная часть $g_\theta^+$ ≡ 0

---

## Порядок параметров в массиве (p[0..43])

```python
# p[0..2]:   A_bw, a_bw, gW
# p[3..5]:   AH, mH, gH
# p[6..9]:   AT, tz, tH, gT          (gt_neg rational)
# p[10..12]: BT, eT, cT              (gt_neg Gaussian)
# p[13..15]: C0n, An, nn             (C_neg sinh)
# p[16..18]: Dn, en, cn              (C_neg Gaussian 1)
# p[19..21]: Dn2, en2, cn2           (C_neg Gaussian 2)
# p[22..24]: C0p, Ac, nc             (C_pos sinh)
# p[25..26]: Bc, cc                  (C_pos Gaussian 1, center=0)
# p[27..29]: Dp, ep, cp              (C_pos Gaussian 2)
# p[30]:     k2
# p[31]:     alpha_F                 (DEGENERATE ≈0)
# p[32]:     k1
# p[33..35]: BT_pos, eT_pos, cT_pos  (gt_pos Gaussian)
# p[36]:     beta_tc                 (DEGENERATE ≈10)
# p[37..40]: AT_pos, tz_pos, tH_pos, gT_pos  (DEGENERATE, AT_pos≈0)
# p[41..43]: Dp2, ep2, cp2           (C_pos Gaussian 3)
```
