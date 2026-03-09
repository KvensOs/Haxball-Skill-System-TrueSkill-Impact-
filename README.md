<div align="center">

# Sistema TrueSkill + Impact para HaxBall

![TrueSkill](https://img.shields.io/badge/Algorithm-TrueSkill-blue?style=flat-square)
![JavaScript](https://img.shields.io/badge/Language-JavaScript-yellow?style=flat-square)

Un sistema de ranking que mezcla **TrueSkill** con algunas métricas de rendimiento del jugador.

</div>

---

## Por qué hice esto

Subo esto porque varias personas me preguntaron cómo estaba manejando el ranking en mi sala.

La idea es simple: usar **TrueSkill** como base, pero añadir algunas métricas del partido para que el sistema tenga en cuenta **qué tanto aportó cada jugador**, no solo si su equipo ganó o perdió.

El problema del ELO clásico es que **todos ganan o pierden lo mismo**, aunque uno haya hecho todo y otro nada.

### Ejemplo rápido

**Sistema tradicional:**

```
Equipo gana 5-0
Jugador A (5 goles): +25 ELO
Jugador B (AFK): +25 ELO
```

**Con este sistema:**

```
Equipo gana 5-0
Jugador A (Impact 68): +35 ELO
Jugador B (Impact 2): +6 ELO
```

No es perfecto, pero ayuda bastante a que el ranking refleje mejor quién realmente está jugando bien.

---

## Cómo funciona

El sistema tiene tres partes principales.

### 1. TrueSkill

Cada jugador tiene dos valores:

* **μ (mu)** → estimación de habilidad
* **σ (sigma)** → qué tan seguro está el sistema de ese valor

Valores iniciales:

```
mu = 25
sigma = 8.333
```

Cuando alguien es nuevo, **sigma es alto**, entonces el rating cambia mucho.
Con más partidas, sigma baja y el ranking se vuelve más estable.

| Partidos | Sigma aprox | Cambios típicos |
| -------- | ----------- | --------------- |
| 0-20     | ~8          | ±100-150        |
| 20-50    | ~3          | ±30-60          |
| 50-150   | ~1.5        | ±15-30          |
| 150+     | ~0.5        | ±5-15           |

---

### 2. Impact del partido

Después del match se calcula una puntuación simple de rendimiento:

```
Impact = (goles × 4.5) + (asists × 3.5) + bonus_CS
```

Valores actuales:

* Gol → **4.5**
* Asistencia → **3.5**
* Clean sheet → **5 + (tiempo / 80)**
* Tiempo jugado → pequeño bonus

Si alguien jugó **menos de 2 minutos**, el impact se reduce a la mitad para evitar farming.

---

### 3. Ajustes por mérito

Hay dos pequeños ajustes:

**Anti-carry**

Si tu equipo gana pero tu impact es muy bajo:

```
impact < 2
```

Se aplica una pequeña penalización.

**Protección en derrotas**

Si perdés pero jugaste muy bien:

```
impact > 45
```

La pérdida de rating se reduce un poco.

---

## Cómo se calcula el ELO visible

El ranking que ve la gente se calcula así:

```
ELO = (mu × 160) − (sigma × 40)
```

Con esto el sistema penaliza un poco la **incertidumbre alta**, que suele ser el caso de jugadores nuevos.

---

## Qué busco con esto

Principalmente:

* Que los jugadores que **realmente cargan partidas** suban más rápido
* Que los AFK o jugadores pasivos **no suban solo por ganar**
* Que el ranking se estabilice más rápido
* Tener equipos más balanceados al armar partidas

No es un sistema perfecto, pero en la práctica funciona bastante bien.

---

## Instalación

Esto está pensado para usarse sobre el script de **Wazar94**:

[https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js)

### 1. Agregar propiedades TrueSkill

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

*(motor principal igual que en tu ejemplo)*

---

## Configuración opcional

Podés ajustar varias cosas dependiendo del estilo de tu sala.

### Cambiar dificultad del ranking

```javascript
// normal
(mu * 160) - (sigma * 40)

// más difícil
(mu * 155) - (sigma * 50)

// muy difícil
(mu * 150) - (sigma * 60)
```

---

### Cambiar peso del impact

```javascript
// dar más valor a las asistencias
(goals * 4.0) + (assists * 4.5)
```

---

### Ajustar protecciones

```javascript
// anti carry más estricto
if (isWinner && matchImpact < 5) muAdjustment = -0.15;

// protección más fuerte
if (!isWinner && matchImpact > 30) muAdjustment = 0.20;
```

---

## Distribución de niveles

| Nivel | ELO        | Descripción  |
| ----- | ---------- | ------------ |
| 1-15  | 1000-2000  | Principiante |
| 15-30 | 2000-3000  | Amateur      |
| 30-50 | 3000-4500  | Competitivo  |
| 50-65 | 4500-6000  | Semi-Pro     |
| 65-80 | 6000-7500  | Elite        |
| 80-99 | 7500-10000 | Leyenda      |

---

## Notas

No es un sistema perfecto.

Por ejemplo, si jugás de arquero y tu equipo domina, probablemente tengas poco impact porque casi no participás en jugadas ofensivas. El clean sheet ayuda un poco pero no compensa completamente.

Los valores de **4.5 y 3.5** los elegí probando varias partidas. Dependiendo del estilo de tu sala capaz tengas que ajustarlos.

El filtro de **2 minutos** es para evitar que alguien entre, toque una pelota y salga para farmear stats.

---

## Referencias

Script base
[https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js)

TrueSkill
[https://www.microsoft.com/en-us/research/project/trueskill-ranking-system/](https://www.microsoft.com/en-us/research/project/trueskill-ranking-system/)

Librería
[https://www.npmjs.com/package/trueskill](https://www.npmjs.com/package/trueskill)

---
