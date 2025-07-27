<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Respiração Consciente</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.7.77/Tone.min.js"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        .breather-circle {
            transition: all 3s cubic-bezier(0.4, 0, 0.2, 1);
        }
        .breather-container {
            perspective: 1000px;
        }
        .toast {
            animation: fade-in-out 5s forwards;
        }
        @keyframes fade-in-out {
            0% { opacity: 0; transform: translateY(20px); }
            10% { opacity: 1; transform: translateY(0); }
            90% { opacity: 1; transform: translateY(0); }
            100% { opacity: 0; transform: translateY(20px); }
        }
    </style>
</head>
<body class="bg-slate-50 text-slate-800 flex flex-col items-center justify-center min-h-screen p-4">

    <div id="toast-container" class="fixed top-5 right-5 z-50"></div>

    <div class="w-full max-w-md mx-auto bg-white rounded-2xl shadow-lg p-6 md:p-8 text-center">
        
        <header class="mb-6">
            <h1 class="text-3xl font-bold text-sky-800">Respiração Consciente</h1>
            <p class="text-slate-500 mt-2">Acalme sua mente, um respiro de cada vez.</p>
        </header>

        <main class="mb-8">
            <div class="breather-container flex items-center justify-center h-64">
                <div id="breather-circle" class="relative w-32 h-32 bg-sky-500 rounded-full flex items-center justify-center shadow-2xl shadow-sky-500/30">
                    <span id="breather-text" class="text-white text-2xl font-medium z-10">Começar</span>
                </div>
            </div>
            <p id="timer-display" class="text-4xl font-mono text-slate-700 h-10 mt-4">00:00</p>
        </main>

        <div id="controls" class="space-y-6">
            <div>
                <label for="duration" class="block text-sm font-medium text-slate-600 mb-2">Duração da Sessão</label>
                <div class="flex justify-center space-x-2">
                    <button data-duration="60" class="duration-btn bg-slate-200 text-slate-700 px-4 py-2 rounded-full font-medium">1 min</button>
                    <button data-duration="300" class="duration-btn bg-sky-100 text-sky-800 px-4 py-2 rounded-full font-medium ring-2 ring-sky-600">5 min</button>
                    <button data-duration="600" class="duration-btn bg-slate-200 text-slate-700 px-4 py-2 rounded-full font-medium">10 min</button>
                </div>
            </div>
            
            <button id="start-stop-btn" class="w-full bg-sky-600 hover:bg-sky-700 text-white font-bold py-3 px-4 rounded-lg text-lg transition-all duration-300 shadow-md hover:shadow-lg">
                Iniciar Sessão
            </button>
        </div>

        <div id="stats" class="mt-8 pt-6 border-t border-slate-200">
             <h3 class="text-lg font-semibold text-slate-700 mb-2">Seu Progresso</h3>
             <div class="flex justify-around text-slate-600">
                <div class="text-center">
                    <p class="text-2xl font-bold text-sky-700" id="sessions-today">0</p>
                    <p class="text-sm">Sessões hoje</p>
                </div>
                <div class="text-center">
                    <p class="text-2xl font-bold text-sky-700" id="total-time">0</p>
                    <p class="text-sm">Minutos totais</p>
                </div>
             </div>
             <div class="text-center mt-4">
                 <p class="text-sm text-slate-400">ID de Usuário: <span id="user-id-display">Carregando...</span></p>
             </div>
        </div>
    </div>
    
    <footer class="mt-8 text-center text-slate-400 text-sm">
        <p>Inspirado na "A Ciência da Meditação".</p>
        <p>&copy; 2025 Ferramenta de Bem-estar Digital.</p>
    </footer>

    <script type="module">
        // Importações do Firebase
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, onSnapshot, updateDoc, increment } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // FIX: Adiciona um listener para o evento 'load' da janela. 
        // Isso garante que todo o código só será executado depois que a página inteira, 
        // incluindo scripts externos como o Tone.js, for completamente carregada.
        window.addEventListener('load', () => {
            // --- Configuração do Firebase ---
            const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : { apiKey: "YOUR_API_KEY", authDomain: "YOUR_AUTH_DOMAIN", projectId: "YOUR_PROJECT_ID" };
            const appId = typeof __app_id !== 'undefined' ? __app_id : 'breathing-app-pt-default';

            const app = initializeApp(firebaseConfig);
            const auth = getAuth(app);
            const db = getFirestore(app);

            let userId = null;
            let userStatsUnsubscribe = null;

            // --- Elementos da UI ---
            const breatherCircle = document.getElementById('breather-circle');
            const breatherText = document.getElementById('breather-text');
            const timerDisplay = document.getElementById('timer-display');
            const startStopBtn = document.getElementById('start-stop-btn');
            const controlsDiv = document.getElementById('controls');
            const durationButtons = document.querySelectorAll('.duration-btn');
            const sessionsTodayEl = document.getElementById('sessions-today');
            const totalTimeEl = document.getElementById('total-time');
            const userIdDisplay = document.getElementById('user-id-display');
            const toastContainer = document.getElementById('toast-container');

            // --- Áudio ---
            let synth = null;

            // --- Estado da Aplicação ---
            let state = {
                isRunning: false,
                sessionDuration: 300, // 5 minutos por padrão
                timeLeft: 300,
                intervalId: null,
                breathingCycle: 'inhale', // inhale, exhale
            };

            // --- Lógica de Autenticação e Dados ---
            onAuthStateChanged(auth, async (user) => {
                if (user) {
                    userId = user.uid;
                    userIdDisplay.textContent = userId.substring(0, 8) + '...';
                    setupUserListener(userId);
                } else {
                    try {
                        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                            await signInWithCustomToken(auth, __initial_auth_token);
                        } else {
                            await signInAnonymously(auth);
                        }
                    } catch (error) {
                        console.error("Erro na autenticação:", error);
                        showToast("Não foi possível conectar. Tente recarregar a página.", "error");
                    }
                }
            });

            async function setupUserListener(uid) {
                const userDocRef = doc(db, `artifacts/${appId}/users/${uid}/breathingData/userStats`);
                
                const docSnap = await getDoc(userDocRef);
                if (!docSnap.exists()) {
                    await setDoc(userDocRef, {
                        sessionsToday: 0,
                        totalMinutes: 0,
                        lastSessionDate: new Date().toISOString().split('T')[0]
                    });
                }

                if (userStatsUnsubscribe) userStatsUnsubscribe();
                userStatsUnsubscribe = onSnapshot(userDocRef, (doc) => {
                    if (doc.exists()) {
                        const data = doc.data();
                        const todayStr = new Date().toISOString().split('T')[0];
                        
                        if(data.lastSessionDate !== todayStr) {
                             updateDoc(userDocRef, { sessionsToday: 0, lastSessionDate: todayStr });
                             sessionsTodayEl.textContent = 0;
                        } else {
                            sessionsTodayEl.textContent = data.sessionsToday || 0;
                        }
                        totalTimeEl.textContent = data.totalMinutes || 0;
                    }
                });
            }
            
            async function updateUserStats() {
                if (!userId) return;
                const userDocRef = doc(db, `artifacts/${appId}/users/${userId}/breathingData/userStats`);
                const sessionMinutes = Math.round(state.sessionDuration / 60);

                try {
                    await updateDoc(userDocRef, {
                        sessionsToday: increment(1),
                        totalMinutes: increment(sessionMinutes),
                        lastSessionDate: new Date().toISOString().split('T')[0]
                    });
                    showToast(`Ótimo trabalho! +${sessionMinutes} min de prática.`, "success");
                } catch (error) {
                    console.error("Erro ao atualizar estatísticas:", error);
                    if (error.code === 'not-found') {
                        await setDoc(userDocRef, {
                            sessionsToday: 1,
                            totalMinutes: sessionMinutes,
                            lastSessionDate: new Date().toISOString().split('T')[0]
                        });
                    }
                }
            }

            // --- Lógica da Respiração ---
            function startBreathing() {
                if (state.isRunning) return;
                state.isRunning = true;
                state.timeLeft = state.sessionDuration;
                updateTimerDisplay();
                
                startStopBtn.textContent = 'Parar Sessão';
                startStopBtn.classList.remove('bg-sky-600', 'hover:bg-sky-700');
                startStopBtn.classList.add('bg-red-500', 'hover:bg-red-600');
                controlsDiv.classList.add('opacity-50', 'pointer-events-none');

                breathingAnimation();
                state.intervalId = setInterval(tick, 1000);
            }

            function stopBreathing(completed = false) {
                if (!state.isRunning) return;
                
                clearInterval(state.intervalId);
                state.isRunning = false;
                state.breathingCycle = 'inhale';

                startStopBtn.textContent = 'Iniciar Sessão';
                startStopBtn.classList.add('bg-sky-600', 'hover:bg-sky-700');
                startStopBtn.classList.remove('bg-red-500', 'hover:bg-red-600');
                controlsDiv.classList.remove('opacity-50', 'pointer-events-none');
                
                breatherText.textContent = completed ? "Concluído!" : "Pausado";
                breatherCircle.style.transform = 'scale(1)';
                breatherCircle.style.backgroundColor = '#0ea5e9'; // Cor padrão do céu
                
                if (completed) {
                    updateUserStats();
                }

                setTimeout(() => {
                    if (!state.isRunning) {
                         breatherText.textContent = "Começar";
                         state.timeLeft = state.sessionDuration;
                         updateTimerDisplay();
                    }
                }, 2000);
            }

            function breathingAnimation() {
                if (!state.isRunning) return;

                if (state.breathingCycle === 'inhale') {
                    breatherText.textContent = 'Inspire...';
                    breatherCircle.style.transform = 'scale(1.5)';
                    breatherCircle.style.transitionDuration = '4s';
                    breatherCircle.style.backgroundColor = '#6ee7b7'; // Verde menta
                    if (synth) synth.triggerAttackRelease("C5", "8n"); // Som agudo

                    setTimeout(() => {
                        if (state.isRunning) {
                            state.breathingCycle = 'exhale';
                            breathingAnimation();
                        }
                    }, 4000); // 4 segundos inspirando
                } else {
                    breatherText.textContent = 'Expire...';
                    breatherCircle.style.transform = 'scale(1)';
                    breatherCircle.style.transitionDuration = '6s';
                    breatherCircle.style.backgroundColor = '#818cf8'; // Lilás
                    if (synth) synth.triggerAttackRelease("C4", "8n"); // Som grave
                    
                    setTimeout(() => {
                        if (state.isRunning) {
                            state.breathingCycle = 'inhale';
                            breathingAnimation();
                        }
                    }, 6000); // 6 segundos expirando
                }
            }

            function tick() {
                state.timeLeft--;
                updateTimerDisplay();
                if (state.timeLeft <= 0) {
                    stopBreathing(true);
                }
            }

            function updateTimerDisplay() {
                const minutes = Math.floor(state.timeLeft / 60);
                const seconds = state.timeLeft % 60;
                timerDisplay.textContent = `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
            }

            // --- Manipuladores de Eventos ---
            startStopBtn.addEventListener('click', async () => {
                if (!synth) {
                    try {
                        await Tone.start();
                        synth = new Tone.Synth().toDestination();
                        console.log("Contexto de áudio iniciado.");
                    } catch (e) {
                        console.error("Erro ao iniciar o áudio:", e);
                        showToast("Não foi possível habilitar o som.", "error");
                    }
                }

                if (state.isRunning) {
                    stopBreathing(false);
                } else {
                    startBreathing();
                }
            });

            durationButtons.forEach(button => {
                button.addEventListener('click', () => {
                    if (state.isRunning) return;
                    
                    state.sessionDuration = parseInt(button.dataset.duration);
                    state.timeLeft = state.sessionDuration;
                    updateTimerDisplay();

                    durationButtons.forEach(btn => {
                        btn.classList.remove('bg-sky-100', 'text-sky-800', 'ring-2', 'ring-sky-600');
                        btn.classList.add('bg-slate-200', 'text-slate-700');
                    });
                    button.classList.add('bg-sky-100', 'text-sky-800', 'ring-2', 'ring-sky-600');
                    button.classList.remove('bg-slate-200', 'text-slate-700');
                });
            });

            // --- Funções Utilitárias ---
            function showToast(message, type = 'info') {
                const toast = document.createElement('div');
                const bgColor = type === 'success' ? 'bg-green-500' : type === 'error' ? 'bg-red-500' : 'bg-blue-500';
                toast.className = `toast ${bgColor} text-white text-sm font-medium px-4 py-2 rounded-lg shadow-md mb-2`;
                toast.textContent = message;
                toastContainer.appendChild(toast);
                setTimeout(() => {
                    toast.remove();
                }, 5000);
            }

            // --- Inicialização ---
            updateTimerDisplay();
        });
    </script>
</body>
</html>
