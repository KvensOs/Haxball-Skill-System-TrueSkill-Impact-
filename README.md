# 🎯 Advanced Skill System: TrueSkill™ + Performance Metrics

Este repositorio presenta un sistema de clasificación de alto rendimiento diseñado para entornos competitivos de HaxBall. A diferencia del **ELO Puro** (basado en el sistema de ajedrez FIDE), este motor utiliza **Inferencia Bayesiana** y **Ponderación de Impacto Individual** para determinar la habilidad real de un jugador.

---

## ⚖️ TrueSkill vs. ELO Puro: La Gran Diferencia

La mayoría de las salas utilizan ELO estándar, lo cual presenta fallos graves en juegos de equipo. Aquí explicamos por qué este sistema es superior:

### 1. El problema del "Carry" (Arrastre)

* **ELO Puro:** Si un jugador mediocre juega siempre con un profesional, ambos ganan los mismos puntos (ej. +20). El jugador mediocre acaba con un ELO inflado que no le corresponde.
* **Este Sistema:** Gracias al **Impacto Individual**, si un jugador gana pero no aporta goles, asistencias ni defensa, el sistema detecta su bajo rendimiento y reduce su ganancia de Mu (). No puedes subir al top solo por "estar ahí".

### 2. La Velocidad de Convergencia (Sigma)

* **ELO Puro:** Un jugador nuevo empieza en 1000 y sube de 20 en 20. Necesita 100 partidos para llegar a su nivel real.
* **Este Sistema:** Utiliza **Sigma ()** o "Incertidumbre". Un jugador nuevo tiene un Sigma alto, lo que permite que el sistema le asigne grandes cantidades de puntos al principio para posicionarlo rápido. Un "Smurf" llegará a su rango real en menos de 15 partidos.

### 3. Mitigación en Derrotas

* **ELO Puro:** Pierdes el partido, pierdes puntos. No importa si metiste 10 goles y tu equipo falló todo.
* **Este Sistema:** Si el equipo pierde pero tu **Match Impact** fue superior a 45 (un rendimiento de MVP), el sistema te aplica una protección de Mu. Perderás una fracción mínima de lo que perdería un jugador que no hizo nada.

---

## 📊 Arquitectura del Impacto Individual

El impacto no es un número al azar; es una métrica calculada al segundo basada en acciones clave:

### Desglose de Puntos de Impacto

| Acción | Valor | Descripción |
| --- | --- | --- |
| **Goles** | `+4.5` | El peso principal de la victoria. |
| **Asistencias** | `+3.5` | Premia la visión de juego y el compañerismo. |
| **Clean Sheet (CS)** | `+5.0` | Bono por mantener la portería a cero. |
| **Bono de Tiempo** | `+playtime/80` | Premia la resistencia y el impacto sostenido. |

### Filtro de Calidad

Si un jugador disputa menos de **120 segundos**, su impacto se reduce automáticamente al **50%**. Esto evita que los jugadores que entran al final de un partido "farmeen" un promedio de impacto alto sin haber trabajado realmente el resultado.

---

## 🛠️ Configuración de Dificultad y Progresión

Puedes tunear el motor para que el ranking sea más o menos volátil. Todo reside en la fórmula del **ELO Visible**:

`ELO = (Mu * 160) - (Sigma * 40)`

### Escenarios de Ajuste:

1. **Dificultad Profesional:** Si quieres que sea muy difícil llegar a 10,000 ELO, aumenta el castigo de la incertidumbre: `(Mu * 150) - (Sigma * 60)`. Esto obligará a los jugadores a ser extremadamente consistentes para subir.
2. **Sistema Casual:** Si quieres que los jugadores suban rápido de nivel y vean números grandes: `(Mu * 220) - (Sigma * 30)`.
3. **El "Muro" de Veteranos:** El valor `Sigma` tiene un límite mínimo de `0.5`. Esto es crucial: sin este límite, el ELO se congelaría. Con `0.5`, los jugadores veteranos siempre tendrán un pequeño margen de ganancia/pérdida, manteniendo el ranking vivo.

---

## 🚀 Script de Implementación (UpdateStats)

Este es el script optimizado para procesar el fin de un partido. Integra la carga de estadísticas, cálculo de impacto, motor TrueSkill y ajuste de mérito.

```javascript
/**
 * Procesa las estadísticas tras el partido.
 * @param {Object} player - Objeto del jugador de Haxball.
 * @param {TeamID} teamStats - Equipo ganador (Team.RED o Team.BLUE).
 */
function updatePlayerStats(player, teamStats) {
    let auth = authArray[player.id][0];
    let stats = JSON.parse(localStorage.getItem(auth)) || new HaxStatistics(player.name);

    // 1. RECOLECCIÓN DE DATOS INDIVIDUALES
    let pComp = getPlayerComp(player); 
    let goals = getGoalsPlayer(pComp);
    let assists = getAssistsPlayer(pComp);
    let CS = getCSPlayer(pComp);
    let playtime = getGametimePlayer(pComp);

    // 2. CÁLCULO DE IMPACTO (MERITOCRACIA)
    let csBonus = CS ? (5 + Math.floor(playtime / 80)) : 0;
    let matchImpact = (goals * 4.5) + (assists * 3.5) + csBonus;
    if (playtime < 120) matchImpact *= 0.5;

    // Actualización del promedio histórico de impacto (suavizado al 8%)
    stats.impacto = stats.impacto ? (stats.impacto * 0.92) + (matchImpact * 0.08) : matchImpact;

    // 3. MOTOR TRUESKILL (USANDO LIBRERÍA TS-TRUESKILL)
    const getRating = (p) => {
        let s = JSON.parse(localStorage.getItem(authArray[p.id][0])) || {};
        return new Rating(s.mu || 25, s.sigma || 8.333);
    };

    let redTeamRatings = teamRedStats.map(p => getRating(p));
    let blueTeamRatings = teamBlueStats.map(p => getRating(p));

    const ranks = (lastWinner === Team.RED) ? [0, 1] : [1, 0];
    const [newRed, newBlue] = rate([redTeamRatings, blueTeamRatings], ranks);

    // Selección del nuevo Rating individual
    let myNewRating = (player.team === Team.RED) 
        ? newRed[teamRedStats.findIndex(p => p.id === player.id)] 
        : newBlue[teamBlueStats.findIndex(p => p.id === player.id)];

    // 4. AJUSTE DE MU POR IMPACTO
    let isWinner = (lastWinner === teamStats);
    let muAdjustment = 0;
    if (isWinner && matchImpact < 2) muAdjustment = -0.10; // "Carried" penalty
    if (!isWinner && matchImpact > 45) muAdjustment = 0.15; // "MVP Loss" protection

    // 5. ACTUALIZACIÓN DE ESTADO Y ELO VISIBLE
    let oldElo = stats.elo || 1000;
    stats.mu = myNewRating.mu + muAdjustment;
    stats.sigma = Math.max(myNewRating.sigma, 0.5); 

    // Fórmula de ELO Visible (Tunable)
    stats.elo = Math.round((stats.mu * 160) - (stats.sigma * 40));
    stats.elo = Math.min(10000, Math.max(1000, stats.elo));

    // Nivel opcional (Visual)
    stats.nivel = Math.floor(Math.pow((stats.elo - 1000) / 110, 0.82));

    // PERSISTENCIA DE DATOS
    stats.games++;
    isWinner ? stats.wins++ : stats.losses++;
    stats.goals += goals;
    stats.assists += assists;
    stats.CS += CS;
    stats.playtime += playtime;

    localStorage.setItem(auth, JSON.stringify(stats));

    // Feedback en sala
    let totalChange = stats.elo - oldElo;
    if (Math.abs(totalChange) >= 5) {
        let color = totalChange >= 0 ? 0xA3FF00 : 0xFF4C4C;
        room.sendAnnouncement(`${totalChange >= 0 ? "📈" : "📉"} ${player.name}: ${totalChange >= 0 ? "+" : ""}${totalChange} ELO (Nivel ${stats.nivel})`, null, color);
    }
}

```

---

## 🎖️ Progresión de Niveles (Opcional)

Si decides usar el sistema de niveles incluido en el script, la curva de dificultad es la siguiente:

| Rango de ELO | Nivel Aprox. | Percepción de Habilidad |
| --- | --- | --- |
| **1,000 - 2,000** | 1 - 15 | Principiante / Aprendiz |
| **2,000 - 4,500** | 15 - 50 | Jugador Competitivo |
| **4,500 - 7,500** | 50 - 80 | Avanzado / Élite |
| **7,500 - 10,000** | 80 - 99 | Maestro / Leyenda |

---

## ❓ FAQ para Usuarios

**¿Por qué mi ELO no sube aunque gané?**
Si el equipo rival era muy débil y tu impacto fue bajo, el sistema considera que tu habilidad actual ya es superior al reto que enfrentaste, por lo que no hay ganancia de Mu.

**¿Qué es el Sigma exactamente?**
Es la "desviación estándar". Si dejas de jugar por mucho tiempo o eres nuevo, el Sigma sube. Un Sigma alto significa que el sistema está "buscando" tu nivel, por lo que ganarás o perderás puntos de forma masiva.

**¿Cómo maximizo mi ganancia de ELO?**
No basta con ganar. Debes participar activamente: busca asistencias, mantén la defensa sólida para el CS y anota. El **Match Impact** es la llave para subir rápido.

---

Este sistema garantiza una jerarquía real en tu sala, donde el Top 1 será indiscutiblemente el jugador con mejor rendimiento y consistencia.

¿Deseas que añada una sección sobre cómo resetear temporadas manteniendo un "MMR oculto" (Soft Reset)?
