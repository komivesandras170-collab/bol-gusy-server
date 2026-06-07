const express = require("express");
const http = require("http");
const cors = require("cors");
const { Server } = require("socket.io");

const app = express();
app.use(cors());
app.use(express.json());

app.get("/", (req, res) => {
  res.send("Bol Gusy multiplayer szerver fut.");
});

app.get("/health", (req, res) => {
  res.json({ ok: true, name: "bol-gusy-server" });
});

const server = http.createServer(app);

const io = new Server(server, {
  cors: {
    origin: "*",
    methods: ["GET", "POST"]
  },
  transports: ["websocket", "polling"]
});

const rooms = new Map();

function getRoom(roomCode) {
  const code = String(roomCode || "BOL123").toUpperCase().slice(0, 12);
  if (!rooms.has(code)) rooms.set(code, new Map());
  return { code, players: rooms.get(code) };
}

function publicPlayers(players) {
  return Array.from(players.values()).map(p => ({
    id: p.id,
    name: p.name,
    skin: p.skin,
    x: p.x || 0,
    y: p.y || 0,
    z: p.z || 0,
    vx: p.vx || 0,
    vy: p.vy || 0,
    eliminated: !!p.eliminated,
    slide: !!p.slide,
    punch: !!p.punch,
    t: p.t || Date.now()
  }));
}

io.on("connection", (socket) => {
  socket.data.room = null;
  socket.data.name = "Játékos";

  socket.on("joinRoom", (data = {}) => {
    const { code, players } = getRoom(data.room);
    const name = String(data.name || "Játékos").slice(0, 24);
    const skin = String(data.skin || "bolgusy_start_3d").slice(0, 40);

    if (socket.data.room && socket.data.room !== code) {
      socket.leave(socket.data.room);
      const old = rooms.get(socket.data.room);
      if (old) old.delete(socket.id);
    }

    socket.data.room = code;
    socket.data.name = name;
    socket.join(code);

    const player = {
      id: socket.id,
      name,
      skin,
      x: 700,
      y: 1300,
      z: 0,
      vx: 0,
      vy: 0,
      eliminated: false,
      slide: false,
      punch: false,
      t: Date.now()
    };

    players.set(socket.id, player);

    socket.emit("roomState", publicPlayers(players));
    socket.to(code).emit("playerJoined", player);
    socket.to(code).emit("roomState", publicPlayers(players));
  });

  socket.on("playerState", (data = {}) => {
    const code = socket.data.room || String(data.room || "").toUpperCase();
    if (!code || !rooms.has(code)) return;

    const players = rooms.get(code);
    const p = players.get(socket.id);
    if (!p) return;

    p.name = String(data.name || p.name || "Játékos").slice(0, 24);
    p.skin = String(data.skin || p.skin || "bolgusy_start_3d").slice(0, 40);
    p.x = Number(data.x) || 0;
    p.y = Number(data.y) || 0;
    p.z = Number(data.z) || 0;
    p.vx = Number(data.vx) || 0;
    p.vy = Number(data.vy) || 0;
    p.eliminated = !!data.eliminated;
    p.slide = !!data.slide;
    p.punch = !!data.punch;
    p.t = Date.now();

    socket.to(code).emit("playerUpdate", p);
  });

  socket.on("eliminated", (data = {}) => {
    const code = socket.data.room || String(data.room || "").toUpperCase();
    if (!code || !rooms.has(code)) return;

    const players = rooms.get(code);
    const p = players.get(socket.id);
    if (!p) return;

    p.eliminated = true;
    p.x = Number(data.x) || p.x || 0;
    p.y = Number(data.y) || p.y || 0;
    p.z = Number(data.z) || p.z || 0;
    p.t = Date.now();

    socket.to(code).emit("playerEliminated", p);
  });

  socket.on("disconnect", () => {
    const code = socket.data.room;
    if (!code || !rooms.has(code)) return;

    const players = rooms.get(code);
    players.delete(socket.id);

    socket.to(code).emit("playerLeft", socket.id);
    socket.to(code).emit("roomState", publicPlayers(players));

    if (players.size === 0) rooms.delete(code);
  });
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`Bol Gusy multiplayer szerver fut a ${PORT} porton`);
});
