import React, { useState, useRef, useEffect } from 'react';
import { Undo2, RotateCcw, Users, Target, Trophy, Play, Volume2, VolumeX } from 'lucide-react';

// 定義靶面各區塊的分數與外觀 (半徑為累加)。加入 multFill 供 2 倍分數區塊上色
const RINGS = [
  { points: 50, radius: 40, fill: '#ef4444', textFill: '#ffffff', textY: 300, multFill: null },
  { points: 20, radius: 90, fill: '#f59e0b', textFill: '#000000', textY: 235, multFill: '#dc2626' }, // 黃圈的兩倍區: 紅
  { points: 15, radius: 140, fill: '#3b82f6', textFill: '#ffffff', textY: 185, multFill: '#f97316' }, // 藍圈的兩倍區: 橘
  { points: 10, radius: 190, fill: '#22c55e', textFill: '#ffffff', textY: 135, multFill: '#dc2626' }, // 綠圈的兩倍區: 紅
  { points: 5, radius: 240, fill: '#ffffff', textFill: '#000000', textY: 85, multFill: '#16a34a' },  // 白圈的兩倍區: 綠
  { points: 1, radius: 290, fill: '#1f2937', textFill: '#ffffff', textY: 35, multFill: '#dc2626' }   // 黑圈的兩倍區: 紅
];

// --- Web Audio API 音效合成系統 ---
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();

const playSound = (type) => {
  if (audioCtx.state === 'suspended') {
    audioCtx.resume();
  }

  const oscillator = audioCtx.createOscillator();
  const gainNode = audioCtx.createGain();
  
  oscillator.connect(gainNode);
  gainNode.connect(audioCtx.destination);

  const now = audioCtx.currentTime;

  switch (type) {
    case 'bullseye': // 50分 - 高亢慶祝音
      oscillator.type = 'triangle';
      oscillator.frequency.setValueAtTime(523.25, now);
      oscillator.frequency.exponentialRampToValueAtTime(1046.50, now + 0.1);
      gainNode.gain.setValueAtTime(0, now);
      gainNode.gain.linearRampToValueAtTime(0.5, now + 0.05);
      gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.5);
      oscillator.start(now);
      oscillator.stop(now + 0.5);
      break;

    case 'double': // 兩倍分數音效 (專屬清脆金屬高音)
      oscillator.type = 'sine';
      oscillator.frequency.setValueAtTime(1046.50, now); // C6
      oscillator.frequency.exponentialRampToValueAtTime(2093, now + 0.1); // C7
      gainNode.gain.setValueAtTime(0, now);
      gainNode.gain.linearRampToValueAtTime(0.5, now + 0.02);
      gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.4);
      oscillator.start(now);
      oscillator.stop(now + 0.4);
      break;

    case 'high': // 20, 15分 - 清脆金屬聲
      oscillator.type = 'sine';
      oscillator.frequency.setValueAtTime(880, now); 
      gainNode.gain.setValueAtTime(0, now);
      gainNode.gain.linearRampToValueAtTime(0.4, now + 0.02);
      gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.3);
      oscillator.start(now);
      oscillator.stop(now + 0.3);
      break;

    case 'mid': // 10, 5, 1分 - 沉穩木板聲
      oscillator.type = 'square';
      oscillator.frequency.setValueAtTime(150, now);
      oscillator.frequency.exponentialRampToValueAtTime(50, now + 0.1);
      gainNode.gain.setValueAtTime(0, now);
      gainNode.gain.linearRampToValueAtTime(0.3, now + 0.01);
      gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.15);
      oscillator.start(now);
      oscillator.stop(now + 0.15);
      break;

    case 'miss': // 0分脫靶 - 低沉失誤
      oscillator.type = 'sawtooth';
      oscillator.frequency.setValueAtTime(100, now);
      oscillator.frequency.linearRampToValueAtTime(50, now + 0.3);
      gainNode.gain.setValueAtTime(0.3, now);
      gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.3);
      oscillator.start(now);
      oscillator.stop(now + 0.3);
      break;
      
    case 'bust': // 爆鏢提示音
      oscillator.type = 'triangle';
      oscillator.frequency.setValueAtTime(300, now);
      oscillator.frequency.exponentialRampToValueAtTime(100, now + 0.5);
      gainNode.gain.setValueAtTime(0.5, now);
      gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.8);
      oscillator.start(now);
      oscillator.stop(now + 0.8);
      break;
      
    case 'win': // 獲勝音效 (簡單琶音)
      const freqs = [523.25, 659.25, 783.99, 1046.50]; 
      freqs.forEach((freq, i) => {
        const osc = audioCtx.createOscillator();
        const g = audioCtx.createGain();
        osc.connect(g);
        g.connect(audioCtx.destination);
        osc.type = 'sine';
        osc.frequency.value = freq;
        const time = now + i * 0.15;
        g.gain.setValueAtTime(0, time);
        g.gain.linearRampToValueAtTime(0.4, time + 0.05);
        g.gain.exponentialRampToValueAtTime(0.01, time + 0.5);
        osc.start(time);
        osc.stop(time + 0.5);
      });
      break;
  }
};


export default function App() {
  const [gameState, setGameState] = useState('setup'); 
  const [numPlayers, setNumPlayers] = useState(2);
  const [startingScore, setStartingScore] = useState(301);
  const [soundEnabled, setSoundEnabled] = useState(true);

  const [players, setPlayers] = useState([]);
  const [activePlayerIndex, setActivePlayerIndex] = useState(0);
  const [turnStartScore, setTurnStartScore] = useState(0);
  const [turnDarts, setTurnDarts] = useState([]);
  const [turnMessage, setTurnMessage] = useState('');
  const [winner, setWinner] = useState(null);
  
  // 記錄每個分數圈被選中作為 2 倍區的 1/12 區塊索引 (0 ~ 11)
  const [multipliers, setMultipliers] = useState({});

  const svgRef = useRef(null);
  const lastThrowTime = useRef(0); // 用於 3秒防連觸

  // 畫面載入時產生一次預設的雙倍區
  useEffect(() => {
    generateMultipliers();
  }, []);

  // 隨機產生各分數圈的 2倍分數區位 (0-11)
  const generateMultipliers = () => {
    const newMultipliers = {};
    RINGS.forEach(r => {
      if (r.points !== 50) {
        newMultipliers[r.points] = Math.floor(Math.random() * 12); // 0 到 11
      }
    });
    setMultipliers(newMultipliers);
  };

  // 初始化遊戲
  const startGame = () => {
    if (audioCtx.state === 'suspended') {
      audioCtx.resume();
    }
    generateMultipliers(); // 每局重新隨機分配 2 倍區
    
    const newPlayers = Array.from({ length: numPlayers }, (_, i) => ({
      id: i,
      name: `玩家 ${i + 1}`,
      score: startingScore
    }));
    setPlayers(newPlayers);
    setActivePlayerIndex(0);
    setTurnStartScore(startingScore);
    setTurnDarts([]);
    setTurnMessage('');
    setWinner(null);
    setGameState('playing');
  };

  // 播放對應分數的音效
  const playScoreSound = (points, isDouble) => {
    if (!soundEnabled) return;
    
    if (isDouble) {
      playSound('double');
    } else if (points === 50) {
      playSound('bullseye');
    } else if (points >= 15) {
      playSound('high');
    } else if (points > 0) {
      playSound('mid');
    } else {
      playSound('miss');
    }
  };

  // 處理點擊靶面投擲飛鏢
  const handleBoardClick = (e) => {
    e.preventDefault(); // 阻擋預設點擊行為

    if (gameState !== 'playing' || turnMessage !== '') return;

    // --- 3秒防連觸機制 ---
    const now = Date.now();
    if (now - lastThrowTime.current < 3000) return; 
    lastThrowTime.current = now;

    const svg = svgRef.current;
    const pt = svg.createSVGPoint();
    pt.x = e.clientX;
    pt.y = e.clientY;
    const { x, y } = pt.matrixTransform(svg.getScreenCTM().inverse());

    const dx = x - 300;
    const dy = y - 300;
    const distance = Math.sqrt(dx * dx + dy * dy);

    // 計算落點角度以判斷是否落在 2 倍區塊內
    // Math.atan2 給出弧度，並轉換成 0~360 度 (以正上方 12點鐘方向為 0 度)
    let angleDeg = Math.atan2(dy, dx) * (180 / Math.PI);
    angleDeg = (angleDeg + 90) % 360;
    if (angleDeg < 0) angleDeg += 360;
    
    const segmentHit = Math.floor(angleDeg / 30); // 算出落在第幾個 1/12 區塊 (0~11)

    let points = 0;
    let isDouble = false;

    // 判斷落點與分數
    for (let ring of RINGS) {
      if (distance <= ring.radius) {
        points = ring.points;
        // 如果命中該圈的加倍區
        if (ring.points !== 50 && multipliers[ring.points] === segmentHit) {
          isDouble = true;
          points *= 2; // 分數兩倍
        }
        break;
      }
    }

    playScoreSound(points, isDouble);
    processThrow(points, { x, y }, isDouble);
  };

  // 處理計分邏輯
  const processThrow = (points, pos, isDouble) => {
    const newDarts = [...turnDarts, { points, pos, isDouble }];
    setTurnDarts(newDarts);

    const currentTotal = newDarts.reduce((sum, d) => sum + d.points, 0);
    const tentativeScore = turnStartScore - currentTotal;

    if (tentativeScore < 0) {
      setTurnMessage('爆鏢 (Bust)!');
      if (soundEnabled) setTimeout(() => playSound('bust'), 300);
      setTimeout(() => endTurn(turnStartScore), 2000);
    } else if (tentativeScore === 0) {
      setTurnMessage('獲勝！');
      if (soundEnabled) setTimeout(() => playSound('win'), 300);
      const updatedPlayers = [...players];
      updatedPlayers[activePlayerIndex].score = 0;
      setPlayers(updatedPlayers);
      setWinner(updatedPlayers[activePlayerIndex]);
      setGameState('over');
    } else if (newDarts.length === 3) {
      setTurnMessage('換下一位');
      setTimeout(() => endTurn(tentativeScore), 1500);
    }
  };

  const endTurn = (finalScoreForActivePlayer) => {
    const updatedPlayers = [...players];
    updatedPlayers[activePlayerIndex].score = finalScoreForActivePlayer;
    setPlayers(updatedPlayers);

    const nextPlayerIdx = (activePlayerIndex + 1) % numPlayers;
    setActivePlayerIndex(nextPlayerIdx);
    setTurnStartScore(updatedPlayers[nextPlayerIdx].score);
    setTurnDarts([]);
    setTurnMessage('');
  };

  const undoLastDart = () => {
    if (turnDarts.length === 0 || turnMessage) return;
    setTurnDarts(prev => prev.slice(0, -1));
  };

  const resetGame = () => {
    setGameState('setup');
  };

  // 輔助函式：取得 1/12 區塊的 SVG 路徑
  const getWedgePath = (radius, segmentIndex) => {
    const startAngle = (segmentIndex * 30 - 90) * (Math.PI / 180);
    const endAngle = ((segmentIndex + 1) * 30 - 90) * (Math.PI / 180);
    const startX = 300 + radius * Math.cos(startAngle);
    const startY = 300 + radius * Math.sin(startAngle);
    const endX = 300 + radius * Math.cos(endAngle);
    const endY = 300 + radius * Math.sin(endAngle);
    return `M 300 300 L ${startX} ${startY} A ${radius} ${radius} 0 0 1 ${endX} ${endY} Z`;
  };

  // 輔助函式：取得 1/12 區塊內部的文字座標 (置中於環帶)
  const getWedgeTextPos = (radius, segmentIndex) => {
    const midRadius = radius - 25; // 每個環寬度為50，半徑退後25即為環的中心
    const midAngle = (segmentIndex * 30 - 75) * (Math.PI / 180);
    return {
      x: 300 + midRadius * Math.cos(midAngle),
      y: 300 + midRadius * Math.sin(midAngle)
    };
  };

  // 畫面：遊戲設定
  if (gameState === 'setup') {
    return (
      <div 
        className="min-h-screen bg-slate-100 flex items-center justify-center font-sans p-4"
        onContextMenu={(e) => e.preventDefault()} // 封鎖右鍵
        style={{ touchAction: 'none' }} // 封鎖雙指縮放等觸控行為
      >
        <div className="bg-white rounded-3xl shadow-2xl p-10 max-w-2xl w-full text-center relative">
          <button 
            onClick={() => setSoundEnabled(!soundEnabled)}
            className="absolute top-6 right-6 p-3 bg-slate-100 hover:bg-slate-200 rounded-full text-slate-600 transition-colors"
          >
            {soundEnabled ? <Volume2 className="w-8 h-8" /> : <VolumeX className="w-8 h-8" />}
          </button>

          <Target className="w-24 h-24 text-red-500 mx-auto mb-6" />
          <h1 className="text-5xl font-extrabold text-slate-800 mb-10">電子白板飛鏢對戰</h1>
          
          <div className="mb-10">
            <h2 className="text-2xl font-bold text-slate-600 mb-4 flex items-center justify-center gap-2">
              <Users className="w-8 h-8" /> 選擇遊玩人數
            </h2>
            <div className="flex justify-center gap-4">
              {[1, 2, 3].map(num => (
                <button
                  key={num}
                  onClick={() => setNumPlayers(num)}
                  className={`w-24 h-24 text-4xl font-bold rounded-2xl transition-all ${
                    numPlayers === num 
                      ? 'bg-blue-600 text-white shadow-lg scale-110' 
                      : 'bg-slate-200 text-slate-600 hover:bg-slate-300'
                  }`}
                >
                  {num}
                </button>
              ))}
            </div>
          </div>

          <div className="mb-12">
            <h2 className="text-2xl font-bold text-slate-600 mb-4 flex items-center justify-center gap-2">
              <Target className="w-8 h-8" /> 選擇起始分數
            </h2>
            <div className="flex justify-center gap-6">
              {[301, 501].map(score => (
                <button
                  key={score}
                  onClick={() => setStartingScore(score)}
                  className={`px-8 py-4 text-4xl font-bold rounded-2xl transition-all ${
                    startingScore === score 
                      ? 'bg-green-600 text-white shadow-lg scale-110' 
                      : 'bg-slate-200 text-slate-600 hover:bg-slate-300'
                  }`}
                >
                  {score}
                </button>
              ))}
            </div>
          </div>

          <button
            onClick={startGame}
            className="w-full py-6 bg-red-600 hover:bg-red-700 text-white text-4xl font-black rounded-2xl shadow-xl flex items-center justify-center gap-4 transition-transform active:scale-95"
          >
            <Play className="w-10 h-10" /> 開始對戰
          </button>
        </div>
      </div>
    );
  }

  const currentTotalDarts = turnDarts.reduce((sum, d) => sum + d.points, 0);
  const currentDisplayScore = turnStartScore - currentTotalDarts;

  return (
    <div 
      className="min-h-screen bg-slate-900 text-white flex flex-col md:flex-row font-sans overflow-hidden"
      onContextMenu={(e) => e.preventDefault()} // 封鎖白板長按右鍵
      style={{ touchAction: 'none', userSelect: 'none', WebkitUserSelect: 'none' }} // 徹底阻擋手勢縮放/拖曳/反白
    >
      
      {/* 計分板區域 */}
      <div className="w-full md:w-1/3 bg-slate-800 p-6 flex flex-col shadow-2xl z-10 border-r border-slate-700">
        <div className="flex items-center justify-between mb-8">
          <h2 className="text-3xl font-black text-slate-200 flex items-center gap-3">
            <Target className="text-red-500 w-8 h-8" /> 01 賽制 ({startingScore})
          </h2>
          <div className="flex gap-3">
             <button 
              onClick={() => setSoundEnabled(!soundEnabled)}
              className="p-3 bg-slate-700 hover:bg-slate-600 rounded-xl transition-colors"
            >
              {soundEnabled ? <Volume2 className="w-8 h-8 text-green-400" /> : <VolumeX className="w-8 h-8 text-red-400" />}
            </button>
            <button 
              onClick={resetGame}
              className="p-3 bg-slate-700 hover:bg-slate-600 rounded-xl transition-colors"
            >
              <RotateCcw className="w-8 h-8 text-white" />
            </button>
          </div>
        </div>

        <div className="flex-1 flex flex-col gap-4">
          {players.map((player, idx) => {
            const isActive = idx === activePlayerIndex;
            const displayScore = isActive ? currentDisplayScore : player.score;
            
            return (
              <div 
                key={player.id} 
                className={`relative p-6 rounded-3xl border-4 transition-all duration-300 ${
                  isActive 
                    ? 'bg-blue-600 border-blue-400 shadow-[0_0_30px_rgba(59,130,246,0.5)] scale-[1.02]' 
                    : 'bg-slate-700 border-slate-600 opacity-70'
                }`}
              >
                {isActive && (
                  <div className="absolute -top-4 -right-4 bg-yellow-400 text-yellow-900 text-xl font-black px-4 py-1 rounded-full shadow-lg animate-bounce">
                    本回合
                  </div>
                )}
                <h3 className="text-3xl font-bold mb-2">{player.name}</h3>
                <div className="text-7xl font-black text-right tracking-tighter">
                  {displayScore}
                </div>
              </div>
            );
          })}
        </div>

        {/* 控制面板 */}
        <div className="mt-8 bg-slate-700 rounded-3xl p-6">
          <h4 className="text-xl font-bold text-slate-400 mb-4 text-center">本回合飛鏢</h4>
          <div className="flex justify-center gap-4 mb-6">
            {[0, 1, 2].map(i => {
              const dart = turnDarts[i];
              return (
                <div 
                  key={i} 
                  className={`w-20 h-20 rounded-2xl flex flex-col items-center justify-center border-4 ${
                    dart 
                      ? (dart.isDouble ? 'bg-yellow-100 border-yellow-400 text-yellow-700' : 'bg-white border-slate-300 text-slate-800') 
                      : 'bg-slate-800 border-slate-600 text-slate-600'
                  }`}
                >
                  <span className="text-4xl font-black">{dart ? dart.points : '-'}</span>
                  {dart?.isDouble && <span className="text-[10px] font-bold tracking-widest mt-1">DOUBLE</span>}
                </div>
              )
            })}
          </div>
          <button 
            onClick={undoLastDart}
            disabled={turnDarts.length === 0 || turnMessage !== ''}
            className="w-full py-4 bg-slate-600 disabled:opacity-50 disabled:cursor-not-allowed hover:bg-slate-500 rounded-xl text-2xl font-bold flex items-center justify-center gap-3 transition-colors"
          >
            <Undo2 className="w-8 h-8" /> 復原上一鏢
          </button>
        </div>
      </div>

      {/* 靶面區域 */}
      <div className="w-full md:w-2/3 p-4 flex flex-col items-center justify-center relative bg-[radial-gradient(ellipse_at_center,_var(--tw-gradient-stops))] from-slate-800 to-slate-900">
        
        {turnMessage && !winner && (
          <div className="absolute z-20 top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 bg-black/80 text-white text-7xl font-black py-8 px-16 rounded-3xl border-8 border-red-500 shadow-2xl backdrop-blur-sm animate-pulse whitespace-nowrap">
            {turnMessage}
          </div>
        )}

        {winner && (
          <div className="absolute z-50 inset-0 bg-black/90 flex flex-col items-center justify-center backdrop-blur-md">
            <Trophy className="w-48 h-48 text-yellow-400 mb-8 animate-bounce" />
            <h2 className="text-8xl font-black text-white mb-6 text-center">
              {winner.name} <span className="text-yellow-400">獲勝！</span>
            </h2>
            <button
              onClick={resetGame}
              className="mt-12 px-12 py-6 bg-blue-600 hover:bg-blue-700 text-4xl font-bold rounded-2xl shadow-xl flex items-center gap-4 transition-transform active:scale-95"
            >
              <RotateCcw className="w-10 h-10" /> 再玩一局
            </button>
          </div>
        )}

        {/* SVG 靶面 */}
        <div className="relative max-w-[800px] w-full aspect-square shadow-2xl rounded-full bg-[#8B4513] border-[12px] border-[#5c2e0b]">
          <svg 
            ref={svgRef}
            viewBox="0 0 600 600" 
            className="w-full h-full cursor-crosshair touch-none select-none"
            onPointerDown={handleBoardClick}
          >
            {/* 木質背板/脫靶區 */}
            <circle cx="300" cy="300" r="300" fill="#8B4513" />

            {/* 繪製靶圈與兩倍區 (由大到小繪製，可自然覆蓋不超出圓心) */}
            {RINGS.slice().reverse().map((ring) => {
              const segIdx = multipliers[ring.points];
              const hasMultiplier = ring.points !== 50 && segIdx !== undefined;

              return (
                <g key={`ring-${ring.points}`}>
                  {/* 底色圓圈 */}
                  <circle 
                    cx="300" 
                    cy="300" 
                    r={ring.radius} 
                    fill={ring.fill} 
                    stroke="#000000" 
                    strokeWidth="2"
                    className="transition-opacity hover:opacity-90"
                  />
                  
                  {/* 若有分配 2 倍區，則繪製扇形 */}
                  {hasMultiplier && (() => {
                    const pathStr = getWedgePath(ring.radius, segIdx);
                    const textPos = getWedgeTextPos(ring.radius, segIdx);
                    return (
                      <g>
                        <path 
                          d={pathStr} 
                          fill={ring.multFill} 
                          stroke="#000000" 
                          strokeWidth="2" 
                        />
                        <text 
                          x={textPos.x} 
                          y={textPos.y} 
                          fill="#ffffff" 
                          fontSize="22" 
                          fontWeight="bold" 
                          textAnchor="middle" 
                          dominantBaseline="middle"
                          className="pointer-events-none drop-shadow-md"
                        >
                          {ring.points * 2}
                        </text>
                      </g>
                    );
                  })()}

                  {/* 原始分數文字 (如果兩倍區剛好落在正上方，就不顯示原始文字避免重疊) */}
                  {ring.points !== 50 && segIdx !== 0 && (
                    <text 
                      x="300" 
                      y={ring.textY} 
                      fill={ring.textFill} 
                      fontSize="24" 
                      fontWeight="bold" 
                      textAnchor="middle" 
                      dominantBaseline="middle"
                      className="pointer-events-none"
                    >
                      {ring.points}
                    </text>
                  )}
                </g>
              )
            })}
            
            {/* 中心 50 分文字 */}
            <text 
              x="300" y="300" 
              fill="#ffffff" fontSize="28" fontWeight="black" 
              textAnchor="middle" dominantBaseline="middle"
              className="pointer-events-none"
            >
              50
            </text>

            {/* 繪製本回合投擲的飛鏢落點 */}
            {turnDarts.map((dart, idx) => (
              <g key={idx} transform={`translate(${dart.pos.x}, ${dart.pos.y})`} className="pointer-events-none">
                <circle cx="-3" cy="3" r="10" fill="rgba(0,0,0,0.4)" />
                <circle cx="0" cy="0" r="12" fill={dart.isDouble ? "#facc15" : "#f87171"} stroke="#ffffff" strokeWidth="4" />
                <circle cx="0" cy="0" r="4" fill="#000000" />
                <text x="18" y="-18" fill="#ffffff" fontSize="20" fontWeight="bold" stroke="#000" strokeWidth="1" filter="drop-shadow(1px 1px 1px rgba(0,0,0,0.8))">
                  {idx + 1}
                </text>
              </g>
            ))}
          </svg>
        </div>
        <p className="mt-8 text-2xl text-slate-400 font-bold tracking-widest bg-slate-800/50 px-8 py-3 rounded-full flex items-center gap-3">
          丟吧！中心在你心中
          {soundEnabled || <span className="text-red-400 text-sm">(音效已關閉)</span>}
        </p>
      </div>
    </div>
  );
}
