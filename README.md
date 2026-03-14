<div align="center">

# TrueSkill + ELO para HaxBall

![TrueSkill](https://img.shields.io/badge/Algorithm-TrueSkill-blue?style=flat-square)
![JavaScript](https://img.shields.io/badge/Language-JavaScript-yellow?style=flat-square)

Ejemplo de cómo integrar TrueSkill con métricas de partido para calcular ELO en una sala de HaxBall, basado en el script de [Wazarr94](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js).

</div>

---

## Por qué esto

Varias personas me preguntaron cómo estaba manejando el ranking en mi sala, así que lo subo.

El problema del ELO clásico es que si tu equipo gana, todos suman igual. No importa si hiciste 5 goles o estuviste AFK. Este sistema usa **TrueSkill** como base y le suma una puntuación de rendimiento individual por partido, para que el rating refleje algo más que solo ganar o perder.

```
// ELO tradicional — equipo gana 5-0
Jugador A (5 goles): +25
Jugador B (AFK):     +25

// Con este sistema
Jugador A (Impact alto): +35
Jugador B (Impact bajo):  +6
```

---

## Cómo funciona

### TrueSkill

Cada jugador tiene dos valores internos:

- **μ (mu)** — estimación de habilidad
- **σ (sigma)** — incertidumbre del sistema sobre ese valor

```
Valores iniciales: mu = 25, sigma = 8.333
```

Cuando alguien es nuevo, sigma es alto y el rating cambia mucho. A medida que juega más partidas, sigma baja y el ranking se estabiliza. El cálculo de TrueSkill se delega a un servidor local (`localhost:3000/calculate`) para no correr la librería directo en el script de HaxBall.

| Partidos | Sigma aprox | Cambio típico por partido |
| -------- | ----------- | ------------------------- |
| 0–20     | ~8          | ±100–150                  |
| 20–50    | ~3          | ±30–60                    |
| 50–150   | ~1.5        | ±15–30                    |
| 150+     | ~0.5        | ±5–15                     |

### Impact del partido

Después de cada partido se calcula una puntuación de rendimiento:

```
Impact = (goles × 3.2) + (asistencias × 2.8) + bonus_CS
Clean sheet bonus = 5 + floor(playtime / 110)
```

Este valor afecta dos cosas: cuánto se reduce la pérdida si perdiste jugando bien, y si se aplica una pequeña penalización si ganaste pero casi no participaste.

### ELO visible

```
ELO = (mu × 170) − (sigma × 45)
```

Penaliza la incertidumbre alta. El resultado se clampea entre `1000` y `10000`.

---

## Estructura del objeto de stats

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

## Ejemplo — función principal

Esta es la función que se llama al terminar cada partido. Recibe el jugador y las stats de su equipo, consulta el servidor TrueSkill, aplica los ajustes y guarda el resultado en `localStorage`.

```javascript
async function updatePlayerStats(player, teamStats) {
    const auth = authArray[player.id][0];
    const rawData = localStorage.getItem(auth);

    let stats = rawData
        ? Object.assign(new HaxStatistics(player.name), JSON.parse(rawData))
        : new HaxStatistics(player.name);

    // valores por defecto si el objeto no los tiene
    stats.mu    = stats.mu    ?? 25.0;
    stats.sigma = stats.sigma ?? 8.333;
    stats.elo   = stats.elo   ?? 1000;
    stats.games = stats.games ?? 0;

    const isNewPlayer = stats.games < 10;

    // sigma forzado en los primeros partidos para evitar swings enormes
    if (stats.games < 3)       stats.sigma = 3.0;
    else if (stats.games < 10) stats.sigma = Math.min(stats.sigma, 4.5);

    stats.games++;

    const isWinner = (lastWinner === teamStats);
    const isDraw   = (lastWinner === Team.SPECTATORS || lastWinner === null);

    if (isWinner)     stats.wins++;
    else if (!isDraw) stats.losses++;

    // stats del partido
    const pComp    = getPlayerComp(player);
    const goals    = getGoalsPlayer(pComp);
    const assists  = getAssistsPlayer(pComp);
    const CS       = getCSPlayer(pComp);
    const playtime = getGametimePlayer(pComp);

    // impact individual
    const csBonus     = CS ? (5 + Math.floor(playtime / 110)) : 0;
    const matchImpact = (goals * 3.2) + (assists * 2.8) + csBonus;

    // media móvil del impact histórico del jugador
    stats.impacto = stats.impacto
        ? (stats.impacto * 0.85) + (matchImpact * 0.15)
        : matchImpact;

    // ratings actuales de todos los jugadores para enviar al servidor
    const getRawRating = (p) => {
        const pData  = JSON.parse(localStorage.getItem(authArray[p.id][0])) || {};
        const pGames = pData.games || 0;
        let sigma    = pData.sigma ?? 8.333;
        if (pGames < 3)       sigma = 3.0;
        else if (pGames < 10) sigma = Math.min(sigma, 4.5);
        return { mu: pData.mu ?? 25.0, sigma };
    };

    const oldMu  = stats.mu;
    const oldElo = stats.elo;

    // consulta al servidor TrueSkill
    let myNewRating;
    try {
        const response = await fetch('http://localhost:3000/calculate', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                redTeam:     teamRedStats.map(p => getRawRating(p)),
                blueTeam:    teamBlueStats.map(p => getRawRating(p)),
                isDraw:      isDraw,
                winnerIsRed: lastWinner === Team.RED
            })
        });

        if (!response.ok) throw new Error('Server error');

        const result    = await response.json();
        const teamIndex = (player.team === Team.RED)
            ? teamRedStats.findIndex(p => p.id === player.id)
            : teamBlueStats.findIndex(p => p.id === player.id);

        myNewRating = (player.team === Team.RED)
            ? result.newRed[teamIndex]
            : result.newBlue[teamIndex];

    } catch (e) {
        console.error("❌ Error servidor ELO:", e.message);
        return;
    }

    let muDiff = myNewRating.mu - oldMu;

    // cap de ganancia para jugadores nuevos
    if (isNewPlayer && muDiff > 0) {
        const maxMuGain = stats.games <= 3 ? 1.2 : stats.games <= 7 ? 1.8 : 2.5;
        muDiff = Math.min(muDiff, maxMuGain);
    }

    if (isDraw) {
        muDiff *= 0.5;
    } else if (!isWinner) {
        // reducción de pérdida según impact
        if (muDiff < 0) {
            const reduction = matchImpact >= 12 ? 0.40 : matchImpact >= 7 ? 0.20 : 0;
            muDiff *= (1 - Math.min(reduction, 0.65));
        }
        if (muDiff >= 0) muDiff = -0.02; // siempre pierde algo si perdió
    }

    // factor por duración del partido
    const durationFactor = playtime < 100 ? 0.8 : playtime > 260 ? 1.1 : 1.0;

    stats.mu    = oldMu + (muDiff * durationFactor);
    stats.sigma = Math.max(myNewRating.sigma, 0.45);

    // ELO final
    let finalElo = Math.round((stats.mu * 170) - (stats.sigma * 45));

    // cap de cambio para jugadores nuevos
    if (isNewPlayer) {
        const maxChange = stats.games <= 3 ? 50 : stats.games <= 7 ? 80 : 120;
        const eloChange = finalElo - oldElo;
        if (Math.abs(eloChange) > maxChange) {
            finalElo = oldElo + (Math.sign(eloChange) * maxChange);
        }
    }

    stats.elo = Math.min(10000, Math.max(1000, finalElo));

    // si perdiste, el ELO no puede subir
    if (!isWinner && !isDraw && stats.elo > oldElo) {
        stats.elo = oldElo;
    }

    // nivel 0-99 basado en ELO
    stats.nivel = Math.max(0, Math.min(99,
        Math.floor(Math.pow((stats.elo - 1000) / 91, 0.82))
    ));

    stats.goals    += goals;
    stats.assists  += assists;
    stats.CS       += CS;
    stats.playtime += playtime;
    stats.winrate   = ((stats.wins / stats.games) * 100).toFixed(1) + "%";

    localStorage.setItem(auth, JSON.stringify(stats));

    // anuncio en sala
    const diff  = stats.elo - oldElo;
    if (Math.abs(diff) >= 1 || isDraw) {
        const color = isDraw ? 0x808080 : (diff >= 0 ? 0xA3FF00 : 0xFF4C4C);
        const icon  = isDraw ? "🤝" : (diff >= 0 ? "📈" : "📉");
        room.sendAnnouncement(
            `${icon} ${player.name}: ${diff >= 0 ? "+" : ""}${diff} ELO (Nivel ${stats.nivel})`,
            player.id, color, "italic"
        );
    }
}
```

---

## Ajustes

Los valores que más afectan el comportamiento:

| Parámetro | Valor actual | Efecto si lo subís |
| --------- | ------------ | ------------------ |
| `mu × 170` | multiplicador ELO | ELO más alto en general |
| `sigma × 45` | penalización por incertidumbre | Jugadores nuevos empiezan más bajo |
| `goals × 3.2` | peso de goles en impact | Más diferencia entre quien gola y quien no |
| `assists × 2.8` | peso de asistencias | Más valor al juego colectivo |
| Reducción derrota `0.40` | cap de protección | Más protección si perdiste jugando bien |

---

## Notas

El cálculo de TrueSkill corre en un servidor local separado porque la librería no está pensada para correr dentro del entorno de HaxBall. Si querés el servidor también lo puedo subir.

El sistema tiene limitaciones. Si jugás de arquero y tu equipo domina, el impact va a ser bajo aunque hayas jugado bien — el clean sheet ayuda pero no compensa del todo.

---

## Referencias

- Script base: [Wazarr94/haxball_bot_headless](https://github.com/Wazarr94/haxball_bot_headless/blob/master/HaxBot_public.js)
- TrueSkill: [microsoft.com/research/trueskill](https://www.microsoft.com/en-us/research/project/trueskill-ranking-system/)
- Librería: [npmjs.com/package/trueskill](https://www.npmjs.com/package/trueskill)
