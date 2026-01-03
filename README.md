<div align="center">

# 🎯 Advanced Skill System: TrueSkill™ + Performance Metrics

<img src="https://img.shields.io/badge/Algorithm-TrueSkill-blue?style=for-the-badge&logo=microsoft" alt="TrueSkill"/>
<img src="https://img.shields.io/badge/Platform-HaxBall-red?style=for-the-badge&logo=html5" alt="HaxBall"/>
<img src="https://img.shields.io/badge/Language-JavaScript-yellow?style=for-the-badge&logo=javascript" alt="JavaScript"/>
<img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="License"/>

### Sistema de Ranking Inteligente para Juegos de Equipo

*Porque ganar sin hacer nada no debería subir tu ELO*

[🚀 Inicio Rápido](#-instalación-rápida) • [📖 Docs](#-cómo-funciona) • [❓ FAQ](#-preguntas-frecuentes)

---

</div>

## 🤔 ¿Qué es esto?

Un sistema de ranking **justo** para HaxBall que resuelve el problema más grande de los sistemas tradicionales:

> **"Mi compañero hizo 5 goles y yo ninguno, pero ambos subimos lo mismo"**

Este motor usa **TrueSkill™** (el algoritmo de Xbox Live) + **Match Impact** para dar puntos según tu **desempeño real**, no solo por estar en el equipo ganador.

---

## 🔥 ¿Por qué necesitas esto?

### El Problema del ELO Normal

```
🏆 Tu equipo gana 5-0

Jugador A: 5 goles, 3 asistencias → +25 ELO
Jugador B: AFK todo el partido → +25 ELO

❌ ¿Justo? NO.
```

### La Solución: TrueSkill + Impact

```
🏆 Tu equipo gana 5-0

Jugador A: Impact 68 → +35 ELO ⚡
Jugador B: Impact 2 → +6 ELO 😴

✅ Ahora sí tiene sentido.
```

---

## ⚖️ TrueSkill vs ELO: Las 3 Diferencias Clave

### 1. 🎮 Anti-Carry

**ELO Puro:** Te suben por estar en el equipo ganador, aunque no hayas tocado la bola.

**Este Sistema:** Si ganas pero tu **Match Impact** es bajo (menos de 2), tu ganancia se **reduce** automáticamente.

```javascript
if (isWinner && matchImpact < 2) muAdjustment = -0.10;
```

---

### 2. 🚀 Convergencia Rápida

**ELO Puro:** Necesitas 200+ partidos para llegar a tu nivel real.

**Este Sistema:** Usa **Sigma (σ)** = "Incertidumbre". Cuando eres nuevo, tu σ es alto (8.333) y ganas/pierdes **mucho** ELO por partido. A medida que juegas, σ baja y tu ELO se estabiliza.

| Partidos Jugados | Sigma (σ) | Ganancia/Pérdida |
|------------------|-----------|------------------|
| 0 - 20 | ~8.0 | ±100 - 150 |
| 20 - 50 | ~3.0 | ±30 - 60 |
| 50 - 150 | ~1.5 | ±15 - 30 |
| 150+ | ~0.5 | ±5 - 15 |

---

### 3. 🛡️ Protección en Derrotas

**ELO Puro:** Pierdes 20 puntos fijos, sin importar si marcaste 3 goles.

**Este Sistema:** Si pierdes pero tu **Match Impact** es alto (más de 45), la pérdida se **reduce drásticamente**.

```javascript
if (!isWinner && matchImpact > 45) muAdjustment = 0.15;
```

**Ejemplo real:**
- Pérdida normal: -22 ELO
- Con protección: -3 ELO ✨

---

## 📊 Match Impact: El Corazón del Sistema

### ⚽ Valores Base

| Acción | Puntos | Razón |
|--------|--------|-------|
| **Gol** | +4.5 | Define el resultado |
| **Asistencia** | +3.5 | Crea el juego |
| **Clean Sheet** | +5.0 | Defensa sólida |
| **Bono Tiempo** | +tiempo/80 | Recompensa consistencia |

### 🧮 Fórmula

```javascript
let matchImpact = (goles * 4.5) + (asistencias * 3.5) + csBonus;

// Si jugaste menos de 2 minutos, se reduce al 50%
if (playtime < 120) matchImpact *= 0.5;
```

### 📈 Ejemplos

| Escenario | Cálculo | Impact |
|-----------|---------|--------|
| **Hat-trick + 2 asists** | (3×4.5) + (2×3.5) = 20.5 | 🔥 20.5 |
| **Portero CS (8 min)** | 5 + (480/80) = 11 | ⭐ 11.0 |
| **1 gol AFK (90s)** | (1×4.5) × 0.5 = 2.25 | 😴 2.25 |

---

## 🔬 Fórmulas Técnicas

### 1. ELO Visible

```
ELO = (μ × 160) - (σ × 40)
```

- **μ (Mu):** Tu habilidad estimada (empieza en 25)
- **σ (Sigma):** Cuánto duda el sistema de tu nivel (empieza en 8.333)

### 2. Niveles (Opcional)

```javascript
nivel = Math.floor(Math.pow((elo - 1000) / 110, 0.82));
```

<div align="center">

| ELO | Nivel | Rango |
|-----|-------|-------|
| 1,000 - 2,000 | 1 - 15 | 🔰 Principiante |
| 2,000 - 4,500 | 15 - 50 | ⚔️ Competitivo |
| 4,500 - 7,500 | 50 - 80 | 🔥 Élite |
| 7,500 - 10,000 | 80 - 99 | 🏆 Leyenda |

</div>

---

## 🚀 Instalación Rápida

### Paso 1: Descargar Base

Este sistema funciona sobre el script de **Wazar94**:

🔗 [HaxBot_public.js](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js)

### Paso 2: Agregar Propiedades

Modifica tu clase `HaxStatistics`:

```javascript
function HaxStatistics(name) {
    this.name = name;
    
    // ✨ Propiedades TrueSkill (OBLIGATORIAS)
    this.mu = 25.0;
    this.sigma = 8.333;
    this.elo = 1000;
    this.impacto = 0;
    
    // 🎖️ Opcionales
    this.nivel = 0;
    this.ownGoals = 0;
    
    // 📊 Resto de stats
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

### Paso 3: Copiar el Motor

```javascript
/**
 * 🎯 SISTEMA TRUESKILL + IMPACT
 * Basado en: HaxBot_public.js (Wazar94)
 */

function updatePlayerStats(player, teamStats) {
    // ════════════════════════════════════════════
    // 1️⃣ CARGA DE DATOS
    // ════════════════════════════════════════════
    
    let auth = authArray[player.id][0];
    let stats = localStorage.getItem(auth) 
        ? Object.assign(new HaxStatistics(player.name), JSON.parse(localStorage.getItem(auth)))
        : new HaxStatistics(player.name);
    
    let pComp = getPlayerComp(player);
    stats.games++;
    
    let isWinner = (lastWinner === teamStats);
    isWinner ? stats.wins++ : stats.losses++;
    
    // ════════════════════════════════════════════
    // 2️⃣ RECOLECCIÓN DE MÉTRICAS
    // ════════════════════════════════════════════
    
    let goals = getGoalsPlayer(pComp);
    let assists = getAssistsPlayer(pComp);
    let CS = getCSPlayer(pComp);
    let playtime = getGametimePlayer(pComp);
    
    // ════════════════════════════════════════════
    // 3️⃣ CÁLCULO DE MATCH IMPACT
    // ════════════════════════════════════════════
    
    let csBonus = CS ? (5 + Math.floor(playtime / 80)) : 0;
    let matchImpact = (goals * 4.5) + (assists * 3.5) + csBonus;
    
    // Filtro anti-farm (jugadas muy cortas)
    if (playtime < 120) matchImpact *= 0.5;
    
    // Promedio histórico (suavizado exponencial)
    stats.impacto = stats.impacto 
        ? (stats.impacto * 0.92) + (matchImpact * 0.08) 
        : matchImpact;
    
    // ════════════════════════════════════════════
    // 4️⃣ MOTOR TRUESKILL
    // ════════════════════════════════════════════
    
    // Helper para obtener Rating de un jugador
    const getRating = (p) => {
        let storage = JSON.parse(localStorage.getItem(authArray[p.id][0])) || {};
        return new Rating(storage.mu || 25, storage.sigma || 8.333);
    };
    
    // Obtener ratings de ambos equipos
    let redTeamRatings = teamRedStats.map(p => getRating(p));
    let blueTeamRatings = teamBlueStats.map(p => getRating(p));
    
    // Calcular nuevos ratings (0=ganador, 1=perdedor)
    const ranks = (lastWinner === Team.RED) ? [0, 1] : [1, 0];
    const [newRed, newBlue] = rate([redTeamRatings, blueTeamRatings], ranks);
    
    // Extraer mi nuevo rating
    let myNewRating;
    if (player.team === Team.RED) {
        let index = teamRedStats.findIndex(p => p.id === player.id);
        myNewRating = newRed[index];
    } else {
        let index = teamBlueStats.findIndex(p => p.id === player.id);
        myNewRating = newBlue[index];
    }
    
    // ════════════════════════════════════════════
    // 5️⃣ AJUSTES POR MÉRITO
    // ════════════════════════════════════════════
    
    let muAdjustment = 0;
    
    // Anti-Carry: ganas pero no hiciste nada
    if (isWinner && matchImpact < 2) muAdjustment = -0.10;
    
    // Protección: pierdes pero jugaste increíble
    if (!isWinner && matchImpact > 45) muAdjustment = 0.15;
    
    // ════════════════════════════════════════════
    // 6️⃣ ACTUALIZACIÓN FINAL
    // ════════════════════════════════════════════
    
    let oldElo = stats.elo || 1000;
    
    // Aplicar TrueSkill + Ajuste
    stats.mu = myNewRating.mu + muAdjustment;
    stats.sigma = Math.max(myNewRating.sigma, 0.5); // Mínimo para mantener dinamismo
    
    // Calcular ELO visible
    let calculatedElo = Math.round((stats.mu * 160) - (stats.sigma * 40));
    stats.elo = Math.min(10000, Math.max(1000, calculatedElo));
    
    let totalChange = stats.elo - oldElo;
    
    // Calcular nivel (opcional)
    const prevLevel = stats.nivel || 0;
    stats.nivel = Math.min(99, Math.floor(Math.pow((stats.elo - 1000) / 110, 0.82)));
    
    // ════════════════════════════════════════════
    // 7️⃣ ACTUALIZAR ESTADÍSTICAS
    // ════════════════════════════════════════════
    
    stats.goals += goals;
    stats.assists += assists;
    stats.CS += CS;
    stats.playtime += playtime;
    stats.winrate = ((stats.wins / stats.games) * 100).toFixed(1) + "%";
    
    // ════════════════════════════════════════════
    // 8️⃣ GUARDAR Y NOTIFICAR
    // ════════════════════════════════════════════
    
    localStorage.setItem(auth, JSON.stringify(stats));
    
    // Notificar solo cambios significativos
    if (stats.nivel > prevLevel || Math.abs(totalChange) >= 5) {
        let color = totalChange >= 0 ? 0xA3FF00 : 0xFF4C4C;
        let icon = totalChange >= 0 ? "📈" : "📉";
        
        room.sendAnnouncement(
            `${icon} ${player.name}: ${totalChange >= 0 ? "+" : ""}${totalChange} ELO (Nivel ${stats.nivel})`,
            null,
            color,
            "normal"
        );
    }
}
```

---

## ⚙️ Configuración Personalizada

### 🎚️ Ajustar Dificultad

Si quieres un ranking más **exigente**, cambia los multiplicadores:

```javascript
// Normal (Recomendado)
let calculatedElo = Math.round((stats.mu * 160) - (stats.sigma * 40));

// Competitivo (Más difícil subir)
let calculatedElo = Math.round((stats.mu * 155) - (stats.sigma * 50));

// Pro League (Muy exigente)
let calculatedElo = Math.round((stats.mu * 150) - (stats.sigma * 60));
```

### 🔧 Ajustar Pesos de Impacto

```javascript
// Si quieres valorar más las asistencias:
let matchImpact = (goals * 4.0) + (assists * 4.0) + csBonus;

// Si quieres castigar más a los AFK:
if (playtime < 180) matchImpact *= 0.3; // 70% de reducción
```

### 🛡️ Ajustar Protecciones

```javascript
// Carry más estricto (necesitas impact 5 para no ser penalizado)
if (isWinner && matchImpact < 5) muAdjustment = -0.15;

// Protección más generosa en derrotas
if (!isWinner && matchImpact > 30) muAdjustment = 0.20;
```

---

## 📖 Cómo Funciona (Explicación Simple)

### 1. Recolectas tus Stats
- Goles, asistencias, tiempo jugado, clean sheet

### 2. Calculas tu Impacto
```
Impact = (goles × 4.5) + (asistencias × 3.5) + bonus
```

### 3. TrueSkill Calcula Probabilidad
- "Equipo A tiene 65% de ganar"
- Si ganan: pequeña ganancia (era esperado)
- Si pierden: gran pérdida (sorpresa)

### 4. Ajuste por Mérito
- Si ganaste haciendo nada: **penalización**
- Si perdiste pero jugaste increíble: **protección**

### 5. Tu ELO se Actualiza
```
ELO = (μ × 160) - (σ × 40)
```

---

## 🎖️ Sistema de Niveles (Visual)

<div align="center">

| Nivel | ELO | Rango | Emoji |
|-------|-----|-------|-------|
| 1-5 | 1,000 - 1,500 | Novato | 🥉 |
| 5-15 | 1,500 - 2,000 | Principiante | 🥈 |
| 15-30 | 2,000 - 3,000 | Amateur | 🥇 |
| 30-50 | 3,000 - 4,500 | Competitivo | 💎 |
| 50-65 | 4,500 - 6,000 | Semi-Pro | 💠 |
| 65-80 | 6,000 - 7,500 | Élite | 👑 |
| 80-90 | 7,500 - 9,000 | Maestro | 🌟 |
| 90-99 | 9,000 - 10,000 | Leyenda | ⚡ |

</div>

---

## ❓ Preguntas Frecuentes

### ❓ ¿Por qué no subo al ganar?

Si los rivales eran **muy inferiores** (su μ era bajo) y además tu **Match Impact** fue bajo, el sistema entiende que el partido era **demasiado fácil** para tu nivel. Ganar contra novatos no prueba que mejoraste.

### ❓ ¿Qué significa Sigma (σ)?

Es cuánto **duda** el sistema de tu nivel:
- **σ alto (8.0):** Eres nuevo, los cambios son grandes
- **σ bajo (0.5):** Eres veterano, los cambios son pequeños

### ❓ ¿Cómo subo más rápido?

La clave es el **Match Impact**:
- Goles y asistencias dan puntos directos
- Jugar todo el partido da bonus de tiempo
- Mantener portería a cero (CS) suma mucho

### ❓ ¿Por qué pierdo menos puntos a veces?

Si tu **Match Impact** supera 45 en una derrota, el sistema activa la **protección**. Detecta que hiciste todo lo posible y reduce la pérdida de ELO.

### ❓ ¿Puedo resetear mis stats?

Sí, borra tu `localStorage`:
```javascript
localStorage.removeItem('tu_auth_key');
```

---

## 🏆 Ventajas de Este Sistema

✅ **Justo:** Los puntos se ganan con esfuerzo, no solo estando en el equipo ganador

✅ **Rápido:** Nuevos jugadores convergen en 30-50 partidos (vs 200+ del ELO puro)

✅ **Protegido:** No pierdes tanto ELO si juegas bien en una derrota

✅ **Dinámico:** El ranking nunca se congela, siempre hay movimiento

✅ **Anti-Carry:** Detecta jugadores que suben sin mérito

---

## 🔗 Referencias

### Script Base
📦 [HaxBot_public.js - Wazar94](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js)

### Algoritmo TrueSkill
📚 [TrueSkill™ - Microsoft Research](https://www.microsoft.com/en-us/research/project/trueskill-ranking-system/)

### Librería JavaScript
💻 [trueskill - NPM](https://www.npmjs.com/package/trueskill)

---

## 🎯 Créditos

- **Motor TrueSkill:** Microsoft Research
- **Script Base:** [Wazar94](https://github.com/Wazarr94)
- **Sistema de Impacto:** Diseño personalizado para HaxBall

---

<div align="center">

### ⭐ Si este sistema te funciona, deja una estrella

**Hecho con** ❤️ **para la comunidad de HaxBall**

</div>
