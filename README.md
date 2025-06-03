import { useState } from 'react';
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import * as XLSX from 'xlsx';
import { saveAs } from 'file-saver';

let playerIdCounter = 1;

export default function BadmintonApp() {
  const [players, setPlayers] = useState([]);
  const [name, setName] = useState("");
  const [matches, setMatches] = useState([]);
  const [history, setHistory] = useState([]);
  const [pastMatchups, setPastMatchups] = useState(new Set());

  const addPlayer = () => {
    if (name.trim() !== "") {
      setPlayers([...players, {
        id: playerIdCounter++,
        name,
        stats: { played: 0, won: 0, lost: 0, pointsFor: 0, pointsAgainst: 0 }
      }]);
      setName("");
    }
  };

  const shuffleArray = (array) => {
    return array
      .map(value => ({ value, sort: Math.random() }))
      .sort((a, b) => a.sort - b.sort)
      .map(({ value }) => value);
  };

  const pairKey = (team1, team2) => {
    const allIds = [...team1, ...team2].map(p => p.id).sort((a, b) => a - b);
    return allIds.join("-");
  };

  const generateMatches = () => {
    const shuffled = shuffleArray([...players]);
    const newMatches = [];
    const newMatchups = new Set(pastMatchups);

    for (let i = 0; i < shuffled.length - 1; i++) {
      for (let j = i + 1; j < shuffled.length; j++) {
        for (let k = 0; k < shuffled.length - 1; k++) {
          for (let l = k + 1; l < shuffled.length; l++) {
            const team1 = [shuffled[i], shuffled[j]];
            const team2 = [shuffled[k], shuffled[l]];
            const key = pairKey(team1, team2);

            if (newMatches.find(m =>
              m.some(t => t.some(p => team1.includes(p))) ||
              m.some(t => t.some(p => team2.includes(p)))
            )) continue;

            if (!newMatchups.has(key)) {
              newMatches.push([team1, team2]);
              newMatchups.add(key);
              i = j = k = l = shuffled.length; // Break all loops
            }
          }
        }
      }
    }

    setMatches(newMatches);
    setPastMatchups(newMatchups);
  };

  const reportResult = (matchIndex, winnerIndex, score1, score2) => {
    const match = matches[matchIndex];
    const winnerTeam = match[winnerIndex];
    const loserTeam = match[1 - winnerIndex];

    const updatedPlayers = players.map(player => {
      if (winnerTeam.some(p => p.id === player.id)) {
        return {
          ...player,
          stats: {
            ...player.stats,
            played: player.stats.played + 1,
            won: player.stats.won + 1,
            pointsFor: player.stats.pointsFor + parseInt(score1),
            pointsAgainst: player.stats.pointsAgainst + parseInt(score2)
          }
        };
      } else if (loserTeam.some(p => p.id === player.id)) {
        return {
          ...player,
          stats: {
            ...player.stats,
            played: player.stats.played + 1,
            lost: player.stats.lost + 1,
            pointsFor: player.stats.pointsFor + parseInt(score2),
            pointsAgainst: player.stats.pointsAgainst + parseInt(score1)
          }
        };
      }
      return player;
    });

    setPlayers(updatedPlayers);
    setHistory([...history, { match, winner: winnerTeam, score: [score1, score2] }]);
    setMatches(matches.filter((_, i) => i !== matchIndex));
  };

  const exportToExcel = () => {
    const matchData = history.map((h, idx) => ({
      Partida: idx + 1,
      Equipo1: h.match[0].map(p => p.name).join(" y "),
      Equipo2: h.match[1].map(p => p.name).join(" y "),
      Ganador: h.winner.map(p => p.name).join(" y "),
      "Puntos Equipo1": h.score[0],
      "Puntos Equipo2": h.score[1],
    }));

    const statsData = players.map(p => ({
      Jugador: p.name,
      Jugados: p.stats.played,
      Ganados: p.stats.won,
      Perdidos: p.stats.lost,
      "Puntos+": p.stats.pointsFor,
      "Puntos-": p.stats.pointsAgainst
    }));

    const wb = XLSX.utils.book_new();
    const matchSheet = XLSX.utils.json_to_sheet(matchData);
    const statsSheet = XLSX.utils.json_to_sheet(statsData);
    XLSX.utils.book_append_sheet(wb, matchSheet, "Partidas");
    XLSX.utils.book_append_sheet(wb, statsSheet, "Estadísticas");

    const wbout = XLSX.write(wb, { bookType: 'xlsx', type: 'array' });
    saveAs(new Blob([wbout], { type: "application/octet-stream" }), `badminton_${new Date().toISOString().slice(0,10)}.xlsx`);
  };

  return (
    <div className="p-6 space-y-6">
      <Card>
        <CardContent className="flex gap-2 items-center">
          <Input value={name} onChange={e => setName(e.target.value)} placeholder="Nombre del jugador" />
          <Button onClick={addPlayer}>Agregar</Button>
          <Button onClick={generateMatches}>Generar Partidas</Button>
          <Button onClick={exportToExcel}>Exportar Excel</Button>
        </CardContent>
      </Card>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        {matches.map((match, index) => (
          <Card key={index}>
            <CardContent className="space-y-2">
              <div className="font-bold">Partida {index + 1}</div>
              <div>Equipo 1: {match[0].map(p => p.name).join(" y ")}</div>
              <div>Equipo 2: {match[1].map(p => p.name).join(" y ")}</div>
              <div className="flex gap-2">
                <Button onClick={() => reportResult(index, 0, 21, 18)}>Ganó Equipo 1</Button>
                <Button onClick={() => reportResult(index, 1, 18, 21)}>Ganó Equipo 2</Button>
              </div>
            </CardContent>
          </Card>
        ))}
      </div>

      <Card>
        <CardContent>
          <h2 className="text-lg font-bold mb-2">Estadísticas</h2>
          <table className="w-full text-sm">
            <thead>
              <tr>
                <th className="text-left">Jugador</th>
                <th>Jugados</th>
                <th>Ganados</th>
                <th>Perdidos</th>
                <th>Puntos+</th>
                <th>Puntos-</th>
              </tr>
            </thead>
            <tbody>
              {players.map(p => (
                <tr key={p.id}>
                  <td>{p.name}</td>
                  <td className="text-center">{p.stats.played}</td>
                  <td className="text-center">{p.stats.won}</td>
                  <td className="text-center">{p.stats.lost}</td>
                  <td className="text-center">{p.stats.pointsFor}</td>
                  <td className="text-center">{p.stats.pointsAgainst}</td>
                </tr>
              ))}
            </tbody>
          </table>
        </CardContent>
      </Card>
    </div>
  );
}
