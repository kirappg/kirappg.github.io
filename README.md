import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, collection, doc, setDoc, onSnapshot, updateDoc, addDoc, query, orderBy, limit, deleteDoc, serverTimestamp, where } from 'firebase/firestore';
import { Dices, Scroll, Sword, Crown, FlaskConical, Shield, Map as MapIcon, Plus, Trash2, User, Save, FilePlus, Skull, BookOpen, Eye, Trees, Grid, Activity, X, CheckCircle, AlertCircle, Flame, Ghost, GripVertical, Box, Anchor, Heart, HeartCrack, Ban } from 'lucide-react';

// --- CONFIGURAÇÃO DO FIREBASE ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// --- CONSTANTES ---
const CLASSES = [
  { id: 'cavaleiro', name: 'Cavaleiro Negro', icon: <Shield size={18} />, bonus: 'Força +2' },
  { id: 'mago', name: 'Necromante', icon: <Skull size={18} />, bonus: 'Inteligência +2' },
  { id: 'alquimista', name: 'Alquimista de Sangue', icon: <FlaskConical size={18} />, bonus: 'Sabedoria +2' },
  { id: 'realeza', name: 'Lorde Vampiro', icon: <Crown size={18} />, bonus: 'Carisma +2' },
  { id: 'ladino', name: 'Assassino das Sombras', icon: <Sword size={18} />, bonus: 'Destreza +2' },
];

const ATTRIBUTES = ['Força', 'Destreza', 'Constituição', 'Inteligência', 'Sabedoria', 'Carisma'];

const DEFAULT_CHARACTER = {
  name: '',
  class: 'cavaleiro',
  level: 1,
  hp: 20,
  maxHp: 20,
  attributes: {
    Força: 10, Destreza: 10, Constituição: 10, Inteligência: 10, Sabedoria: 10, Carisma: 10
  },
  skills: [],
  inventory: [],
  bio: '',
  gmNotes: '',
  mapId: 'throne',
  x: 5,
  y: 8,
  lastActive: null
};

const MAP_AREAS = {
  throne: { id: 'throne', name: 'Salão do Trono', rows: 12, cols: 10, theme: 'stone', desc: 'O trono do Rei Louco, banhado à luz de tochas.' },
  dungeon: { id: 'dungeon', name: 'Masmorra', rows: 10, cols: 8, theme: 'dirt', desc: 'Onde os inimigos da coroa apodrecem.' },
  library: { id: 'library', name: 'Biblioteca', rows: 10, cols: 10, theme: 'wood', desc: 'Conhecimento proibido e silêncio absoluto.' },
  graveyard: { id: 'graveyard', name: 'Cemitério', rows: 14, cols: 12, theme: 'grass', desc: 'A neblina cobre os mortos inquietos.' }
};

export default function App() {
  // --- ESTADOS ---
  const [user, setUser] = useState(null);
  const [activeTab, setActiveTab] = useState('character');
  const [notification, setNotification] = useState(null);
  
  // Dados do Firestore
  const [characters, setCharacters] = useState([]);
  const [diceLog, setDiceLog] = useState([]);
  
  // Identidade Local
  const [myCharId, setMyCharId] = useState(null);
  const [localChar, setLocalChar] = useState({ ...DEFAULT_CHARACTER }); 

  // Mapa
  const [activeMapId, setActiveMapId] = useState('throne');
  const currentMap = MAP_AREAS[activeMapId];

  // UI Temporária
  const [newItem, setNewItem] = useState('');
  const [newSkill, setNewSkill] = useState('');
  const [rollingDice, setRollingDice] = useState(null);
  
  // MESTRE: Estado corrigido para usar ID
  const [selectedCharId, setSelectedCharId] = useState(null);

  // Estados Derivados
  const myRemoteChar = characters.find(c => c.id === myCharId);
  const isSpectator = myCharId && characters.length > 0 && !myRemoteChar;
  
  // MESTRE: Deriva o personagem selecionado diretamente da lista viva
  const selectedCharForGM = characters.find(c => c.id === selectedCharId);

  // --- HELPER: NOTIFICAÇÃO ---
  const showNotification = (message, type = 'success') => {
    setNotification({ message, type });
    setTimeout(() => setNotification(null), 3000);
  };

  // --- FIREBASE AUTH & SYNC ---
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const charQuery = query(collection(db, 'artifacts', appId, 'public', 'data', 'rpg_characters'));
    const unsubChar = onSnapshot(charQuery, (snapshot) => {
      const chars = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      setCharacters(chars);
      
      const foundRemote = chars.find(c => c.ownerId === user.uid);
      if (foundRemote) {
          if (foundRemote.id !== myCharId) setMyCharId(foundRemote.id);
          // Atualiza local apenas se não estiver editando ativamente (para evitar pular cursor)
          // Mas garante que HP e status críticos estejam sincronizados
          setLocalChar(prev => ({ 
              ...prev, 
              hp: foundRemote.hp, 
              maxHp: foundRemote.maxHp 
          })); 
      }
    }, (error) => console.error("Erro chars:", error));

    const logQuery = query(collection(db, 'artifacts', appId, 'public', 'data', 'rpg_logs'), orderBy('timestamp', 'desc'), limit(50));
    const unsubLog = onSnapshot(logQuery, (snapshot) => {
      setDiceLog(snapshot.docs.map(d => ({ id: d.id, ...d.data() })));
    }, (error) => console.error("Erro logs:", error));

    return () => { unsubChar(); unsubLog(); };
  }, [user]);

  // --- LOGICA DE MAPA ---
  const getMapCellDetails = (mapId, r, c, rows, cols) => {
      let isWall = r === 0 || r === rows - 1 || c === 0 || c === cols - 1;
      let content = null;
      let style = {};
      let blockMove = isWall;
      const seed = (r * 13 + c * 7) % 100;

      if (mapId === 'throne') {
          if (c >= 4 && c <= 5 && r > 1 && r < rows - 1) {
              style.background = 'repeating-linear-gradient(0deg, #450a0a, #450a0a 10px, #550c0c 10px, #550c0c 20px)';
              style.boxShadow = 'inset 0 0 10px #000';
          }
          if (r === 1 && (c === 4 || c === 5)) { content = <Crown size={28} className="text-yellow-500 drop-shadow-[0_0_10px_rgba(234,179,8,0.5)]" />; blockMove = true; }
          if ((r === 4 || r === 8) && (c === 2 || c === 7)) { content = <div className="w-full h-full bg-neutral-800 rounded-full border-4 border-neutral-700 shadow-xl" />; blockMove = true; }
          if ((c === 0 || c === cols - 1) && (r === 3 || r === 6 || r === 9)) { content = <Flame size={20} className="text-orange-500 animate-pulse drop-shadow-[0_0_8px_orange]" />; }
          if (r === 5 && (c === 0 || c === 1)) { content = <div className="w-full h-3/4 bg-red-950 border border-yellow-900 rounded-sm mt-2" title="Mesa" />; blockMove = true; }
      } else if (mapId === 'dungeon') {
          if ((c === 2 || c === 5) && r !== 3 && r !== 7 && r > 0 && r < rows - 1) { content = <GripVertical size={32} className="text-neutral-500 opacity-70" />; blockMove = true; isWall = false; }
          if ((r === 2 && c === 1) || (r === 8 && c === 6)) { content = <Skull size={20} className="text-neutral-400 opacity-80 rotate-45" />; }
          if ((r === 0) && (c === 3 || c === 4)) { content = <Anchor size={16} className="text-neutral-600 top-0 absolute" />; }
          if ((r===1 && c===1) || (r===1 && c===6)) { content = <div className="w-3/4 h-3/4 bg-yellow-900/40 rounded border border-yellow-900/60" />; }
      } else if (mapId === 'library') {
          if ((r % 3 === 0) && c > 1 && c < cols - 2 && r > 0 && r < rows - 1) { content = <div className="w-full h-full bg-amber-900 border-x-4 border-amber-950 flex flex-col justify-center items-center gap-1 shadow-lg"><div className="w-full h-px bg-amber-950"></div><div className="w-full h-px bg-amber-950"></div></div>; blockMove = true; }
          if ((r === 2 || r === 5 || r === 8) && (c === 1 || c === cols - 2)) { content = <div className="relative w-full h-3/4 bg-amber-900 rounded-sm shadow-md flex items-center justify-center"><BookOpen size={14} className="text-amber-200 opacity-70" /></div>; blockMove = true; }
          if (r === 5 && c === 5) { content = <Flame size={12} className="text-yellow-400 animate-pulse" />; }
      } else if (mapId === 'graveyard') {
          if ((r + c) % 5 === 0 && r > 1 && r < rows - 2 && c > 1 && c < cols - 2) { content = <div className="w-3/4 h-3/4 bg-neutral-700 rounded-t-xl border-b-4 border-neutral-800 shadow-md flex items-center justify-center"><span className="text-[8px] text-neutral-900 font-serif">RIP</span></div>; blockMove = true; }
          if ((r === 2 && c === 2) || (r === 10 && c === 10) || (r === 5 && c === 9)) { content = <Trees size={36} className="text-neutral-800 drop-shadow-[2px_2px_2px_rgba(0,0,0,0.8)]" />; blockMove = true; }
          if (seed > 95) { content = <Ghost size={20} className="text-cyan-100/30 animate-bounce duration-[3000ms]" />; }
      }

      if (!content && !isWall && !blockMove) {
          if (seed < 5) content = <div className="w-1 h-1 bg-white/10 rounded-full" />;
          if (seed > 95 && mapId === 'dungeon') content = <div className="w-2 h-0.5 bg-neutral-700 rotate-45" />;
          if (seed > 90 && mapId === 'graveyard') content = <div className="w-1.5 h-1.5 bg-green-900/50 rounded-full" />;
      }
      return { isWall, content, style, blockMove };
  };

  // --- AÇÕES DO JOGADOR ---
  const handleCreateOrUpdate = async () => {
    if (!user) return;
    if (isSpectator) return;
    if (!localChar.name.trim()) { showNotification("Nome é obrigatório!", "error"); return; }

    let charIdToUse = myCharId;
    if (!charIdToUse) {
        const existingChar = characters.find(c => c.ownerId === user.uid);
        charIdToUse = existingChar ? existingChar.id : doc(collection(db, 'artifacts', appId, 'public', 'data', 'rpg_characters')).id;
    }
    
    const charData = { 
        ...localChar, 
        hp: localChar.hp || 20,
        maxHp: localChar.maxHp || 20,
        id: charIdToUse, 
        ownerId: user.uid, 
        lastActive: serverTimestamp() 
    };

    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rpg_characters', charIdToUse), charData);
      setMyCharId(charIdToUse);
      showNotification("Personagem Salvo!");
    } catch (e) { showNotification("Erro ao salvar.", "error"); }
  };

  const handleNewCharacter = async () => {
      if (isSpectator) {
          alert("Sua alma foi ceifada. Você só pode assistir.");
          return;
      }
      if (window.confirm("ATENÇÃO: Deseja reiniciar o formulário?")) {
          setLocalChar({ ...DEFAULT_CHARACTER });
          setMyCharId(null);
      }
  };

  const handleMove = async (r, c, blockMove) => {
    if (isSpectator) { showNotification("Espectadores não interagem com o mundo físico.", "warning"); return; }
    if (!myCharId) { showNotification("Salve seu personagem primeiro!", "error"); return; }
    if (blockMove) return;
    
    setLocalChar(prev => ({ ...prev, x: c, y: r, mapId: activeMapId }));
    try { await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rpg_characters', myCharId), { x: c, y: r, mapId: activeMapId, lastActive: serverTimestamp() }); } catch (e) {}
  };

  // --- AÇÕES DO MESTRE (GM) ---
  const handleGMUpdateHP = async (charId, currentHp, delta) => {
      // Garante que estamos operando sobre valores numéricos
      const safeHp = typeof currentHp === 'number' ? currentHp : 20;
      const newHp = Math.max(0, safeHp + delta);
      
      try {
          // Atualiza diretamente no documento do ID passado
          await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rpg_characters', charId), { hp: newHp });
          // Não precisamos atualizar estado local, o snapshot cuidará disso
      } catch(e) { 
          console.error(e); 
          showNotification("Erro ao atualizar HP", "error");
      }
  };

  const handleGMAnnihilate = async (charId) => {
      if(window.confirm("TEM CERTEZA? Esta ação apagará permanentemente o personagem.")) {
          try {
              setSelectedCharId(null); // Limpa seleção imediatamente para evitar erros de render
              await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rpg_characters', charId));
              showNotification("Personagem Aniquilado.", "error");
          } catch(e) { 
              console.error(e);
              showNotification("Erro ao aniquilar.", "error"); 
          }
      }
  };

  // --- DADOS ---
  const handleRollDice = async (sides) => {
    if (rollingDice) return;
    if (isSpectator && activeTab === 'dice') { showNotification("Fantasmas não rolam dados.", "warning"); return; }

    let steps = 0; const maxSteps = 10;
    const interval = setInterval(() => {
      setRollingDice({ sides, currentVal: Math.floor(Math.random() * sides) + 1, isFinal: false });
      steps++;
      if (steps >= maxSteps) {
        clearInterval(interval);
        const finalResult = Math.floor(Math.random() * sides) + 1;
        const critical = finalResult === sides || finalResult === 1;
        setRollingDice({ sides, currentVal: finalResult, isFinal: true, critical });
        if (user) {
            const charName = characters.find(c => c.id === myCharId)?.name || (isSpectator ? "Espectro" : "Anônimo");
            addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'rpg_logs'), { sides, result: finalResult, critical, timestamp: Date.now(), playerName: charName });
        }
        setTimeout(() => setRollingDice(null), 1500);
      }
    }, 80);
  };
  
  const handleClearHistory = async () => {
      if (!diceLog.length) return;
      if (window.confirm("Apagar histórico global?")) {
          const promises = diceLog.map(log => deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rpg_logs', log.id)));
          await Promise.all(promises);
      }
  }

  // --- UI HELPERS ---
  const handleLocalAttrChange = (attr, val) => setLocalChar(prev => ({ ...prev, attributes: { ...prev.attributes, [attr]: parseInt(val) || 0 } }));
  const addLocalItem = () => { if (newItem.trim()) { setLocalChar(prev => ({ ...prev, inventory: [...prev.inventory, newItem] })); setNewItem(''); }};
  const addLocalSkill = () => { if (newSkill.trim()) { setLocalChar(prev => ({ ...prev, skills: [...prev.skills, { name: newSkill, value: 0 }] })); setNewSkill(''); }};
  const removeLocalItem = (idx) => setLocalChar(prev => ({ ...prev, inventory: prev.inventory.filter((_, i) => i !== idx) }));
  const removeLocalSkill = (idx) => setLocalChar(prev => ({ ...prev, skills: prev.skills.filter((_, i) => i !== idx) }));

  // --- RENDER ---
  return (
    <div className="min-h-screen bg-neutral-950 text-red-100 font-serif selection:bg-red-900 selection:text-white overflow-hidden flex flex-col md:flex-row">
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=UnifrakturMaguntia&family=Cinzel:wght@400;700&display=swap');
        .font-gothic { font-family: 'UnifrakturMaguntia', cursive; }
        .font-cinzel { font-family: 'Cinzel', serif; }
        ::-webkit-scrollbar { width: 8px; }
        ::-webkit-scrollbar-track { background: #171717; }
        ::-webkit-scrollbar-thumb { background: #7f1d1d; border-radius: 4px; }
        
        .stone-floor { background-color: #1a1a1a; background-image: url("data:image/svg+xml,%3Csvg width='40' height='40' viewBox='0 0 40 40' xmlns='http://www.w3.org/2000/svg'%3E%3Cpath d='M20 20.5V18H0v-2h20v-2H0v-2h20v-2H0V8h20V6H0V4h20V2H0V0h22v20h2V0h2v20h2V0h2v20h2V0h2v20h2v2H0v-1.5z' fill='%23262626' fill-opacity='0.2' fill-rule='evenodd'/%3E%3C/svg%3E"); }
        .dirt-floor { background-color: #2a2018; background-image: radial-gradient(#3a2820 15%, transparent 16%); background-size: 16px 16px; }
        .wood-floor { background-color: #2e1a0f; background-image: linear-gradient(90deg, rgba(0,0,0,0.3) 50%, transparent 50%); background-size: 20px 20px; box-shadow: inset 0 0 50px #000; }
        .grass-floor { background-color: #1a2e1a; background-image: radial-gradient(#264026 20%, transparent 20%); background-size: 12px 12px; }
        .wall-texture { background-color: #0a0a0a; background-image: linear-gradient(45deg, #111 25%, transparent 25%, transparent 75%, #111 75%, #111), linear-gradient(45deg, #111 25%, transparent 25%, transparent 75%, #111 75%, #111); background-size: 20px 20px; background-position: 0 0, 10px 10px; box-shadow: inset 0 0 10px #000; }
        
        @keyframes shake { 0% { transform: rotate(0deg); } 25% { transform: rotate(5deg); } 75% { transform: rotate(-5deg); } 100% { transform: rotate(0deg); } }
        .animate-shake { animation: shake 0.2s infinite; }
        .animate-slide-down { animation: slideDown 0.3s ease-out forwards; }
        @keyframes slideDown { from { transform: translate(-50%, -100%); opacity: 0; } to { transform: translate(-50%, 0); opacity: 1; } }
      `}</style>

      {notification && (
        <div className={`fixed top-4 left-1/2 -translate-x-1/2 z-[100] px-6 py-3 rounded-lg shadow-[0_0_20px_rgba(0,0,0,0.8)] border flex items-center gap-3 animate-slide-down ${notification.type === 'error' ? 'bg-red-950/90 border-red-500 text-red-200' : notification.type === 'warning' ? 'bg-yellow-900/90 border-yellow-500 text-yellow-100' : 'bg-green-900/90 border-green-500 text-green-100'}`}>
            {notification.type === 'error' ? <AlertCircle size={20} /> : notification.type === 'warning' ? <AlertCircle size={20} /> : <CheckCircle size={20} />}
            <span className="font-cinzel font-bold">{notification.message}</span>
        </div>
      )}

      {/* Sidebar */}
      <nav className="bg-neutral-900 border-r border-red-900/30 w-full md:w-24 flex md:flex-col justify-between items-center p-4 z-50 shadow-2xl">
        <div className="hidden md:flex flex-col items-center mb-8">
          <div className="w-12 h-12 bg-black rounded-full flex items-center justify-center border border-red-800 mb-2 shadow-[0_0_15px_rgba(220,38,38,0.4)]">
            <Skull className="text-red-600" />
          </div>
          <span className="text-xs text-red-700 font-cinzel text-center tracking-widest uppercase">Trevas</span>
        </div>
        <div className="flex md:flex-col gap-6 w-full justify-around md:justify-start">
          <NavButton active={activeTab === 'character'} onClick={() => setActiveTab('character')} icon={isSpectator ? <Ghost className="text-cyan-400"/> : <User />} label={isSpectator ? "Espectro" : "Alma"} />
          <NavButton active={activeTab === 'map'} onClick={() => setActiveTab('map')} icon={<MapIcon />} label="Mapa" />
          <NavButton active={activeTab === 'dice'} onClick={() => setActiveTab('dice')} icon={<Dices />} label="Sorte" />
          <div className="w-full h-px bg-red-900/30 md:my-2"></div>
          <NavButton active={activeTab === 'gm'} onClick={() => setActiveTab('gm')} icon={<BookOpen />} label="Mestre" />
        </div>
        <div className="hidden md:block mt-auto text-[10px] text-green-500/50 text-center font-mono">
            {isSpectator ? <span className="text-cyan-500 animate-pulse">MODO ESPECTADOR</span> : `Online • ${characters.length}`}
        </div>
      </nav>

      {/* Main */}
      <main className="flex-1 relative overflow-y-auto bg-neutral-950 p-4 md:p-8">
        <div className="max-w-6xl mx-auto">
          <header className="mb-8 border-b border-red-900/30 pb-4 flex justify-between items-end">
            <div>
              <h1 className="text-4xl md:text-6xl font-gothic text-red-600 drop-shadow-[0_2px_4px_rgba(0,0,0,0.8)] tracking-wide">Fortaleza Sombria</h1>
              <p className="text-neutral-500 font-cinzel mt-2 tracking-widest text-sm uppercase">RPG Online • {characters.length} Jogadores</p>
            </div>
            <div className="hidden md:block text-right">
              <p className="text-xs text-red-900 uppercase tracking-widest">Conectado como</p>
              {isSpectator ? (
                  <p className="font-bold text-cyan-500 text-lg font-gothic animate-pulse flex items-center gap-2 justify-end"><Ghost size={16}/> Alma Perdida</p>
              ) : (
                  <p className="font-bold text-red-500 text-lg font-gothic">{localChar.name || "Espectro"}</p>
              )}
            </div>
          </header>

          <div className="grid gap-8">
            {/* CHARACTER TAB */}
            {activeTab === 'character' && (
              isSpectator ? (
                  <div className="flex flex-col items-center justify-center h-96 animate-fadeIn border border-red-900/30 rounded-lg bg-black/50 p-8">
                      <Skull size={64} className="text-red-800 mb-6 animate-pulse" />
                      <h2 className="text-3xl font-gothic text-red-600 mb-2">Você foi Aniquilado</h2>
                      <p className="text-neutral-400 font-cinzel text-center max-w-md">
                          Sua chama se apagou. Seu corpo não mais existe neste plano.
                          <br/><br/>
                          <span className="text-cyan-500">Você agora é um espectro. Apenas observe o destino dos vivos.</span>
                      </p>
                  </div>
              ) : (
              <div className="animate-fadeIn">
                <div className="mb-6 flex flex-wrap gap-4 bg-neutral-900/50 p-4 rounded border border-red-900/20 items-center justify-between">
                  <div className="flex gap-2">
                    <button onClick={handleNewCharacter} className="flex items-center gap-2 px-4 py-2 bg-neutral-800 hover:bg-neutral-700 text-neutral-300 rounded border border-neutral-700 transition-colors"><FilePlus size={16} /> Novo</button>
                    <button onClick={handleCreateOrUpdate} className="flex items-center gap-2 px-4 py-2 bg-red-900/80 hover:bg-red-800 text-white rounded border border-red-700 shadow-[0_0_10px_rgba(153,27,27,0.3)] transition-colors animate-pulse"><Save size={16} /> Salvar Tudo</button>
                  </div>
                  {myCharId ? <span className="text-xs text-green-500 font-mono flex items-center gap-1"><CheckCircle size={12}/> Vinculado</span> : <span className="text-xs text-yellow-500 font-mono italic">Rascunho</span>}
                </div>
                <div className="grid md:grid-cols-2 gap-8">
                  <div className="space-y-6">
                    <div className="bg-neutral-900/80 p-6 rounded border border-red-900/20 shadow-lg relative">
                      <h2 className="text-2xl font-gothic text-red-500 mb-4 flex items-center gap-2 border-b border-red-900/20 pb-2"><User size={20} /> Identidade</h2>
                      
                      {/* VITALIDADE (HP) */}
                      <div className="mb-6 bg-black p-3 rounded border border-red-900/50 flex items-center justify-between">
                          <div className="flex items-center gap-2 text-red-500 font-cinzel font-bold">
                              <Heart className="fill-red-900" size={20} /> VITALIDADE
                          </div>
                          <div className="text-2xl font-mono font-bold text-white flex items-baseline gap-1">
                              {localChar.hp || 20}<span className="text-xs text-neutral-500">/{localChar.maxHp || 20}</span>
                          </div>
                      </div>
                      <div className="w-full bg-red-900/20 h-2 rounded-full mb-6 overflow-hidden">
                          <div className="bg-red-600 h-full transition-all duration-500" style={{ width: `${((localChar.hp || 20) / (localChar.maxHp || 20)) * 100}%` }}></div>
                      </div>

                      <div className="space-y-4">
                        <div><label className="block text-red-900/70 text-xs mb-1 font-cinzel uppercase">Nome</label><input type="text" value={localChar.name} onChange={(e) => setLocalChar({...localChar, name: e.target.value})} className="w-full bg-black border border-red-900/30 rounded p-2 text-red-100 focus:border-red-600 outline-none font-gothic text-xl" /></div>
                        <div><label className="block text-red-900/70 text-xs mb-1 font-cinzel uppercase">Classe</label><div className="grid grid-cols-2 sm:grid-cols-3 gap-2">{CLASSES.map((cls) => (<button key={cls.id} onClick={() => setLocalChar({...localChar, class: cls.id})} className={`p-2 rounded border flex flex-col items-center justify-center transition-all ${localChar.class === cls.id ? 'bg-red-950/50 border-red-600 text-red-500' : 'bg-black border-neutral-800 hover:border-red-900 text-neutral-600'}`}>{cls.icon}<span className="text-[10px] mt-1 font-bold uppercase">{cls.name}</span></button>))}</div></div>
                        <div><label className="block text-red-900/70 text-xs mb-1 font-cinzel uppercase">Biografia</label><textarea value={localChar.bio} onChange={(e) => setLocalChar({...localChar, bio: e.target.value})} className="w-full bg-black border border-red-900/30 rounded p-2 text-sm h-20 resize-none outline-none text-neutral-400" /></div>
                      </div>
                    </div>
                    <div className="bg-neutral-900/80 p-6 rounded border border-red-900/20 shadow-lg">
                      <h2 className="text-2xl font-gothic text-red-500 mb-4 flex items-center gap-2 border-b border-red-900/20 pb-2"><Activity size={20} /> Atributos</h2>
                      <div className="grid grid-cols-2 gap-4">{ATTRIBUTES.map(attr => (<div key={attr} className="flex justify-between items-center bg-black p-2 rounded border border-neutral-800"><span className="font-cinzel text-xs text-neutral-500 uppercase">{attr}</span><input type="number" value={localChar.attributes[attr]} onChange={(e) => handleLocalAttrChange(attr, e.target.value)} className="w-12 bg-transparent text-center font-bold text-red-500 border-none outline-none" /></div>))}</div>
                    </div>
                  </div>
                  <div className="space-y-6">
                    <div className="bg-neutral-900/80 p-6 rounded border border-red-900/20 shadow-lg">
                      <h2 className="text-2xl font-gothic text-red-500 mb-4 flex items-center gap-2 border-b border-red-900/20 pb-2"><Scroll size={20} /> Inventário</h2>
                      <div className="flex gap-2 mb-4"><input type="text" value={newItem} onChange={(e) => setNewItem(e.target.value)} onKeyDown={(e) => e.key === 'Enter' && addLocalItem()} placeholder="Novo item..." className="flex-1 bg-black border border-red-900/30 rounded p-2 text-sm focus:border-red-600 outline-none text-neutral-300" /><button onClick={addLocalItem} className="bg-red-900 hover:bg-red-800 text-white p-2 rounded"><Plus size={16} /></button></div>
                      <ul className="space-y-2 max-h-40 overflow-y-auto custom-scrollbar">{localChar.inventory.map((item, idx) => (<li key={idx} className="flex justify-between bg-black p-2 rounded border border-neutral-800 text-sm text-neutral-400"><span>{item}</span><button onClick={() => removeLocalItem(idx)} className="text-red-900 hover:text-red-500"><Trash2 size={12} /></button></li>))}</ul>
                    </div>
                    <div className="bg-neutral-900/80 p-6 rounded border border-red-900/20 shadow-lg">
                      <h2 className="text-2xl font-gothic text-red-500 mb-4 flex items-center gap-2 border-b border-red-900/20 pb-2"><FlaskConical size={20} /> Perícias</h2>
                      <div className="flex gap-2 mb-4"><input type="text" value={newSkill} onChange={(e) => setNewSkill(e.target.value)} onKeyDown={(e) => e.key === 'Enter' && addLocalSkill()} placeholder="Nova perícia..." className="flex-1 bg-black border border-red-900/30 rounded p-2 text-sm focus:border-red-600 outline-none text-neutral-300" /><button onClick={addLocalSkill} className="bg-red-900 hover:bg-red-800 text-white p-2 rounded"><Plus size={16} /></button></div>
                      <div className="grid grid-cols-2 gap-2 max-h-40 overflow-y-auto custom-scrollbar">{localChar.skills.map((skill, idx) => (<div key={idx} className="flex justify-between items-center bg-black p-2 rounded border border-neutral-800 text-sm text-neutral-400"><span>{skill.name}</span><button onClick={() => removeLocalSkill(idx)} className="text-red-900 hover:text-red-500"><Trash2 size={12} /></button></div>))}</div>
                    </div>
                  </div>
                </div>
              </div>
              )
            )}

            {/* MAP TAB */}
            {activeTab === 'map' && (
              <div className="flex flex-col items-center animate-fadeIn w-full">
                {isSpectator && (
                    <div className="w-full bg-cyan-900/30 border border-cyan-500/50 p-2 mb-4 text-center rounded text-cyan-200 text-xs font-cinzel animate-pulse">
                        <Ghost className="inline mr-2" size={14}/> MODO OBSERVADOR - Você não pode interagir fisicamente.
                    </div>
                )}
                <div className="w-full max-w-4xl mb-4 flex gap-2 overflow-x-auto pb-2">
                   {Object.values(MAP_AREAS).map(area => (<button key={area.id} onClick={() => setActiveMapId(area.id)} className={`flex-shrink-0 px-4 py-2 rounded border flex items-center gap-2 transition-all ${activeMapId === area.id ? 'bg-red-950 border-red-600 text-white' : 'bg-neutral-900 border-neutral-800 text-neutral-500'}`}>{area.id === 'graveyard' ? <Trees size={16} /> : area.id === 'library' ? <BookOpen size={16} /> : <Crown size={16} />}<span className="font-gothic text-sm">{area.name}</span></button>))}
                </div>
                <div className="bg-neutral-900 border border-red-900/50 rounded p-4 shadow-[0_0_50px_rgba(0,0,0,1)] relative inline-block">
                  <div className="absolute -top-4 left-1/2 transform -translate-x-1/2 bg-black px-4 py-1 border border-red-900 rounded shadow-xl z-20 whitespace-nowrap"><span className="text-red-600 font-gothic tracking-wider">{currentMap.name}</span></div>
                  <div className={`grid gap-0 bg-black p-4 overflow-x-auto border-[20px] border-neutral-800 relative ${activeMapId === 'library' ? 'wood-floor' : activeMapId === 'dungeon' ? 'dirt-floor' : activeMapId === 'graveyard' ? 'grass-floor' : 'stone-floor'}`} style={{ gridTemplateColumns: `repeat(${currentMap.cols}, minmax(40px, 1fr))` }}>
                    {Array.from({ length: currentMap.rows * currentMap.cols }).map((_, i) => {
                      const row = Math.floor(i / currentMap.cols);
                      const col = i % currentMap.cols;
                      const { isWall, content, style, blockMove } = getMapCellDetails(activeMapId, row, col, currentMap.rows, currentMap.cols);
                      const charsHere = characters.filter(c => (c.mapId === activeMapId || (!c.mapId && activeMapId === 'throne')) && c.y === row && c.x === col);

                      return (
                        <div key={i} onClick={() => handleMove(row, col, isWall || blockMove)} className={`h-12 w-12 md:h-16 md:w-16 flex items-center justify-center relative transition-all duration-300 ${isWall ? 'wall-texture z-0' : 'border border-white/5 hover:border-red-500/50 cursor-pointer'}`} style={style}>
                          {content && <div className="absolute inset-0 flex items-center justify-center pointer-events-none z-10">{content}</div>}
                          {charsHere.map((char, idx) => {
                             const isMe = char.id === myCharId;
                             return (
                                <div key={char.id} className="absolute z-20 transition-all duration-500" style={{ transform: charsHere.length > 1 ? `translate(${idx*4}px, ${idx*4}px)` : 'none' }}>
                                    <div className={`w-8 h-8 md:w-10 md:h-10 rounded-full border-2 flex items-center justify-center shadow-lg text-white text-[10px] overflow-hidden ${isMe ? 'bg-red-600 border-white ring-2 ring-red-500' : 'bg-neutral-800 border-neutral-500'}`} title={char.name}>{CLASSES.find(c => c.id === char.class)?.icon || <User size={12}/>}</div>
                                    {isMe && <div className="absolute -bottom-4 left-1/2 -translate-x-1/2 text-[8px] bg-black/50 px-1 rounded text-white z-30">Você</div>}
                                </div>
                             );
                          })}
                        </div>
                      );
                    })}
                  </div>
                </div>
              </div>
            )}

            {activeTab === 'dice' && (
              <div className="grid md:grid-cols-2 gap-8">
                <div className="bg-neutral-900/80 p-6 rounded border border-red-900/20 shadow-lg relative overflow-hidden">
                   {rollingDice && <div className="absolute inset-0 bg-black/90 z-20 flex flex-col items-center justify-center"><div className={`text-6xl font-gothic mb-4 ${!rollingDice.isFinal ? 'animate-shake blur-sm' : 'scale-125'}`}>{rollingDice.currentVal}</div><div className="text-red-500 font-cinzel text-sm animate-pulse">{rollingDice.isFinal ? "RESULTADO" : "ROLANDO..."}</div></div>}
                  <h2 className="text-2xl font-gothic text-red-500 mb-6 text-center">Mesa de Dados</h2>
                  <div className="grid grid-cols-3 gap-4">{[4, 6, 8, 10, 12, 20].map(sides => (<button key={sides} onClick={() => handleRollDice(sides)} disabled={rollingDice !== null} className="h-20 bg-neutral-950 border border-neutral-800 rounded flex flex-col items-center justify-center hover:bg-red-900/20 hover:border-red-600 transition-all"><Dices className="mb-2 text-neutral-500" size={24} /><span className="font-gothic text-xl font-bold text-neutral-300">D{sides}</span></button>))}
                  <button onClick={() => handleRollDice(100)} className="col-span-3 h-12 bg-black/50 border border-neutral-800 rounded text-neutral-400 font-cinzel text-sm hover:text-red-400">D100 (Percentual)</button></div>
                </div>
                <div className="bg-black p-4 rounded border border-neutral-800 h-[400px] flex flex-col">
                  <div className="flex justify-between items-center mb-2 border-b border-red-900/20 pb-2"><h3 className="text-red-900 font-cinzel text-xs uppercase tracking-widest">Histórico</h3>{diceLog.length > 0 && (<button onClick={handleClearHistory} className="text-[10px] text-neutral-600 hover:text-red-500 flex items-center gap-1 transition-colors"><Trash2 size={10} /> Limpar</button>)}</div>
                  <div className="flex-1 overflow-y-auto space-y-2 custom-scrollbar">{diceLog.map((log) => (<div key={log.id} className="bg-neutral-900 p-2 rounded flex justify-between items-center border-l-2 border-red-900/30"><div><span className="text-xs font-bold text-red-400 block">{log.playerName}</span><span className="text-[10px] text-neutral-600 uppercase">Rolou D{log.sides}</span></div><div className={`text-xl font-bold font-gothic ${log.critical ? 'text-yellow-500' : 'text-neutral-300'}`}>{log.result}</div></div>))}</div>
                </div>
              </div>
            )}

            {/* GM TAB - AGORA COM PODERES REAIS E SINCRONIZADOS */}
            {activeTab === 'gm' && (
              <div className="grid md:grid-cols-3 gap-6 h-[600px] animate-fadeIn">
                <div className="bg-neutral-900/80 p-4 rounded border border-red-900/30 overflow-y-auto custom-scrollbar md:col-span-1">
                    <h2 className="text-xl font-gothic text-red-500 mb-4 flex items-center gap-2"><BookOpen size={20} /> Grimório</h2>
                    <div className="space-y-2">
                        {characters.map(char => (
                            // Usa ID para selecionar e atualiza estilo baseado na seleção ativa
                            <div key={char.id} onClick={() => setSelectedCharId(char.id)} className={`p-3 rounded cursor-pointer border transition-all ${selectedCharId === char.id ? 'bg-red-900/20 border-red-600' : 'bg-black border-neutral-800 hover:border-neutral-600'}`}>
                                <div className="flex justify-between items-center">
                                    <span className="font-cinzel font-bold text-neutral-200">{char.name}</span>
                                    <div className="text-[10px] text-neutral-500 bg-neutral-900 px-1 rounded uppercase">{char.class}</div>
                                </div>
                            </div>
                        ))}
                    </div>
                </div>
                <div className="bg-black p-6 rounded border border-neutral-800 md:col-span-2 overflow-y-auto custom-scrollbar">
                    {selectedCharForGM ? (
                        <div className="space-y-6">
                            <div className="flex justify-between items-start border-b border-red-900/20 pb-4">
                                <div>
                                    <h3 className="text-3xl font-gothic text-red-600">{selectedCharForGM.name}</h3>
                                    <p className="text-sm text-neutral-500 font-cinzel uppercase tracking-widest">{CLASSES.find(c => c.id === selectedCharForGM.class)?.name}</p>
                                </div>
                                <div className="text-right">
                                    <button onClick={() => handleGMAnnihilate(selectedCharForGM.id)} className="bg-red-950 hover:bg-red-800 text-red-500 border border-red-800 px-3 py-1 rounded text-xs flex items-center gap-1 transition-all"><Skull size={14}/> ANIQUILAR</button>
                                </div>
                            </div>
                            
                            {/* GM CONTROLE DE VIDA - AGORA REATIVO */}
                            <div className="bg-neutral-900 p-4 rounded border border-red-900/30">
                                <h4 className="text-xs text-red-500 uppercase mb-2 font-bold flex items-center gap-2"><Heart size={14}/> Vitalidade do Personagem</h4>
                                <div className="flex items-center justify-between gap-4">
                                    <button onClick={() => handleGMUpdateHP(selectedCharForGM.id, selectedCharForGM.hp, -5)} className="p-2 bg-red-950 rounded hover:bg-red-800 text-white font-bold">-5</button>
                                    <button onClick={() => handleGMUpdateHP(selectedCharForGM.id, selectedCharForGM.hp, -1)} className="p-2 bg-neutral-800 rounded hover:bg-neutral-700 text-white font-bold">-1</button>
                                    
                                    <div className="flex flex-col items-center w-full">
                                        <div className="text-2xl font-mono font-bold text-white">{selectedCharForGM.hp !== undefined ? selectedCharForGM.hp : 20} <span className="text-sm text-neutral-500">/ {selectedCharForGM.maxHp || 20}</span></div>
                                        <div className="w-full bg-neutral-800 h-2 rounded-full mt-1 overflow-hidden">
                                            <div className="bg-red-600 h-full transition-all" style={{ width: `${((selectedCharForGM.hp !== undefined ? selectedCharForGM.hp : 20) / (selectedCharForGM.maxHp || 20)) * 100}%` }}></div>
                                        </div>
                                    </div>
                                    
                                    <button onClick={() => handleGMUpdateHP(selectedCharForGM.id, selectedCharForGM.hp, 1)} className="p-2 bg-neutral-800 rounded hover:bg-neutral-700 text-white font-bold">+1</button>
                                    <button onClick={() => handleGMUpdateHP(selectedCharForGM.id, selectedCharForGM.hp, 5)} className="p-2 bg-green-900/50 rounded hover:bg-green-800 text-white font-bold">+5</button>
                                </div>
                            </div>

                            <div className="grid grid-cols-2 gap-4">
                                <div className="bg-neutral-900 p-3 rounded">
                                    <h4 className="text-xs text-red-800 uppercase mb-2 font-bold">Atributos</h4>
                                    <div className="grid grid-cols-2 gap-y-1 gap-x-4 text-sm text-neutral-300">{Object.entries(selectedCharForGM.attributes || {}).map(([key, val]) => (<div key={key} className="flex justify-between"><span>{key.slice(0,3)}</span> <span className="text-white">{val}</span></div>))}</div>
                                </div>
                                <div className="bg-neutral-900 p-3 rounded">
                                    <h4 className="text-xs text-red-800 uppercase mb-2 font-bold">Inventário</h4>
                                    <ul className="text-xs text-neutral-300 space-y-1 list-disc pl-4">
                                        {(selectedCharForGM.inventory || []).map((i, idx) => <li key={idx}>{i}</li>)}
                                        {(!selectedCharForGM.inventory || selectedCharForGM.inventory.length === 0) && <li className="opacity-50">Vazio</li>}
                                    </ul>
                                </div>
                            </div>
                            <div className="pt-4 border-t border-red-900/20"><label className="flex items-center gap-2 text-red-500 font-cinzel text-sm mb-2"><Eye size={16} /> Notas do Mestre</label><textarea className="w-full bg-neutral-900 border border-neutral-700 rounded p-2 text-neutral-300 text-sm focus:border-red-600 outline-none h-24 font-mono" placeholder="Segredos..." defaultValue={selectedCharForGM.gmNotes || ''} onBlur={async (e) => { try { await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rpg_characters', selectedCharForGM.id), { gmNotes: e.target.value }); } catch (err) { console.error("Erro nota:", err); } }} /></div>
                        </div>
                    ) : (
                        <div className="h-full flex flex-col items-center justify-center text-neutral-700 opacity-50"><BookOpen size={48} className="mb-4" /><p className="font-cinzel">Selecione um jogador...</p></div>
                    )}
                </div>
              </div>
            )}
          </div>
        </div>
      </main>
    </div>
  );
}

function NavButton({ active, onClick, icon, label }) { return (<button onClick={onClick} className={`flex md:flex-col items-center gap-2 p-3 rounded transition-all w-full md:w-auto ${active ? 'bg-neutral-800 text-red-500 shadow-[0_0_15px_rgba(220,38,38,0.2)] border border-red-900/50' : 'text-neutral-500 hover:bg-neutral-900 hover:text-red-900'}`}><div className={`${active ? 'text-red-500' : ''}`}>{icon}</div><span className="font-cinzel text-xs font-bold md:text-[10px] uppercase tracking-widest">{label}</span></button>); }
