import React, { useState, useEffect, useRef, useMemo } from 'react';
import { Play, Pause, RotateCcw, Flag, Trophy, Zap, TrendingUp, Target, Award, Activity } from 'lucide-react';

export default function Stopwatch() {
  const [time, setTime] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const [laps, setLaps] = useState([]);
  const [particleSystem, setParticleSystem] = useState([]);
  const [soundEnabled, setSoundEnabled] = useState(true);
  const intervalRef = useRef(null);
  const canvasRef = useRef(null);
  const bgCanvasRef = useRef(null);
  const particleCanvasRef = useRef(null);
  const audioContextRef = useRef(null);

  // Initialize audio context
  useEffect(() => {
    audioContextRef.current = new (window.AudioContext || window.webkitAudioContext)();
  }, []);

  // Sound effects
  const playSound = (frequency, duration, type = 'sine') => {
    if (!soundEnabled || !audioContextRef.current) return;
    
    const oscillator = audioContextRef.current.createOscillator();
    const gainNode = audioContextRef.current.createGain();
    
    oscillator.connect(gainNode);
    gainNode.connect(audioContextRef.current.destination);
    
    oscillator.frequency.value = frequency;
    oscillator.type = type;
    
    gainNode.gain.setValueAtTime(0.1, audioContextRef.current.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(0.01, audioContextRef.current.currentTime + duration);
    
    oscillator.start(audioContextRef.current.currentTime);
    oscillator.stop(audioContextRef.current.currentTime + duration);
  };

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setTime(prevTime => prevTime + 10);
      }, 10);
    } else {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [isRunning]);

  // Particle system
  useEffect(() => {
    if (!isRunning) return;

    const interval = setInterval(() => {
      setParticleSystem(prev => {
        const newParticles = [...prev];
        
        // Add new particle
        if (Math.random() > 0.7) {
          newParticles.push({
            x: Math.random() * 100,
            y: Math.random() * 100,
            vx: (Math.random() - 0.5) * 0.5,
            vy: (Math.random() - 0.5) * 0.5,
            life: 1,
            size: Math.random() * 3 + 1,
            hue: Math.random() * 60 + 180
          });
        }
        
        // Update and filter particles
        return newParticles
          .map(p => ({
            ...p,
            x: p.x + p.vx,
            y: p.y + p.vy,
            life: p.life - 0.02
          }))
          .filter(p => p.life > 0);
      });
    }, 50);

    return () => clearInterval(interval);
  }, [isRunning]);

  // Render particles
  useEffect(() => {
    const canvas = particleCanvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    particleSystem.forEach(p => {
      ctx.beginPath();
      ctx.arc(p.x * canvas.width / 100, p.y * canvas.height / 100, p.size, 0, Math.PI * 2);
      ctx.fillStyle = `hsla(${p.hue}, 80%, 60%, ${p.life * 0.6})`;
      ctx.fill();
      
      ctx.shadowBlur = 10;
      ctx.shadowColor = `hsla(${p.hue}, 80%, 60%, ${p.life})`;
    });
  }, [particleSystem]);

  // Animated background grid
  useEffect(() => {
    const canvas = bgCanvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    let animationId;
    let offset = 0;

    const animate = () => {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      
      const gridSize = 40;
      const cols = Math.ceil(canvas.width / gridSize) + 1;
      const rows = Math.ceil(canvas.height / gridSize) + 1;
      
      offset += 0.3;
      
      for (let i = 0; i < cols; i++) {
        for (let j = 0; j < rows; j++) {
          const x = i * gridSize - (offset % gridSize);
          const y = j * gridSize;
          
          const distance = Math.sqrt(Math.pow(x - canvas.width / 2, 2) + Math.pow(y - canvas.height / 2, 2));
          const maxDistance = Math.sqrt(Math.pow(canvas.width / 2, 2) + Math.pow(canvas.height / 2, 2));
          const opacity = 0.1 * (1 - distance / maxDistance);
          
          ctx.strokeStyle = `rgba(59, 130, 246, ${opacity})`;
          ctx.lineWidth = 1;
          ctx.strokeRect(x, y, gridSize, gridSize);
        }
      }
      
      animationId = requestAnimationFrame(animate);
    };

    animate();

    return () => cancelAnimationFrame(animationId);
  }, []);

  // Main circular display
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    const centerX = canvas.width / 2;
    const centerY = canvas.height / 2;
    const radius = 160;

    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Outer glow rings
    for (let i = 0; i < 3; i++) {
      ctx.beginPath();
      ctx.arc(centerX, centerY, radius + 15 + i * 8, 0, 2 * Math.PI);
      ctx.strokeStyle = `rgba(59, 130, 246, ${0.1 - i * 0.03})`;
      ctx.lineWidth = 2;
      ctx.stroke();
    }

    // Background track
    ctx.beginPath();
    ctx.arc(centerX, centerY, radius, 0, 2 * Math.PI);
    ctx.strokeStyle = 'rgba(30, 41, 59, 0.5)';
    ctx.lineWidth = 12;
    ctx.stroke();

    // Progress arc
    const progress = (time % 60000) / 60000;
    const startAngle = -Math.PI / 2;
    const endAngle = startAngle + (2 * Math.PI * progress);

    const gradient = ctx.createConicGradient(0, centerX, centerY);
    gradient.addColorStop(0, '#06b6d4');
    gradient.addColorStop(0.33, '#3b82f6');
    gradient.addColorStop(0.66, '#8b5cf6');
    gradient.addColorStop(1, '#06b6d4');

    ctx.beginPath();
    ctx.arc(centerX, centerY, radius, startAngle, endAngle);
    ctx.strokeStyle = gradient;
    ctx.lineWidth = 12;
    ctx.lineCap = 'round';
    ctx.stroke();

    // Inner glow
    if (isRunning) {
      ctx.shadowBlur = 30;
      ctx.shadowColor = '#3b82f6';
      ctx.beginPath();
      ctx.arc(centerX, centerY, radius, startAngle, endAngle);
      ctx.strokeStyle = gradient;
      ctx.lineWidth = 12;
      ctx.lineCap = 'round';
      ctx.stroke();
    }

    // Tick marks
    for (let i = 0; i < 60; i++) {
      const angle = (i * Math.PI * 2) / 60 - Math.PI / 2;
      const isMajor = i % 5 === 0;
      const innerRadius = radius - (isMajor ? 15 : 10);
      const outerRadius = radius - 5;
      
      const x1 = centerX + Math.cos(angle) * innerRadius;
      const y1 = centerY + Math.sin(angle) * innerRadius;
      const x2 = centerX + Math.cos(angle) * outerRadius;
      const y2 = centerY + Math.sin(angle) * outerRadius;
      
      ctx.beginPath();
      ctx.moveTo(x1, y1);
      ctx.lineTo(x2, y2);
      ctx.strokeStyle = isMajor ? 'rgba(255, 255, 255, 0.6)' : 'rgba(255, 255, 255, 0.2)';
      ctx.lineWidth = isMajor ? 3 : 1.5;
      ctx.stroke();
    }

    // Current position indicator
    const currentAngle = startAngle + (2 * Math.PI * progress);
    const indicatorX = centerX + Math.cos(currentAngle) * radius;
    const indicatorY = centerY + Math.sin(currentAngle) * radius;
    
    ctx.beginPath();
    ctx.arc(indicatorX, indicatorY, 8, 0, Math.PI * 2);
    ctx.fillStyle = '#ffffff';
    ctx.shadowBlur = 15;
    ctx.shadowColor = '#ffffff';
    ctx.fill();

  }, [time, isRunning]);

  // Analytics calculations
  const analytics = useMemo(() => {
    if (laps.length === 0) return null;

    const lapTimes = laps.map((lap, i) => 
      i === laps.length - 1 ? lap : lap - laps[i + 1]
    );

    const bestLapTime = Math.min(...lapTimes);
    const worstLapTime = Math.max(...lapTimes);
    const avgLapTime = lapTimes.reduce((a, b) => a + b, 0) / lapTimes.length;
    const bestLapIndex = lapTimes.indexOf(bestLapTime);
    const worstLapIndex = lapTimes.indexOf(worstLapTime);

    // Consistency score (lower is better)
    const variance = lapTimes.reduce((sum, time) => sum + Math.pow(time - avgLapTime, 2), 0) / lapTimes.length;
    const stdDev = Math.sqrt(variance);
    const consistency = Math.max(0, 100 - (stdDev / avgLapTime * 100));

    // Trend analysis
    const recentLaps = lapTimes.slice(0, Math.min(3, lapTimes.length));
    const olderLaps = lapTimes.slice(Math.min(3, lapTimes.length));
    const recentAvg = recentLaps.reduce((a, b) => a + b, 0) / recentLaps.length;
    const olderAvg = olderLaps.length > 0 ? olderLaps.reduce((a, b) => a + b, 0) / olderLaps.length : recentAvg;
    const trend = ((recentAvg - olderAvg) / olderAvg * 100);

    return {
      bestLapTime,
      worstLapTime,
      avgLapTime,
      bestLapIndex,
      worstLapIndex,
      consistency: consistency.toFixed(1),
      trend: trend.toFixed(1),
      lapTimes
    };
  }, [laps]);

  const formatTime = (milliseconds) => {
    const mins = Math.floor(milliseconds / 60000);
    const secs = Math.floor((milliseconds % 60000) / 1000);
    const ms = Math.floor((milliseconds % 1000) / 10);
    
    return {
      mins: mins.toString().padStart(2, '0'),
      secs: secs.toString().padStart(2, '0'),
      ms: ms.toString().padStart(2, '0')
    };
  };

  const handleStartPause = () => {
    setIsRunning(!isRunning);
    playSound(isRunning ? 440 : 880, 0.1);
  };

  const handleReset = () => {
    setIsRunning(false);
    setTime(0);
    setLaps([]);
    setParticleSystem([]);
    playSound(220, 0.15);
  };

  const handleLap = () => {
    if (time > 0) {
      setLaps(prevLaps => [time, ...prevLaps]);
      playSound(1320, 0.08, 'triangle');
      
      // Burst particles
      setParticleSystem(prev => [
        ...prev,
        ...Array.from({ length: 20 }, () => ({
          x: 50,
          y: 50,
          vx: (Math.random() - 0.5) * 2,
          vy: (Math.random() - 0.5) * 2,
          life: 1,
          size: Math.random() * 4 + 2,
          hue: Math.random() * 60 + 200
        }))
      ]);
    }
  };

  const formatted = formatTime(time);

  return (
    <div className="min-h-screen bg-black flex items-center justify-center p-4 overflow-hidden relative">
      {/* Animated background */}
      <canvas
        ref={bgCanvasRef}
        width="1920"
        height="1080"
        className="absolute inset-0 w-full h-full opacity-20"
      />

      {/* Particle system */}
      <canvas
        ref={particleCanvasRef}
        width="1920"
        height="1080"
        className="absolute inset-0 w-full h-full pointer-events-none"
      />

      {/* Radial gradient overlay */}
      <div className="absolute inset-0 bg-gradient-radial from-blue-950/20 via-transparent to-transparent"></div>

      <div className="w-full max-w-7xl relative z-10">
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
          {/* Main Stopwatch */}
          <div className="lg:col-span-2">
            <div className="bg-gradient-to-br from-slate-900/90 to-slate-950/90 backdrop-blur-2xl rounded-3xl shadow-2xl p-8 border border-slate-700/30 relative overflow-hidden">
              {/* Scanline effect */}
              <div className="absolute inset-0 pointer-events-none opacity-5" style={{
                backgroundImage: 'repeating-linear-gradient(0deg, transparent, transparent 2px, white 2px, white 4px)'
              }}></div>

              {/* Canvas Circle */}
              <div className="relative flex justify-center mb-8">
                <canvas
                  ref={canvasRef}
                  width="400"
                  height="400"
                  className="absolute"
                />
                <div className="relative z-10 flex flex-col items-center justify-center h-96 mt-2">
                  <div className="mb-6">
                    <div className="text-sm uppercase tracking-widest text-cyan-400 font-bold mb-2 text-center flex items-center gap-2 justify-center">
                      <Activity className="w-4 h-4 animate-pulse" />
                      PRECISION CHRONOMETER
                    </div>
                  </div>
                  <div className="font-mono text-8xl font-extralight text-white tracking-tighter drop-shadow-2xl" style={{
                    textShadow: '0 0 40px rgba(59, 130, 246, 0.5)'
                  }}>
                    {formatted.mins}:{formatted.secs}
                  </div>
                  <div className="font-mono text-6xl text-cyan-400 mt-2 drop-shadow-lg" style={{
                    textShadow: '0 0 20px rgba(6, 182, 212, 0.8)'
                  }}>
                    .{formatted.ms}
                  </div>
                  {isRunning && (
                    <div className="mt-6 px-6 py-2 bg-gradient-to-r from-emerald-500/30 to-green-500/30 rounded-full border-2 border-emerald-400/50 animate-pulse shadow-lg shadow-emerald-500/30">
                      <span className="text-emerald-300 text-sm font-bold uppercase tracking-widest">● ACTIVE</span>
                    </div>
                  )}
                </div>
              </div>

              {/* Control Buttons */}
              <div className="flex gap-6 justify-center items-center">
                <button
                  onClick={handleStartPause}
                  className={`relative w-28 h-28 rounded-full flex items-center justify-center transition-all duration-300 transform hover:scale-110 active:scale-95 ${
                    isRunning
                      ? 'bg-gradient-to-br from-orange-500 via-red-500 to-red-600 shadow-2xl shadow-red-500/50'
                      : 'bg-gradient-to-br from-emerald-500 via-green-500 to-green-600 shadow-2xl shadow-emerald-500/50'
                  }`}
                  style={{
                    boxShadow: isRunning 
                      ? '0 0 60px rgba(239, 68, 68, 0.6), inset 0 0 20px rgba(255, 255, 255, 0.2)'
                      : '0 0 60px rgba(16, 185, 129, 0.6), inset 0 0 20px rgba(255, 255, 255, 0.2)'
                  }}
                >
                  {isRunning ? (
                    <Pause className="w-12 h-12 text-white relative z-10" fill="white" />
                  ) : (
                    <Play className="w-12 h-12 text-white ml-1 relative z-10" fill="white" />
                  )}
                </button>

                <button
                  onClick={handleLap}
                  disabled={!isRunning && time === 0}
                  className="w-28 h-28 rounded-full bg-gradient-to-br from-blue-500 via-purple-500 to-purple-600 hover:from-blue-600 hover:to-purple-700 disabled:from-slate-800 disabled:to-slate-900 disabled:opacity-20 flex items-center justify-center transition-all duration-300 transform hover:scale-110 active:scale-95 relative disabled:shadow-none"
                  style={{
                    boxShadow: '0 0 60px rgba(139, 92, 246, 0.6), inset 0 0 20px rgba(255, 255, 255, 0.2)'
                  }}
                >
                  <Flag className="w-12 h-12 text-white relative z-10" />
                </button>

                <button
                  onClick={handleReset}
                  disabled={time === 0 && laps.length === 0}
                  className="w-28 h-28 rounded-full bg-gradient-to-br from-slate-700 via-slate-800 to-slate-900 hover:from-slate-600 hover:to-slate-800 disabled:opacity-20 flex items-center justify-center transition-all duration-300 shadow-xl disabled:shadow-none transform hover:scale-110 active:scale-95 relative"
                  style={{
                    boxShadow: '0 0 40px rgba(71, 85, 105, 0.4), inset 0 0 20px rgba(255, 255, 255, 0.1)'
                  }}
                >
                  <RotateCcw className="w-11 h-11 text-white relative z-10" />
                </button>
              </div>
            </div>
          </div>

          {/* Analytics Panel */}
          <div className="space-y-6">
            {analytics && (
              <>
                <div className="bg-gradient-to-br from-slate-900/90 to-slate-950/90 backdrop-blur-2xl rounded-3xl p-6 border border-slate-700/30 shadow-2xl">
                  <div className="flex items-center gap-2 mb-4">
                    <Target className="w-5 h-5 text-cyan-400" />
                    <h3 className="text-cyan-400 font-bold uppercase tracking-wider text-sm">Performance Analytics</h3>
                  </div>
                  
                  <div className="space-y-4">
                    <div>
                      <div className="flex justify-between items-center mb-2">
                        <span className="text-slate-400 text-xs uppercase tracking-wide">Consistency Score</span>
                        <span className="text-2xl font-bold text-white">{analytics.consistency}%</span>
                      </div>
                      <div className="h-2 bg-slate-800 rounded-full overflow-hidden">
                        <div 
                          className="h-full bg-gradient-to-r from-cyan-500 to-blue-500 transition-all duration-500 rounded-full"
                          style={{ width: `${analytics.consistency}%` }}
                        ></div>
                      </div>
                    </div>

                    <div>
                      <div className="flex justify-between items-center mb-1">
                        <span className="text-slate-400 text-xs uppercase tracking-wide">Performance Trend</span>
                        <span className={`text-lg font-bold flex items-center gap-1 ${
                          parseFloat(analytics.trend) < 0 ? 'text-emerald-400' : 'text-red-400'
                        }`}>
                          <TrendingUp className={`w-4 h-4 ${parseFloat(analytics.trend) > 0 ? 'rotate-180' : ''}`} />
                          {Math.abs(parseFloat(analytics.trend)).toFixed(1)}%
                        </span>
                      </div>
                      <p className="text-xs text-slate-500">
                        {parseFloat(analytics.trend) < 0 ? 'Getting faster! 🚀' : 'Pace slowing'}
                      </p>
                    </div>
                  </div>
                </div>

                <div className="grid grid-cols-2 gap-4">
                  <div className="bg-gradient-to-br from-emerald-900/40 to-emerald-950/40 backdrop-blur-xl rounded-2xl p-4 border border-emerald-700/30 shadow-lg">
                    <div className="flex items-center gap-2 mb-2">
                      <Trophy className="w-4 h-4 text-emerald-400" />
                      <span className="text-emerald-400 text-xs uppercase tracking-wider font-bold">Best</span>
                    </div>
                    <div className="font-mono text-xl text-white">
                      {(() => {
                        const t = formatTime(analytics.bestLapTime);
                        return `${t.mins}:${t.secs}.${t.ms}`;
                      })()}
                    </div>
                  </div>

                  <div className="bg-gradient-to-br from-blue-900/40 to-blue-950/40 backdrop-blur-xl rounded-2xl p-4 border border-blue-700/30 shadow-lg">
                    <div className="flex items-center gap-2 mb-2">
                      <Zap className="w-4 h-4 text-blue-400" />
                      <span className="text-blue-400 text-xs uppercase tracking-wider font-bold">Avg</span>
                    </div>
                    <div className="font-mono text-xl text-white">
                      {(() => {
                        const t = formatTime(analytics.avgLapTime);
                        return `${t.mins}:${t.secs}.${t.ms}`;
                      })()}
                    </div>
                  </div>
                </div>
              </>
            )}

            {laps.length > 0 && (
              <div className="bg-gradient-to-br from-slate-900/90 to-slate-950/90 backdrop-blur-2xl rounded-3xl p-6 border border-slate-700/30 shadow-2xl">
                <div className="flex items-center justify-between mb-4">
                  <div className="flex items-center gap-2">
                    <Award className="w-5 h-5 text-purple-400" />
                    <h3 className="text-purple-400 font-bold uppercase tracking-wider text-sm">Lap Records</h3>
                  </div>
                  <span className="text-slate-500 text-xs font-bold px-3 py-1 bg-slate-800/50 rounded-full border border-slate-700/50">
                    {laps.length}
                  </span>
                </div>
                
                <div className="max-h-96 overflow-y-auto space-y-2 pr-2 custom-scrollbar">
                  {laps.map((lap, index) => {
                    const lapTime = analytics.lapTimes[index];
                    const lapFormatted = formatTime(lapTime);
                    const isBest = index === analytics.bestLapIndex && laps.length > 1;
                    const isWorst = index === analytics.worstLapIndex && laps.length > 1;
                    
                    return (
                      <div
                        key={index}
                        className={`p-4 rounded-2xl transition-all duration-300 ${
                          isBest
                            ? 'bg-gradient-to-r from-emerald-500/30 to-green-500/20 border-2 border-emerald-400/50 shadow-lg shadow-emerald-500/20'
                            : isWorst
                            ? 'bg-gradient-to-r from-red-500/30 to-orange-500/20 border-2 border-red-400/50 shadow-lg shadow-red-500/20'
                            : 'bg-slate-800/40 border border-slate-700/40 hover:bg-slate-800/60 hover:border-slate-600/60'
                        }`}
                      >
                        <div className="flex items-center justify-between">
                          <div className="flex items-center gap-3">
                            <div className={`w-10 h-10 rounded-xl flex items-center justify-center font-bold text-sm ${
                              isBest
                                ? 'bg-emerald-500/40 text-emerald-200 border-2 border-emerald-400/60 shadow-lg shadow-emerald-500/30'
                                : isWorst
                                ? 'bg-red-500/40 text-red-200 border-2 border-red-400/60 shadow-lg shadow-red-500/30'
                                : 'bg-slate-700/60 text-slate-300 border border-slate-600/60'
                            }`}>
                              {laps.length - index}
                            </div>
                            <div>
                              <div className="font-mono text-white text-xl font-light">
                                {lapFormatted.mins}:{lapFormatted.secs}<span className="text-base text-slate-400">.{lapFormatted.ms}</span>
                              </div>
                              {isBest && (
                                <div className="text-emerald-300 text-xs mt-0.5 flex items-center gap-1 font-bold">
                                  <Trophy className="w-3 h-3" />
                                  FASTEST
                                </div>
                              )}
                              {isWorst && (
                                <div className="text-red-300 text-xs mt-0.5 font-bold">SLOWEST</div>
                              )}
                            </div>
                          </div>
                        </div>
                      </div>
                    );
                  })}
                </div>
              </div>
            )}
          </div>
        </div>
      </div>

      <style jsx>{`
        .custom-scrollbar::-webkit-scrollbar {
          width: 6px;
        }
        .custom-scrollbar::-webkit-scrollbar-track {
          background: rgba(15, 23, 42, 0.5);
          border-radius: 10px;
        }
        .custom-scrollbar::-webkit-scrollbar-thumb {
          background: rgba(59, 130, 246, 0.5);
          border-radius: 10px;
        }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover {
          background: rgba(59, 130, 246, 0.7);
        }
      `}</style>
    </div>
  );
}
