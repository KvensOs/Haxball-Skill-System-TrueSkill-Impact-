<div align="center">

# TrueSkill + ELO para HaxBall

![TrueSkill](https://img.shields.io/badge/Algorithm-TrueSkill-blue?style=flat-square)
![JavaScript](https://img.shields.io/badge/Language-JavaScript-yellow?style=flat-square)

Ejemplo de cómo usar TrueSkill combinado con métricas de partido para calcular ELO en una sala de HaxBall.

</div>

---

## Por qué esto

Varias personas me preguntaron cómo estaba manejando el ranking en mi sala, así que lo subo.

El problema del ELO clásico es simple: si tu equipo gana, todos suman igual. No importa si hiciste 5 goles o estuviste AFK. Este sistema usa **TrueSkill** como base y le suma una puntuación de rendimiento por partido, para que el rating refleje algo más que solo ganar o perder.

```
// ELO tradicional — equipo gana 5-0
Jugador A (5 goles): +25
Jugador B (AFK):     +25

// Con este sistema
Jugador A (Impact 68): +35
Jugador B (Impact 2):   +6
```

No es perfecto, pero funciona bastante bien en la práctica.

---

## Cómo funciona

### TrueSkill

Cada jugador tiene dos valores internos:

- **μ (mu)** — estimación de habilidad
- **σ (sigma)** — incertidumbre del sistema sobre ese valor

```
Valores iniciales: mu = 25, sigma = 8.333
```

Cuando alguien es nuevo, sigma es alto y el rating cambia mucho. A medida que juega más partidas, sigma baja y el ranking se estabiliza.

| Partidos | Sigma aprox | Cambio típico por partido |
| -------- | ----------- | ------------------------- |
| 0–20     | ~8          | ±100–150                  |
| 20–50    | ~3          | ±30–60                    |
| 50–150   | ~1.5        | ±15–30                    |
| 150+     | ~0.5        | ±5–15                     |

### Impact del partido

Después de cada partido se calcula una puntuación de rendimiento individual:

```
Impact = (goles × 4.5) + (asistencias × 3.5) + bonus_CS
```

- Gol → **4.5**
- Asistencia → **3.5**
- Clean sheet → **5 + (tiempo / 80)**

Si alguien jugó menos de 2 minutos, el impact se reduce a la mitad. Esto evita que alguien entre, toque una pelota y salga para farmear stats.

### Ajustes por rendimiento

Dos casos especiales:

**Anti-carry** — si tu equipo gana pero tu impact es muy bajo (`< 2`), se aplica una pequeña penalización. Ganar sin participar no debería sumar lo mismo.

**Protección en derrota** — si perdiste pero tu impact fue alto (`> 45`), la pérdida de rating se reduce. Cargar un partido y perder igual no debería castigarse igual.

### ELO visible

El número que ve la gente se calcula así:

```
ELO = (mu × 160) − (sigma × 40)
```

Penaliza la incertidumbre alta, que es lo habitual en jugadores nuevos.

---

## Instalación

Está pensado para usarse sobre el script de [Wazar94](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js).

Agregar las propiedades TrueSkill al constructor de estadísticas:

```javascript
function HaxStatistics(name) {
    this.name = name;

    // trueskill
    this.mu = 25.0;
    this.sigma = 8.333;
    this.elo = 1000;
    this.impacto = 0;
    this.nivel = 0;

    // stats
    this.games = 0;
    this.wins = 0;
    this.losses = 0;
    this.goals = 0;
    this.assists = 0;
    this.CS = 0;
    this.playtime = 0;
    this.winrate = "0%";
}
```

---

## Configuración

Dependiendo del estilo de tu sala podés ajustar varias cosas.

**Dificultad del ranking**

```javascript
// normal
(mu * 160) - (sigma * 40)

// más difícil
(mu * 155) - (sigma * 50)

// muy difícil
(mu * 150) - (sigma * 60)
```

**Peso del impact**

```javascript
// dar más valor a las asistencias
(goals * 4.0) + (assists * 4.5)
```

**Ajustar los umbrales de protección**

```javascript
// anti-carry más estricto
if (isWinner && matchImpact < 5) muAdjustment = -0.15;

// protección más fuerte en derrotas
if (!isWinner && matchImpact > 30) muAdjustment = 0.20;
```

Los valores de `4.5` y `3.5` los fui ajustando probando partidas. Dependiendo del estilo de tu sala quizás necesites cambiarlos.

---

## Niveles

| Nivel | ELO        |
| ----- | ---------- |
| 1–15  | 1000–2000  |
| 15–30 | 2000–3000  |
| 30–50 | 3000–4500  |
| 50–65 | 4500–6000  |
| 65–80 | 6000–7500  |
| 80–99 | 7500–10000 |

---

## Notas

El sistema tiene limitaciones. Si jugás de arquero y tu equipo domina el partido, tu impact va a ser bajo aunque hayas jugado bien — el clean sheet ayuda pero no compensa del todo. Es algo a tener en cuenta si tu sala tiene posiciones fijas.

---

## Referencias

- Script base: [Wazarr94/haxball_bot_headless](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js)
- TrueSkill: [microsoft.com/research/trueskill](https://www.microsoft.com/en-us/research/project/trueskill-ranking-system/)
- Librería: [npmjs.com/package/trueskill](https://www.npmjs.com/package/trueskill)
