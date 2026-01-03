<div align="center">

# Sistema TrueSkill + Impact para HaxBall

![TrueSkill](https://img.shields.io/badge/Algorithm-TrueSkill-blue?style=flat-square)
![JavaScript](https://img.shields.io/badge/Language-JavaScript-yellow?style=flat-square)

Sistema de ranking inteligente que combina TrueSkill con métricas de rendimiento individual

</div>

---

## Por qué hice esto

Lo subí porque varios me lo pidieron. Básicamente es un sistema que mezcla TrueSkill (el algoritmo de Xbox Live) con stats de performance para que el ranking refleje mejor tu nivel real.

El problema que resuelve es simple: en ELO normal, si tu equipo gana 5-0 pero vos no hiciste nada, subís igual que el que metió los 5 goles. Con esto no pasa.

### Diferencia clave vs ELO tradicional

**Sistema Normal:**
```
Equipo gana 5-0
- Jugador A (5 goles): +25 ELO
- Jugador B (AFK): +25 ELO
```

**Este Sistema:**
```
Equipo gana 5-0
- Jugador A (Impact 68): +35 ELO
- Jugador B (Impact 2): +6 ELO
```

---

## Cómo funciona

El sistema tiene 3 partes principales:

### 1. TrueSkill Base

Usa dos valores:
- **μ (Mu):** Tu habilidad estimada (arranca en 25)
- **σ (Sigma):** Incertidumbre sobre tu nivel (arranca en 8.333)

Cuando sos nuevo, sigma es alto y los cambios son grandes. Con el tiempo sigma baja y el ELO se estabiliza.

| Partidos | Sigma | Cambios Típicos |
|----------|-------|-----------------|
| 0-20 | ~8.0 | ±100-150 |
| 20-50 | ~3.0 | ±30-60 |
| 50-150 | ~1.5 | ±15-30 |
| 150+ | ~0.5 | ±5-15 |

### 2. Match Impact

Calcula tu rendimiento en el partido:

```javascript
Impact = (goles × 4.5) + (asists × 3.5) + bonus_CS
```

**Valores:**
- Gol: 4.5 puntos
- Asistencia: 3.5 puntos
- Clean Sheet: 5.0 + (tiempo/80) puntos
- Tiempo jugado: bonus incremental

Si jugaste menos de 2 minutos, el impact se reduce al 50% para evitar farming.

### 3. Ajustes por Mérito

Hay dos protecciones:

**Anti-Carry:** Si ganás pero tu impact es bajo (menos de 2), hay una penalización de -0.10 mu.

**Protección en Derrotas:** Si perdés pero tu impact es alto (más de 45), la pérdida se reduce con un ajuste de +0.15 mu.

El ELO visible se calcula:
```
ELO = (μ × 160) - (σ × 40)
```

---

## Qué espero de esto

La idea es que después de probar el sistema:

- Los jugadores que realmente cargan equipos suban más rápido
- Los que se afkean o juegan mal no suban solo por estar en el equipo ganador
- La convergencia sea más rápida (30-50 partidos en vez de 200+)
- El ranking refleje mejor el nivel real de cada uno

También espero que ayude a balancear mejor las partidas. Si el sistema funciona, los equipos deberían quedar más parejos porque el ELO va a ser más preciso.

---

## Instalación

Esto va sobre el script de **Wazar94** ([HaxBot_public.js](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js)).

### Paso 1: Agregar propiedades TrueSkill

```javascript
function HaxStatistics(name) {
    this.name = name;
    
    // propiedades trueskill
    this.mu = 25.0;
    this.sigma = 8.333;
    this.elo = 1000;
    this.impacto = 0;
    this.nivel = 0;
    
    // stats normales
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

### Paso 2: Motor principal

```javascript
function updatePlayerStats(player, teamStats) {
    // cargar datos del jugador
    let auth = authArray[player.id][0];
    let stats = localStorage.getItem(auth) 
        ? Object.assign(new HaxStatistics(player.name), JSON.parse(localStorage.getItem(auth)))
        : new HaxStatistics(player.name);
    
    let pComp = getPlayerComp(player);
    stats.games++;
    
    let isWinner = (lastWinner === teamStats);
    isWinner ? stats.wins++ : stats.losses++;
    
    // obtener metricas del partido
    let goals = getGoalsPlayer(pComp);
    let assists = getAssistsPlayer(pComp);
    let CS = getCSPlayer(pComp);
    let playtime = getGametimePlayer(pComp);
    
    // calcular match impact
    let csBonus = CS ? (5 + Math.floor(playtime / 80)) : 0;
    let matchImpact = (goals * 4.5) + (assists * 3.5) + csBonus;
    
    if (playtime < 120) matchImpact *= 0.5;
    
    // promedio historico (suavizado exponencial)
    stats.impacto = stats.impacto 
        ? (stats.impacto * 0.92) + (matchImpact * 0.08) 
        : matchImpact;
    
    // obtener ratings de todos
    const getRating = (p) => {
        let storage = JSON.parse(localStorage.getItem(authArray[p.id][0])) || {};
        return new Rating(storage.mu || 25, storage.sigma || 8.333);
    };
    
    let redTeamRatings = teamRedStats.map(p => getRating(p));
    let blueTeamRatings = teamBlueStats.map(p => getRating(p));
    
    // calcular nuevos ratings
    const ranks = (lastWinner === Team.RED) ? [0, 1] : [1, 0];
    const [newRed, newBlue] = rate([redTeamRatings, blueTeamRatings], ranks);
    
    // extraer mi nuevo rating
    let myNewRating;
    if (player.team === Team.RED) {
        let index = teamRedStats.findIndex(p => p.id === player.id);
        myNewRating = newRed[index];
    } else {
        let index = teamBlueStats.findIndex(p => p.id === player.id);
        myNewRating = newBlue[index];
    }
    
    // ajustes por merito
    let muAdjustment = 0;
    
    if (isWinner && matchImpact < 2) muAdjustment = -0.10;
    if (!isWinner && matchImpact > 45) muAdjustment = 0.15;
    
    // actualizar elo
    let oldElo = stats.elo || 1000;
    
    stats.mu = myNewRating.mu + muAdjustment;
    stats.sigma = Math.max(myNewRating.sigma, 0.5);
    
    let calculatedElo = Math.round((stats.mu * 160) - (stats.sigma * 40));
    stats.elo = Math.min(10000, Math.max(1000, calculatedElo));
    
    let totalChange = stats.elo - oldElo;
    
    const prevLevel = stats.nivel || 0;
    stats.nivel = Math.min(99, Math.floor(Math.pow((stats.elo - 1000) / 110, 0.82)));
    
    // actualizar resto de stats
    stats.goals += goals;
    stats.assists += assists;
    stats.CS += CS;
    stats.playtime += playtime;
    stats.winrate = ((stats.wins / stats.games) * 100).toFixed(1) + "%";
    
    // guardar
    localStorage.setItem(auth, JSON.stringify(stats));
    
    // notificar cambios significativos
    if (stats.nivel > prevLevel || Math.abs(totalChange) >= 5) {
        let color = totalChange >= 0 ? 0xA3FF00 : 0xFF4C4C;
        let sign = totalChange >= 0 ? "+" : "";
        
        room.sendAnnouncement(
            `${player.name}: ${sign}${totalChange} ELO (Nivel ${stats.nivel})`,
            null,
            color,
            "normal"
        );
    }
}
```

---

## Configuración opcional

### Ajustar dificultad

```javascript
// normal (recomendado)
let calculatedElo = Math.round((stats.mu * 160) - (stats.sigma * 40));

// mas dificil
let calculatedElo = Math.round((stats.mu * 155) - (stats.sigma * 50));

// muy dificil
let calculatedElo = Math.round((stats.mu * 150) - (stats.sigma * 60));
```

### Modificar pesos de impact

```javascript
// valorar mas las asists
let matchImpact = (goals * 4.0) + (assists * 4.5) + csBonus;

// castigar mas el afk
if (playtime < 180) matchImpact *= 0.3;
```

### Ajustar protecciones

```javascript
// anti-carry mas estricto
if (isWinner && matchImpact < 5) muAdjustment = -0.15;

// proteccion mas generosa
if (!isWinner && matchImpact > 30) muAdjustment = 0.20;
```

---

## Distribución de niveles

<div align="center">

| Nivel | ELO | Descripción |
|-------|-----|-------------|
| 1-15 | 1,000 - 2,000 | Principiante |
| 15-30 | 2,000 - 3,000 | Amateur |
| 30-50 | 3,000 - 4,500 | Competitivo |
| 50-65 | 4,500 - 6,000 | Semi-Pro |
| 65-80 | 6,000 - 7,500 | Elite |
| 80-99 | 7,500 - 10,000 | Leyenda |

</div>

---

## Notas

El sistema no es perfecto. Si jugás de arquero y tu equipo domina, vas a tener poco impact porque no tocás la pelota. El clean sheet ayuda pero no compensa tanto como meter goles.

Los valores de 4.5 y 3.5 para goles/asists los elegí probando. Capaz en tu sala necesites ajustarlos dependiendo del estilo de juego.

El filtro de 2 minutos es para evitar farming (entrar y salir rápido para acumular stats falsas).

---

**Referencias:**
- Script base: [HaxBot_public.js](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js) (Wazar94)
- Algoritmo: [TrueSkill](https://www.microsoft.com/en-us/research/project/trueskill-ranking-system/) (Microsoft Research)
- Librería: [trueskill](https://www.npmjs.com/package/trueskill) (NPM)
