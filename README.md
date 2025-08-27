import React, { useEffect, useRef, useState } from "react";
import { motion } from "framer-motion";
import { PiggyBank, Settings } from "lucide-react";

// Utility functions
function formatMoney(amount) {
  return `₹${amount.toFixed(2)}`;
}

// Dummy functions for seeds (replace with real provably fair logic)
function initialServerSeed() {
  return "server-seed-placeholder";
}
function initialClientSeed() {
  return "client-seed-placeholder";
}
async function drawCrashPoint() {
  // Random crash point between 1.0x and 10.0x
  const crash = 1 + Math.random() * 9;
  return { hash: Math.random().toString(36).slice(2), crash: Math.round(crash * 100) / 100 };
}

export default function LA100() {
  // Game settings / state
  const [serverSeed, setServerSeed] = useState(initialServerSeed());
  const [clientSeed, setClientSeed] = useState(initialClientSeed());
  const [nonce, setNonce] = useState(1);
  const [lambda, setLambda] = useState(0.6);
  const [houseEdge, setHouseEdge] = useState(0.01);
  const [commissionRate, setCommissionRate] = useState(0.03); // 3%

  const [bet, setBet] = useState(100);
  const [autoCashout, setAutoCashout] = useState(2.0);
  const [balance, setBalance] = useState(10000);
  const [houseBalance, setHouseBalance] = useState(0);

  const [round, setRound] = useState({
    status: "IDLE",
    startTime: 0,
    crashPoint: 1,
    hash: "",
  });

  const [multiplier, setMultiplier] = useState(1.0);
  const rafRef = useRef(0);
  const startTsRef = useRef(0);

  // Start a new round
  async function startRound() {
    if (balance < bet) return;

    const { hash, crash } = await drawCrashPoint();

    // commission deduction
    const commission = Math.round(bet * commissionRate * 100) / 100;
    const effectiveBet = bet - commission;

    setHouseBalance((h) => h + commission);
    setBalance((b) => b - bet);

    setRound({ status: "RUNNING", startTime: performance.now(), crashPoint: crash, hash });
    startTsRef.current = performance.now();
    setMultiplier(1.0);

    cancelAnimationFrame(rafRef.current);
    const animate = () => {
      const elapsed = (performance.now() - startTsRef.current) / 1000;
      const r = 0.9;
      const m = Math.max(1, Math.exp(r * elapsed));
      setMultiplier(Math.round(m * 100) / 100);
      if (m >= crash) {
        setMultiplier(crash);
        setRound((r0) => ({ ...r0, status: "CRASHED" }));
        return;
      }
      rafRef.current = requestAnimationFrame(animate);
    };
    rafRef.current = requestAnimationFrame(animate);
  }

  // Auto cashout
  useEffect(() => {
    if (round.status === "RUNNING" && multiplier >= autoCashout) {
      const commission = Math.round(bet * commissionRate * 100) / 100;
      const effectiveBet = bet - commission;
      const netWin = Math.round(effectiveBet * autoCashout * 100) / 100;

      setBalance((b) => b + netWin);
      setRound((r0) => ({ ...r0, status: "SETTLING" }));
    }
  }, [multiplier, autoCashout, bet, commissionRate, round.status]);

  return (
    <div className="min-h-screen w-full bg-gradient-to-br from-slate-950 via-slate-900 to-slate-800 text-slate-100 p-6">
      <div className="max-w-6xl mx-auto grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2">
          <div className="bg-slate-900/60 backdrop-blur rounded-2xl p-6 shadow-xl border border-slate-800">
            <div className="text-lg font-bold mb-4">LA100 Game</div>

            <div className="text-sm text-slate-400">Player Balance</div>
            <div className="text-2xl font-semibold">{formatMoney(balance)}</div>

            <div className="mt-2 flex items-center gap-2 text-sm text-green-400">
              <PiggyBank className="w-4 h-4"/> House Commission: {formatMoney(houseBalance)}
            </div>

            <div className="mt-4 grid grid-cols-2 gap-4">
              <div>
                <label className="text-xs text-slate-400">Bet Amount</label>
                <input type="number" value={bet} onChange={(e) => setBet(Number(e.target.value))}
                  className="mt-1 w-full bg-slate-800 border border-slate-700 rounded-xl px-3 py-2 outline-none"/>
              </div>
              <div>
                <label className="text-xs text-slate-400">Auto Cashout (x)</label>
                <input type="number" step="0.1" value={autoCashout} onChange={(e) => setAutoCashout(Number(e.target.value))}
                  className="mt-1 w-full bg-slate-800 border border-slate-700 rounded-xl px-3 py-2 outline-none"/>
              </div>
            </div>

            <div className="mt-4">
              <button onClick={startRound} className="bg-green-600 hover:bg-green-500 px-4 py-2 rounded-xl shadow">Start Round</button>
            </div>

            <div className="mt-6">
              <div className="text-sm text-slate-400">Round Status: {round.status}</div>
              <div className="text-xl font-semibold text-yellow-400">Multiplier: {multiplier}x</div>
              <div className="text-sm text-slate-500">Crash Point: {round.crashPoint}x</div>
            </div>

            <div className="mt-6 bg-slate-900 rounded-2xl p-4 border border-slate-800">
              <div className="flex items-center gap-2 text-slate-300"><Settings className="w-4 h-4"/> Engine</div>
              <div className="mt-3 grid grid-cols-2 gap-3">
                <div>
                  <label className="text-xs text-slate-400">Lambda</label>
                  <input type="number" step="0.01" min={0}
                    className="mt-1 w-full bg-slate-800 border border-slate-700 rounded-xl px-3 py-2 outline-none"
                    value={lambda}
                    onChange={(e) => setLambda(Math.max(0, Number(e.target.value)))} />
                </div>
                <div>
                  <label className="text-xs text-slate-400">House Edge (%)</label>
                  <input type="number" step="0.001" min={0}
                    className="mt-1 w-full bg-slate-800 border border-slate-700 rounded-xl px-3 py-2 outline-none"
                    value={houseEdge}
                    onChange={(e) => setHouseEdge(Math.max(0, Number(e.target.value)))} />
                </div>
                <div>
                  <label className="text-xs text-slate-400">Commission (%)</label>
                  <input type="number" step="0.001" min={0}
                    className="mt-1 w-full bg-slate-800 border border-slate-700 rounded-xl px-3 py-2 outline-none"
                    value={commissionRate}
                    onChange={(e) => setCommissionRate(Math.max(0, Number(e.target.value)))} />
                </div>
              </div>
            </div>

          </div>
        </div>
      </div>
    </div>
  );
}
