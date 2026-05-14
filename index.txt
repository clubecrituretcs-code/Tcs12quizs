import { useState, useEffect, useRef } from "react";

// ─── Palette & Design ─────────────────────────────────────────────
// Deep academic midnight + electric lime accents — "smart classroom" vibe

const COLORS = ["#FF6B6B", "#4ECDC4", "#FFE66D", "#A29BFE"];
const COLOR_NAMES = ["Rouge", "Cyan", "Jaune", "Violet"];

// ─── Utility ──────────────────────────────────────────────────────
const genCode = () => Math.random().toString(36).substring(2, 7).toUpperCase();

// ─── Main App ─────────────────────────────────────────────────────
export default function TCS12Quiz() {
  // "screen" : landing | admin_setup | admin_lobby | admin_game | admin_results
  //          : join | player_wait | player_game | player_results
  const [screen, setScreen] = useState("landing");

  // Shared state (simulated in-memory "server")
  const [rooms, setRooms] = useState({}); // { [code]: roomObj }
  const [currentRoom, setCurrentRoom] = useState(null); // code
  const [playerName, setPlayerName] = useState("");
  const [joinCode, setJoinCode] = useState("");
  const [joinError, setJoinError] = useState("");

  // Admin: quiz builder
  const [quizTitle, setQuizTitle] = useState("");
  const [questions, setQuestions] = useState([
    { text: "", answers: ["", "", "", ""], correct: 0, time: 20 }
  ]);

  // Game state
  const [currentQ, setCurrentQ] = useState(0);
  const [timeLeft, setTimeLeft] = useState(0);
  const [playerAnswer, setPlayerAnswer] = useState(null);
  const [showAnswer, setShowAnswer] = useState(false);
  const [scores, setScores] = useState({}); // { playerName: score }
  const [playerAnswers, setPlayerAnswers] = useState({}); // { playerName: answerIndex }
  const timerRef = useRef(null);

  // ── Derived ──
  const room = currentRoom ? rooms[currentRoom] : null;

  // ── Admin creates room ──
  const createRoom = () => {
    if (!quizTitle.trim()) return;
    const validQs = questions.filter(q => q.text.trim() && q.answers.some(a => a.trim()));
    if (validQs.length === 0) return;
    const code = genCode();
    const newRoom = {
      code,
      title: quizTitle,
      questions: validQs,
      players: [],
      status: "lobby", // lobby | question | answer | finished
      currentQ: 0,
    };
    setRooms(prev => ({ ...prev, [code]: newRoom }));
    setCurrentRoom(code);
    setCurrentQ(0);
    setScores({});
    setScreen("admin_lobby");
  };

  // ── Player joins ──
  const joinRoom = () => {
    const code = joinCode.trim().toUpperCase();
    if (!rooms[code]) { setJoinError("Code invalide ou room introuvable."); return; }
    if (rooms[code].status !== "lobby") { setJoinError("Le quiz a déjà commencé."); return; }
    if (!playerName.trim()) { setJoinError("Entre ton prénom!"); return; }
    setRooms(prev => {
      const r = { ...prev[code] };
      if (!r.players.includes(playerName)) r.players = [...r.players, playerName];
      return { ...prev, [code]: r };
    });
    setScores(prev => ({ ...prev, [playerName]: 0 }));
    setCurrentRoom(code);
    setJoinError("");
    setScreen("player_wait");
  };

  // ── Admin starts game ──
  const startGame = () => {
    setRooms(prev => ({ ...prev, [currentRoom]: { ...prev[currentRoom], status: "question" } }));
    setCurrentQ(0);
    setShowAnswer(false);
    setPlayerAnswers({});
    startTimer(rooms[currentRoom].questions[0].time);
    setScreen("admin_game");
  };

  // ── Timer ──
  const startTimer = (seconds) => {
    clearInterval(timerRef.current);
    setTimeLeft(seconds);
    timerRef.current = setInterval(() => {
      setTimeLeft(prev => {
        if (prev <= 1) {
          clearInterval(timerRef.current);
          setShowAnswer(true);
          return 0;
        }
        return prev - 1;
      });
    }, 1000);
  };

  // ── Admin next question ──
  const nextQuestion = () => {
    const r = rooms[currentRoom];
    const nextIdx = currentQ + 1;
    if (nextIdx >= r.questions.length) {
      setRooms(prev => ({ ...prev, [currentRoom]: { ...prev[currentRoom], status: "finished" } }));
      setScreen("admin_results");
      return;
    }
    setCurrentQ(nextIdx);
    setShowAnswer(false);
    setPlayerAnswers({});
    startTimer(r.questions[nextIdx].time);
  };

  // ── Player submits answer ──
  const submitAnswer = (idx) => {
    if (playerAnswer !== null || showAnswer) return;
    setPlayerAnswer(idx);
    const r = rooms[currentRoom];
    const correct = r.questions[currentQ].correct;
    const pts = idx === correct ? Math.max(100, timeLeft * 10) : 0;
    setScores(prev => ({ ...prev, [playerName]: (prev[playerName] || 0) + pts }));
    setPlayerAnswers(prev => ({ ...prev, [playerName]: idx }));
  };

  // ── Watch for room status changes (player side) ──
  useEffect(() => {
    if (!currentRoom || !room) return;
    if (screen === "player_wait" && room.status === "question") {
      setPlayerAnswer(null);
      setShowAnswer(false);
      setCurrentQ(room.currentQ || 0);
      setScreen("player_game");
    }
    if (screen === "player_game" && room.status === "finished") {
      setScreen("player_results");
    }
  }, [room?.status, screen]);

  // Sync currentQ to room for players
  useEffect(() => {
    if (screen === "admin_game") {
      setRooms(prev => ({ ...prev, [currentRoom]: { ...prev[currentRoom], currentQ } }));
      setPlayerAnswer(null);
    }
    if (screen === "player_game" && room) {
      if (room.currentQ !== undefined && room.currentQ !== currentQ) {
        setCurrentQ(room.currentQ);
        setPlayerAnswer(null);
        setShowAnswer(false);
        startTimer(room.questions[room.currentQ]?.time || 20);
      }
    }
  }, [currentQ, screen]);

  // Sorted leaderboard
  const leaderboard = Object.entries(scores)
    .sort((a, b) => b[1] - a[1])
    .map(([name, score], i) => ({ rank: i + 1, name, score }));

  // ── RENDER ──────────────────────────────────────────────────────
  return (
    <div style={styles.root}>
      <style>{css}</style>

      {/* ── LANDING ── */}
      {screen === "landing" && (
        <div style={styles.center}>
          <div style={styles.logo}>
            <span style={styles.logoT}>TCS</span>
            <span style={styles.logo12}>12</span>
            <span style={styles.logoQ}>Quiz</span>
          </div>
          <p style={styles.tagline}>La plateforme de quiz interactive pour la classe</p>
          <div style={styles.btnRow}>
            <button className="btn-primary" onClick={() => setScreen("admin_setup")}>
              🎛️ Créer un Quiz
            </button>
            <button className="btn-secondary" onClick={() => setScreen("join")}>
              🎮 Rejoindre
            </button>
          </div>
        </div>
      )}

      {/* ── ADMIN SETUP ── */}
      {screen === "admin_setup" && (
        <div style={styles.panel}>
          <div style={styles.panelHeader}>
            <button className="btn-back" onClick={() => setScreen("landing")}>← Retour</button>
            <h2 style={styles.panelTitle}>🎛️ Créer un Quiz</h2>
          </div>

          <input
            style={styles.input}
            placeholder="Titre du quiz (ex: Biologie Chapitre 3)"
            value={quizTitle}
            onChange={e => setQuizTitle(e.target.value)}
          />

          <div style={styles.qList}>
            {questions.map((q, qi) => (
              <div key={qi} style={styles.qCard}>
                <div style={styles.qCardHead}>
                  <span style={styles.qNum}>Q{qi + 1}</span>
                  <input
                    style={{ ...styles.input, flex: 1, marginBottom: 0 }}
                    placeholder="Question..."
                    value={q.text}
                    onChange={e => {
                      const copy = [...questions];
                      copy[qi] = { ...copy[qi], text: e.target.value };
                      setQuestions(copy);
                    }}
                  />
                  <div style={styles.timerPill}>
                    ⏱ <input
                      type="number" min={5} max={120}
                      style={styles.timerInput}
                      value={q.time}
                      onChange={e => {
                        const copy = [...questions];
                        copy[qi] = { ...copy[qi], time: parseInt(e.target.value) || 20 };
                        setQuestions(copy);
                      }}
                    />s
                  </div>
                  {questions.length > 1 && (
                    <button className="btn-danger-sm" onClick={() => setQuestions(questions.filter((_, i) => i !== qi))}>✕</button>
                  )}
                </div>

                <div style={styles.answerGrid}>
                  {q.answers.map((ans, ai) => (
                    <div key={ai} style={{ ...styles.answerItem, borderColor: q.correct === ai ? "#A8FF3E" : "#2a2a3e" }}>
                      <div
                        style={{ ...styles.colorDot, background: COLORS[ai] }}
                        onClick={() => {
                          const copy = [...questions];
                          copy[qi] = { ...copy[qi], correct: ai };
                          setQuestions(copy);
                        }}
                        title="Définir comme bonne réponse"
                      >
                        {q.correct === ai ? "✓" : ai + 1}
                      </div>
                      <input
                        style={styles.ansInput}
                        placeholder={`Réponse ${ai + 1}`}
                        value={ans}
                        onChange={e => {
                          const copy = [...questions];
                          copy[qi].answers[ai] = e.target.value;
                          setQuestions(copy);
                        }}
                      />
                    </div>
                  ))}
                </div>
              </div>
            ))}
          </div>

          <div style={styles.btnRow}>
            <button className="btn-secondary" onClick={() =>
              setQuestions([...questions, { text: "", answers: ["", "", "", ""], correct: 0, time: 20 }])
            }>+ Ajouter Question</button>
            <button className="btn-primary" onClick={createRoom}>🚀 Créer la Room</button>
          </div>
        </div>
      )}

      {/* ── ADMIN LOBBY ── */}
      {screen === "admin_lobby" && room && (
        <div style={styles.center}>
          <div style={styles.codeBox}>
            <p style={styles.codeLabel}>Code de la Room</p>
            <div style={styles.codeBig}>{room.code}</div>
            <p style={styles.codeHint}>Partagez ce code avec vos élèves</p>
          </div>
          <div style={styles.playerList}>
            <p style={styles.playerCount}>{room.players.length} joueur(s) connecté(s)</p>
            <div style={styles.playerChips}>
              {room.players.map(p => (
                <span key={p} style={styles.chip}>{p}</span>
              ))}
              {room.players.length === 0 && <span style={styles.waiting}>En attente de joueurs…</span>}
            </div>
          </div>
          <button
            className="btn-primary"
            disabled={room.players.length === 0}
            onClick={startGame}
            style={{ opacity: room.players.length === 0 ? 0.4 : 1 }}
          >▶ Démarrer le Quiz ({room.questions.length} questions)</button>
        </div>
      )}

      {/* ── ADMIN GAME ── */}
      {screen === "admin_game" && room && (
        <div style={styles.gameWrap}>
          <div style={styles.gameHeader}>
            <span style={styles.qProgress}>Q{currentQ + 1} / {room.questions.length}</span>
            <span style={styles.gameTitle}>{room.title}</span>
            <div style={{ ...styles.timerCircle, borderColor: timeLeft < 5 ? "#FF6B6B" : "#A8FF3E" }}>
              {timeLeft}
            </div>
          </div>

          <div style={styles.questionBox}>
            <p style={styles.questionText}>{room.questions[currentQ].text}</p>
          </div>

          <div style={styles.answersGrid}>
            {room.questions[currentQ].answers.map((ans, ai) => (
              <div key={ai} style={{ ...styles.answerCard, background: COLORS[ai] + "22", borderColor: COLORS[ai] }}>
                <span style={{ ...styles.ansColorDot, background: COLORS[ai] }}>{COLOR_NAMES[ai]}</span>
                <span style={styles.ansText}>{ans}</span>
                {showAnswer && ai === room.questions[currentQ].correct && (
                  <span style={styles.correctBadge}>✓ Bonne réponse</span>
                )}
              </div>
            ))}
          </div>

          {showAnswer && (
            <div style={styles.answerReveal}>
              <div style={styles.answerStats}>
                {room.players.map(p => (
                  <div key={p} style={styles.playerStat}>
                    <span>{p}</span>
                    <span style={{ color: playerAnswers[p] === room.questions[currentQ].correct ? "#A8FF3E" : "#FF6B6B" }}>
                      {playerAnswers[p] !== undefined
                        ? (playerAnswers[p] === room.questions[currentQ].correct ? `+${Math.max(100, timeLeft * 10)}` : "✕")
                        : "—"}
                    </span>
                  </div>
                ))}
              </div>
              <button className="btn-primary" onClick={nextQuestion}>
                {currentQ + 1 < room.questions.length ? "Question Suivante →" : "🏆 Voir Résultats"}
              </button>
            </div>
          )}
        </div>
      )}

      {/* ── ADMIN RESULTS ── */}
      {screen === "admin_results" && (
        <div style={styles.center}>
          <h2 style={styles.panelTitle}>🏆 Résultats Finaux</h2>
          <p style={styles.tagline}>{quizTitle}</p>
          <div style={styles.leaderboard}>
            {leaderboard.map((e, i) => (
              <div key={e.name} style={{ ...styles.lbRow, background: i === 0 ? "#A8FF3E22" : i === 1 ? "#FFE66D11" : "#ffffff08" }}>
                <span style={styles.lbRank}>{["🥇", "🥈", "🥉"][i] || `#${e.rank}`}</span>
                <span style={styles.lbName}>{e.name}</span>
                <span style={styles.lbScore}>{e.score} pts</span>
              </div>
            ))}
          </div>
          <button className="btn-secondary" onClick={() => {
            setScreen("landing");
            setRooms({});
            setCurrentRoom(null);
            setQuestions([{ text: "", answers: ["", "", "", ""], correct: 0, time: 20 }]);
            setQuizTitle("");
          }}>🔁 Nouveau Quiz</button>
        </div>
      )}

      {/* ── JOIN ── */}
      {screen === "join" && (
        <div style={styles.center}>
          <div style={styles.logo}>
            <span style={styles.logoT}>TCS</span>
            <span style={styles.logo12}>12</span>
            <span style={styles.logoQ}>Quiz</span>
          </div>
          <div style={styles.joinBox}>
            <input
              style={styles.inputLarge}
              placeholder="Code de la Room"
              value={joinCode}
              onChange={e => setJoinCode(e.target.value.toUpperCase())}
              maxLength={5}
            />
            <input
              style={styles.inputLarge}
              placeholder="Ton prénom"
              value={playerName}
              onChange={e => setPlayerName(e.target.value)}
            />
            {joinError && <p style={styles.error}>{joinError}</p>}
            <button className="btn-primary" onClick={joinRoom}>Rejoindre →</button>
            <button className="btn-back" onClick={() => setScreen("landing")}>← Retour</button>
          </div>
        </div>
      )}

      {/* ── PLAYER WAIT ── */}
      {screen === "player_wait" && room && (
        <div style={styles.center}>
          <div style={styles.waitAnim}>
            <div style={styles.waitDot} className="dot1" />
            <div style={styles.waitDot} className="dot2" />
            <div style={styles.waitDot} className="dot3" />
          </div>
          <p style={styles.waitName}>👋 Salut, <strong>{playerName}</strong>!</p>
          <p style={styles.tagline}>En attente du début du quiz…</p>
          <div style={styles.codeBoxSmall}>{room.code}</div>
          <p style={styles.codeHint}>{room.players.length} joueur(s) dans la room</p>
          {/* Simulate start for demo */}
          <button className="btn-secondary" style={{ marginTop: 20, fontSize: 12 }}
            onClick={() => {
              setRooms(prev => ({ ...prev, [currentRoom]: { ...prev[currentRoom], status: "question", currentQ: 0 } }));
              setPlayerAnswer(null);
              setShowAnswer(false);
              setCurrentQ(0);
              startTimer(room.questions[0].time);
              setScreen("player_game");
            }}>
            ▶ Simuler démarrage (démo)
          </button>
        </div>
      )}

      {/* ── PLAYER GAME ── */}
      {screen === "player_game" && room && (
        <div style={styles.gameWrap}>
          <div style={styles.gameHeader}>
            <span style={styles.qProgress}>Q{currentQ + 1} / {room.questions.length}</span>
            <span style={styles.gameTitle}>{playerName}</span>
            <div style={{ ...styles.timerCircle, borderColor: timeLeft < 5 ? "#FF6B6B" : "#A8FF3E" }}>
              {timeLeft}
            </div>
          </div>

          <div style={styles.questionBox}>
            <p style={styles.questionText}>{room.questions[currentQ]?.text}</p>
          </div>

          <div style={styles.answersGrid}>
            {room.questions[currentQ]?.answers.map((ans, ai) => (
              <button
                key={ai}
                className="answer-btn"
                style={{
                  background: playerAnswer === ai
                    ? COLORS[ai]
                    : (playerAnswer !== null ? COLORS[ai] + "33" : COLORS[ai] + "22"),
                  borderColor: COLORS[ai],
                  opacity: playerAnswer !== null && playerAnswer !== ai ? 0.5 : 1,
                  cursor: playerAnswer !== null ? "default" : "pointer",
                  transform: playerAnswer === ai ? "scale(1.03)" : "scale(1)",
                }}
                onClick={() => submitAnswer(ai)}
                disabled={playerAnswer !== null}
              >
                <span style={{ ...styles.ansColorDot, background: COLORS[ai] }}>{COLOR_NAMES[ai]}</span>
                <span style={styles.ansText}>{ans}</span>
              </button>
            ))}
          </div>

          {playerAnswer !== null && (
            <div style={styles.submitted}>
              ✅ Réponse envoyée ! En attente des autres…
            </div>
          )}

          {/* Player: simulate next question for demo */}
          {playerAnswer !== null && (
            <button className="btn-secondary" style={{ marginTop: 10, fontSize: 12 }}
              onClick={() => {
                const r = rooms[currentRoom];
                const nextIdx = currentQ + 1;
                if (nextIdx >= r.questions.length) {
                  setScreen("player_results");
                  return;
                }
                setCurrentQ(nextIdx);
                setPlayerAnswer(null);
                setShowAnswer(false);
                startTimer(r.questions[nextIdx].time);
                setRooms(prev => ({ ...prev, [currentRoom]: { ...prev[currentRoom], currentQ: nextIdx } }));
              }}>
              ⏭ Prochaine question (démo)
            </button>
          )}
        </div>
      )}

      {/* ── PLAYER RESULTS ── */}
      {screen === "player_results" && (
        <div style={styles.center}>
          <h2 style={styles.panelTitle}>🏁 Quiz Terminé!</h2>
          <div style={styles.myScore}>
            <p style={styles.codeLabel}>Ton score</p>
            <div style={styles.scoreBig}>{scores[playerName] || 0}</div>
            <p style={styles.codeHint}>points</p>
          </div>
          <div style={styles.leaderboard}>
            {leaderboard.map((e, i) => (
              <div key={e.name} style={{ ...styles.lbRow, background: e.name === playerName ? "#A8FF3E22" : "#ffffff08" }}>
                <span style={styles.lbRank}>{["🥇", "🥈", "🥉"][i] || `#${e.rank}`}</span>
                <span style={styles.lbName}>{e.name}</span>
                <span style={styles.lbScore}>{e.score} pts</span>
              </div>
            ))}
          </div>
          <button className="btn-secondary" onClick={() => setScreen("landing")}>🔁 Accueil</button>
        </div>
      )}
    </div>
  );
}

// ─── STYLES ──────────────────────────────────────────────────────
const styles = {
  root: {
    minHeight: "100vh",
    background: "#0b0b14",
    color: "#e8e8f0",
    fontFamily: "'Syne', 'Space Mono', monospace",
    display: "flex",
    flexDirection: "column",
    alignItems: "center",
    justifyContent: "flex-start",
    padding: "20px 16px",
    boxSizing: "border-box",
  },
  center: {
    display: "flex",
    flexDirection: "column",
    alignItems: "center",
    justifyContent: "center",
    minHeight: "80vh",
    width: "100%",
    maxWidth: 480,
    gap: 20,
  },
  logo: { display: "flex", alignItems: "baseline", gap: 4, marginBottom: 4 },
  logoT: { fontSize: 48, fontWeight: 900, color: "#e8e8f0", letterSpacing: -2 },
  logo12: { fontSize: 48, fontWeight: 900, color: "#A8FF3E", letterSpacing: -2 },
  logoQ: { fontSize: 48, fontWeight: 900, color: "#4ECDC4", letterSpacing: -2 },
  tagline: { color: "#8888a8", fontSize: 14, textAlign: "center", marginTop: -10 },
  btnRow: { display: "flex", gap: 12, flexWrap: "wrap", justifyContent: "center" },

  panel: { width: "100%", maxWidth: 560, display: "flex", flexDirection: "column", gap: 14 },
  panelHeader: { display: "flex", alignItems: "center", gap: 12 },
  panelTitle: { fontSize: 22, fontWeight: 700, color: "#A8FF3E", margin: 0 },

  input: {
    background: "#14141f",
    border: "1px solid #2a2a3e",
    borderRadius: 10,
    color: "#e8e8f0",
    padding: "10px 14px",
    fontSize: 14,
    width: "100%",
    boxSizing: "border-box",
    outline: "none",
    fontFamily: "inherit",
    marginBottom: 6,
  },
  inputLarge: {
    background: "#14141f",
    border: "2px solid #2a2a3e",
    borderRadius: 12,
    color: "#e8e8f0",
    padding: "14px 18px",
    fontSize: 18,
    width: "100%",
    boxSizing: "border-box",
    outline: "none",
    fontFamily: "inherit",
    textAlign: "center",
    letterSpacing: 4,
    fontWeight: 700,
  },

  qList: { display: "flex", flexDirection: "column", gap: 12, maxHeight: "55vh", overflowY: "auto", paddingRight: 4 },
  qCard: { background: "#14141f", borderRadius: 14, padding: 14, border: "1px solid #2a2a3e" },
  qCardHead: { display: "flex", alignItems: "center", gap: 8, marginBottom: 10 },
  qNum: { background: "#A8FF3E", color: "#0b0b14", borderRadius: 6, padding: "2px 8px", fontWeight: 800, fontSize: 13, flexShrink: 0 },
  timerPill: { display: "flex", alignItems: "center", gap: 4, background: "#1e1e2f", borderRadius: 8, padding: "4px 10px", fontSize: 13, color: "#8888a8", flexShrink: 0 },
  timerInput: { background: "transparent", border: "none", color: "#e8e8f0", width: 32, fontSize: 13, outline: "none", fontFamily: "inherit", textAlign: "center" },

  answerGrid: { display: "grid", gridTemplateColumns: "1fr 1fr", gap: 8 },
  answerItem: { display: "flex", alignItems: "center", gap: 8, background: "#0b0b14", borderRadius: 8, padding: "6px 10px", border: "2px solid #2a2a3e", cursor: "pointer" },
  colorDot: { width: 26, height: 26, borderRadius: "50%", display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 800, fontSize: 12, color: "#0b0b14", cursor: "pointer", flexShrink: 0 },
  ansInput: { background: "transparent", border: "none", color: "#e8e8f0", fontSize: 13, outline: "none", fontFamily: "inherit", width: "100%" },

  // Code box
  codeBox: { textAlign: "center", background: "#14141f", border: "2px solid #A8FF3E", borderRadius: 20, padding: "24px 40px" },
  codeBoxSmall: { background: "#14141f", border: "2px solid #4ECDC4", borderRadius: 12, padding: "10px 24px", fontSize: 28, fontWeight: 900, letterSpacing: 6, color: "#4ECDC4" },
  codeLabel: { color: "#8888a8", fontSize: 13, marginBottom: 6, margin: 0 },
  codeBig: { fontSize: 56, fontWeight: 900, letterSpacing: 10, color: "#A8FF3E", lineHeight: 1.1 },
  codeHint: { color: "#8888a8", fontSize: 13, marginTop: 6, margin: 0 },

  playerList: { textAlign: "center", width: "100%" },
  playerCount: { color: "#8888a8", fontSize: 14, marginBottom: 10 },
  playerChips: { display: "flex", flexWrap: "wrap", gap: 8, justifyContent: "center" },
  chip: { background: "#4ECDC422", border: "1px solid #4ECDC4", borderRadius: 20, padding: "4px 14px", fontSize: 14, color: "#4ECDC4" },
  waiting: { color: "#4a4a6a", fontSize: 14, fontStyle: "italic" },

  // Game
  gameWrap: { width: "100%", maxWidth: 640, display: "flex", flexDirection: "column", gap: 14, paddingTop: 10 },
  gameHeader: { display: "flex", alignItems: "center", justifyContent: "space-between", background: "#14141f", borderRadius: 14, padding: "10px 16px" },
  qProgress: { background: "#A8FF3E", color: "#0b0b14", borderRadius: 8, padding: "4px 12px", fontWeight: 800, fontSize: 13 },
  gameTitle: { fontSize: 14, color: "#8888a8", fontWeight: 600 },
  timerCircle: { width: 46, height: 46, borderRadius: "50%", border: "3px solid", display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 900, fontSize: 18, transition: "border-color 0.3s" },

  questionBox: { background: "#14141f", borderRadius: 16, padding: "20px 24px", minHeight: 80, display: "flex", alignItems: "center", justifyContent: "center", textAlign: "center" },
  questionText: { fontSize: 20, fontWeight: 700, color: "#e8e8f0", margin: 0, lineHeight: 1.4 },

  answersGrid: { display: "grid", gridTemplateColumns: "1fr 1fr", gap: 12 },
  answerCard: { borderRadius: 14, padding: "14px 16px", border: "2px solid", display: "flex", flexDirection: "column", gap: 6, transition: "all 0.2s" },
  ansColorDot: { borderRadius: 6, padding: "2px 10px", fontSize: 12, fontWeight: 800, color: "#0b0b14", display: "inline-block", alignSelf: "flex-start" },
  ansText: { fontSize: 15, fontWeight: 600, color: "#e8e8f0" },
  correctBadge: { background: "#A8FF3E", color: "#0b0b14", borderRadius: 6, padding: "2px 8px", fontSize: 12, fontWeight: 800, alignSelf: "flex-start" },

  answerReveal: { background: "#14141f", borderRadius: 16, padding: 16, display: "flex", flexDirection: "column", gap: 12, alignItems: "center" },
  answerStats: { width: "100%", display: "flex", flexDirection: "column", gap: 6 },
  playerStat: { display: "flex", justifyContent: "space-between", padding: "6px 12px", background: "#0b0b14", borderRadius: 8, fontSize: 14 },

  submitted: { textAlign: "center", color: "#A8FF3E", fontSize: 15, fontWeight: 600, padding: 12, background: "#A8FF3E11", borderRadius: 10 },

  // Wait
  waitAnim: { display: "flex", gap: 10, marginBottom: 10 },
  waitDot: { width: 14, height: 14, borderRadius: "50%", background: "#A8FF3E" },
  waitName: { fontSize: 22, fontWeight: 700, textAlign: "center" },

  // Results
  myScore: { textAlign: "center", background: "#A8FF3E11", border: "2px solid #A8FF3E", borderRadius: 20, padding: "20px 40px" },
  scoreBig: { fontSize: 56, fontWeight: 900, color: "#A8FF3E" },
  leaderboard: { width: "100%", maxWidth: 420, display: "flex", flexDirection: "column", gap: 8 },
  lbRow: { display: "flex", alignItems: "center", gap: 12, borderRadius: 12, padding: "10px 16px" },
  lbRank: { fontSize: 22, width: 32, textAlign: "center" },
  lbName: { flex: 1, fontWeight: 600, fontSize: 15 },
  lbScore: { color: "#A8FF3E", fontWeight: 800, fontSize: 16 },

  // Join
  joinBox: { width: "100%", maxWidth: 340, display: "flex", flexDirection: "column", gap: 12, alignItems: "stretch" },
  error: { color: "#FF6B6B", fontSize: 13, textAlign: "center", margin: 0 },
};

const css = `
  @import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;700;800;900&display=swap');

  * { box-sizing: border-box; }

  .btn-primary {
    background: #A8FF3E;
    color: #0b0b14;
    border: none;
    border-radius: 12px;
    padding: 12px 28px;
    font-size: 15px;
    font-weight: 800;
    cursor: pointer;
    font-family: 'Syne', monospace;
    transition: transform 0.15s, box-shadow 0.15s;
    letter-spacing: 0.5px;
  }
  .btn-primary:hover:not(:disabled) { transform: translateY(-2px); box-shadow: 0 6px 20px #A8FF3E44; }

  .btn-secondary {
    background: transparent;
    color: #4ECDC4;
    border: 2px solid #4ECDC4;
    border-radius: 12px;
    padding: 12px 28px;
    font-size: 15px;
    font-weight: 700;
    cursor: pointer;
    font-family: 'Syne', monospace;
    transition: background 0.15s;
  }
  .btn-secondary:hover { background: #4ECDC422; }

  .btn-back {
    background: transparent;
    color: #8888a8;
    border: none;
    padding: 6px 0;
    font-size: 14px;
    cursor: pointer;
    font-family: 'Syne', monospace;
  }
  .btn-back:hover { color: #e8e8f0; }

  .btn-danger-sm {
    background: #FF6B6B22;
    color: #FF6B6B;
    border: 1px solid #FF6B6B44;
    border-radius: 6px;
    padding: 4px 8px;
    font-size: 12px;
    cursor: pointer;
    flex-shrink: 0;
  }

  .answer-btn {
    border-radius: 14px;
    padding: 14px 16px;
    border: 2px solid;
    display: flex;
    flex-direction: column;
    gap: 6px;
    transition: all 0.2s;
    cursor: pointer;
    text-align: left;
    font-family: 'Syne', monospace;
  }
  .answer-btn:hover:not(:disabled) { transform: scale(1.02); }

  .dot1, .dot2, .dot3 {
    animation: bounce 1.2s infinite;
  }
  .dot2 { animation-delay: 0.2s; background: #4ECDC4 !important; }
  .dot3 { animation-delay: 0.4s; background: #FFE66D !important; }

  @keyframes bounce {
    0%, 80%, 100% { transform: scale(0.6); opacity: 0.5; }
    40% { transform: scale(1.2); opacity: 1; }
  }

  ::-webkit-scrollbar { width: 4px; }
  ::-webkit-scrollbar-track { background: #0b0b14; }
  ::-webkit-scrollbar-thumb { background: #2a2a3e; border-radius: 4px; }
`;
