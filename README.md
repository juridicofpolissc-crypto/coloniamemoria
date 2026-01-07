<!doctype html>
<html lang="pt-BR" class="h-full">
 <head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Jogo da Memória - Cervejaria Colônia</title>
  <script src="/_sdk/data_sdk.js"></script>
  <script src="/_sdk/element_sdk.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="https://fonts.googleapis.com/css2?family=Bebas+Neue&amp;family=Quicksand:wght@400;600;700&amp;display=swap" rel="stylesheet">
  <style>
    body {
      box-sizing: border-box;
    }
    
    * {
      font-family: 'Quicksand', sans-serif;
    }
    
    h1, h2, .title-font {
      font-family: 'Bebas Neue', cursive;
      letter-spacing: 0.05em;
    }

    .card {
      perspective: 1000px;
      cursor: pointer;
      aspect-ratio: 1 / 1;
    }

    .card-inner {
      position: relative;
      width: 100%;
      height: 100%;
      transition: transform 0.6s;
      transform-style: preserve-3d;
    }

    .card.flipped .card-inner {
      transform: rotateY(180deg);
    }

    .card-front, .card-back {
      position: absolute;
      width: 100%;
      height: 100%;
      backface-visibility: hidden;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 12px;
      font-size: 3rem;
    }

    .card-back {
      transform: rotateY(180deg);
    }

    .card.matched {
      opacity: 0.4;
      pointer-events: none;
    }

    .sponsor-banner {
      transition: all 0.3s ease;
    }

    .sponsor-banner:hover {
      transform: scale(1.05);
    }
  </style>
  <style>@view-transition { navigation: auto; }</style>
 </head>
 <body class="h-full">
  <div id="app" class="w-full h-full overflow-auto"></div>
  <script>
    const defaultConfig = {
      background_color: "#1a0f0a",
      card_color: "#2d1810",
      accent_color: "#ff6b35",
      text_color: "#fff5e6",
      secondary_color: "#ffa500",
      titulo: "🍺 Jogo da Memória 🍖",
      subtitulo: "Teste sua memória com cerveja e churrasco!",
      botao_texto: "Participar do Jogo",
      mensagem_sucesso: "Inscrição realizada! Boa sorte no jogo!",
      font_family: "Quicksand",
      font_size: 16
    };

    let participants = [];
    let isSubmitting = false;
    let currentPlayer = null;
    let isAdminMode = false;
    let gameState = {
      cards: [],
      flippedCards: [],
      matchedPairs: 0,
      moves: 0,
      startTime: null
    };

    const cardSymbols = ['🍺', '🍖', '🥩', '🌭', '🍔', '🥓', '🍻', '🍗'];

    async function onConfigChange(config) {
      const app = document.getElementById('app');
      const baseSize = config.font_size || defaultConfig.font_size;
      const customFont = config.font_family || defaultConfig.font_family;
      const baseFontStack = 'Quicksand, Arial, sans-serif';
      const titleFontStack = 'Bebas Neue, Impact, sans-serif';
      
      app.style.backgroundColor = config.background_color || defaultConfig.background_color;
      app.style.color = config.text_color || defaultConfig.text_color;
      app.style.fontFamily = `${customFont}, ${baseFontStack}`;
      app.style.fontSize = `${baseSize}px`;
      
      const titleElement = document.getElementById('main-title');
      if (titleElement) {
        titleElement.textContent = config.titulo || defaultConfig.titulo;
        titleElement.style.color = config.accent_color || defaultConfig.accent_color;
        titleElement.style.fontFamily = `${customFont}, ${titleFontStack}`;
        titleElement.style.fontSize = `${baseSize * 2.5}px`;
      }
      
      const subtitleElement = document.getElementById('subtitle');
      if (subtitleElement) {
        subtitleElement.textContent = config.subtitulo || defaultConfig.subtitulo;
        subtitleElement.style.color = config.text_color || defaultConfig.text_color;
        subtitleElement.style.fontSize = `${baseSize * 1.25}px`;
      }

      const sponsorName = document.getElementById('sponsor-name');
      if (sponsorName) {
        sponsorName.style.color = config.accent_color || defaultConfig.accent_color;
        sponsorName.style.fontSize = `${baseSize * 1.5}px`;
      }

      const sponsorNameGame = document.getElementById('sponsor-name-game');
      if (sponsorNameGame) {
        sponsorNameGame.style.color = config.accent_color || defaultConfig.accent_color;
        sponsorNameGame.style.fontSize = `${baseSize * 1.25}px`;
      }
      
      const submitButton = document.getElementById('submit-btn');
      if (submitButton) {
        submitButton.textContent = config.botao_texto || defaultConfig.botao_texto;
        submitButton.style.backgroundColor = config.secondary_color || defaultConfig.secondary_color;
        submitButton.style.fontSize = `${baseSize * 1.125}px`;
      }

      const playButton = document.getElementById('play-btn');
      if (playButton) {
        playButton.style.backgroundColor = config.secondary_color || defaultConfig.secondary_color;
        playButton.style.fontSize = `${baseSize * 1.125}px`;
      }
      
      const formCard = document.querySelectorAll('.content-card');
      formCard.forEach(card => {
        card.style.backgroundColor = config.card_color || defaultConfig.card_color;
      });
      
      const labels = document.querySelectorAll('label');
      labels.forEach(label => {
        label.style.color = config.text_color || defaultConfig.text_color;
        label.style.fontSize = `${baseSize * 0.875}px`;
      });
      
      const inputs = document.querySelectorAll('input, textarea');
      inputs.forEach(input => {
        input.style.fontSize = `${baseSize}px`;
      });

      const gameCards = document.querySelectorAll('.card');
      gameCards.forEach(card => {
        const front = card.querySelector('.card-front');
        const back = card.querySelector('.card-back');
        if (front) {
          front.style.backgroundColor = config.card_color || defaultConfig.card_color;
          front.style.borderColor = config.accent_color || defaultConfig.accent_color;
        }
        if (back) {
          back.style.backgroundColor = config.secondary_color || defaultConfig.secondary_color;
        }
      });

      const statsElements = document.querySelectorAll('.stat-text');
      statsElements.forEach(stat => {
        stat.style.fontSize = `${baseSize}px`;
      });
    }

    function renderRegistrationForm() {
      const app = document.getElementById('app');
      app.innerHTML = `
        <div class="min-h-full flex items-center justify-center p-6">
          <div class="content-card w-full max-w-md rounded-2xl shadow-2xl p-8 space-y-6">
            <div class="text-center space-y-3">
              <h1 id="main-title" class="title-font tracking-wide">🍺 Jogo da Memória 🍖</h1>
              <p id="subtitle" class="font-semibold">Teste sua memória com cerveja e churrasco!</p>
              <div class="pt-3 pb-2">
                <p class="text-sm opacity-75 mb-3">Patrocinador Oficial</p>
                <a href="https://www.bebacolonia.com/cerveja-colonia" target="_blank" rel="noopener noreferrer" class="block">
                  <div class="sponsor-banner flex items-center justify-center gap-3 px-6 py-4 rounded-xl border-2 border-opacity-30 hover:border-opacity-60 cursor-pointer" style="border-color: #ffa500; background: linear-gradient(135deg, rgba(255, 165, 0, 0.1), rgba(255, 215, 0, 0.05));">
                    <svg width="50" height="50" viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
                      <!-- Copo -->
                      <path d="M 30 25 L 35 75 Q 35 80 50 80 Q 65 80 65 75 L 70 25 Z" fill="#FFA500" stroke="#FF8C00" stroke-width="2"/>
                      <!-- Cerveja -->
                      <path d="M 32 30 L 36 70 Q 36 74 50 74 Q 64 74 64 70 L 68 30 Z" fill="#FFD700"/>
                      <!-- Espuma -->
                      <ellipse cx="50" cy="25" rx="20" ry="8" fill="#FFFAF0"/>
                      <ellipse cx="45" cy="22" rx="8" ry="5" fill="#FFFFFF"/>
                      <ellipse cx="55" cy="23" rx="7" ry="4" fill="#FFFFFF"/>
                      <!-- Brilho no copo -->
                      <path d="M 38 35 L 40 65 Q 40 68 45 68 L 43 38 Z" fill="#FFFFFF" opacity="0.3"/>
                      <!-- Alça -->
                      <path d="M 70 35 Q 80 40 80 50 Q 80 60 70 65" stroke="#FF8C00" stroke-width="3" fill="none"/>
                    </svg>
                    <div class="text-left">
                      <p id="sponsor-name" class="text-2xl font-bold tracking-wider leading-tight" style="font-family: 'Bebas Neue', Impact, sans-serif;">CERVEJARIA COLÔNIA</p>
                      <p class="text-xs opacity-75 mt-1">Clique para conhecer</p>
                    </div>
                  </div>
                </a>
              </div>
            </div>
            
            <form id="registration-form" class="space-y-5">
              <div>
                <label for="nome" class="block font-semibold mb-2">Nome Completo</label>
                <input 
                  type="text" 
                  id="nome" 
                  name="nome" 
                  required 
                  class="w-full px-4 py-3 rounded-lg bg-white bg-opacity-10 border-2 border-white border-opacity-20 focus:border-opacity-50 focus:outline-none transition-all"
                  placeholder="Digite seu nome"
                >
              </div>
              
              <div>
                <label for="email" class="block font-semibold mb-2">E-mail</label>
                <input 
                  type="email" 
                  id="email" 
                  name="email" 
                  required 
                  class="w-full px-4 py-3 rounded-lg bg-white bg-opacity-10 border-2 border-white border-opacity-20 focus:border-opacity-50 focus:outline-none transition-all"
                  placeholder="seuemail@exemplo.com"
                >
              </div>
              
              <div>
                <label for="telefone" class="block font-semibold mb-2">Telefone</label>
                <input 
                  type="tel" 
                  id="telefone" 
                  name="telefone" 
                  required 
                  class="w-full px-4 py-3 rounded-lg bg-white bg-opacity-10 border-2 border-white border-opacity-20 focus:border-opacity-50 focus:outline-none transition-all"
                  placeholder="(00) 00000-0000"
                >
              </div>
              
              <div>
                <label for="endereco_envio" class="block font-semibold mb-2">Endereço Completo para Envio</label>
                <textarea 
                  id="endereco_envio" 
                  name="endereco_envio" 
                  required 
                  rows="3"
                  class="w-full px-4 py-3 rounded-lg bg-white bg-opacity-10 border-2 border-white border-opacity-20 focus:border-opacity-50 focus:outline-none transition-all resize-none"
                  placeholder="Rua, número, complemento, bairro, cidade, estado, CEP"
                ></textarea>
              </div>
              
              <button 
                type="submit" 
                id="submit-btn"
                class="w-full py-4 rounded-lg font-bold text-white shadow-lg hover:shadow-xl transform hover:scale-105 transition-all duration-200 uppercase tracking-wide"
              >
                Participar do Jogo
              </button>
            </form>
            
            <div id="success-message" class="hidden p-4 rounded-lg bg-green-600 bg-opacity-20 border-2 border-green-500 text-center">
              <p class="font-bold text-green-300">✅ Inscrição realizada! Boa sorte no jogo!</p>
            </div>

            <button 
              type="button"
              class="w-full py-2 rounded-lg font-semibold text-white bg-opacity-50 hover:bg-opacity-70 transition-all text-sm"
              style="background-color: #333;"
              onclick="showAdminLogin()"
            >
              👤 Acesso Administrador
            </button>
          </div>
        </div>
      `;
      
      const form = document.getElementById('registration-form');
      form.addEventListener('submit', handleSubmit);
      
      onConfigChange(window.elementSdk?.config || defaultConfig);
    }

    function renderGameScreen() {
      const app = document.getElementById('app');
      app.innerHTML = `
        <div class="min-h-full flex flex-col items-center justify-center p-6">
          <div class="content-card w-full max-w-4xl rounded-2xl shadow-2xl p-8 space-y-6">
            <div class="text-center space-y-3">
              <h1 id="main-title" class="title-font tracking-wide">🍺 Jogo da Memória 🍖</h1>
              <p id="subtitle" class="font-semibold">Bem-vindo, ${currentPlayer.nome}!</p>
              <div class="pt-2 pb-1">
                <a href="https://www.bebacolonia.com/cerveja-colonia" target="_blank" rel="noopener noreferrer" class="inline-block">
                  <div class="sponsor-banner flex items-center justify-center gap-2 px-4 py-2 rounded-lg border-2 border-opacity-20 hover:border-opacity-50 cursor-pointer" style="border-color: #ffa500; background: linear-gradient(135deg, rgba(255, 165, 0, 0.08), rgba(255, 215, 0, 0.03));">
                    <svg width="30" height="30" viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
                      <!-- Copo -->
                      <path d="M 30 25 L 35 75 Q 35 80 50 80 Q 65 80 65 75 L 70 25 Z" fill="#FFA500" stroke="#FF8C00" stroke-width="2"/>
                      <!-- Cerveja -->
                      <path d="M 32 30 L 36 70 Q 36 74 50 74 Q 64 74 64 70 L 68 30 Z" fill="#FFD700"/>
                      <!-- Espuma -->
                      <ellipse cx="50" cy="25" rx="20" ry="8" fill="#FFFAF0"/>
                      <ellipse cx="45" cy="22" rx="8" ry="5" fill="#FFFFFF"/>
                      <ellipse cx="55" cy="23" rx="7" ry="4" fill="#FFFFFF"/>
                      <!-- Brilho no copo -->
                      <path d="M 38 35 L 40 65 Q 40 68 45 68 L 43 38 Z" fill="#FFFFFF" opacity="0.3"/>
                      <!-- Alça -->
                      <path d="M 70 35 Q 80 40 80 50 Q 80 60 70 65" stroke="#FF8C00" stroke-width="3" fill="none"/>
                    </svg>
                    <p id="sponsor-name-game" class="text-xl font-bold tracking-wider" style="font-family: 'Bebas Neue', Impact, sans-serif;">CERVEJARIA COLÔNIA</p>
                  </div>
                </a>
              </div>
            </div>

            <div class="flex justify-around text-center mb-4">
              <div>
                <p class="stat-text font-bold">Movimentos</p>
                <p id="moves-count" class="text-2xl font-bold">${gameState.moves}</p>
              </div>
              <div>
                <p class="stat-text font-bold">Pares Encontrados</p>
                <p id="pairs-count" class="text-2xl font-bold">${gameState.matchedPairs}/8</p>
              </div>
              <div>
                <p class="stat-text font-bold">Pontuação</p>
                <p id="score-count" class="text-2xl font-bold" style="color: #ffa500;">${currentPlayer.pontuacao || 0}</p>
              </div>
            </div>
            
            <div id="game-board" class="grid grid-cols-4 gap-4 max-w-2xl mx-auto">
            </div>

            <button 
              id="play-btn"
              class="w-full py-4 rounded-lg font-bold text-white shadow-lg hover:shadow-xl transform hover:scale-105 transition-all duration-200 uppercase tracking-wide mt-6"
              onclick="resetGame()"
            >
              Novo Jogo
            </button>
          </div>
        </div>
      `;

      initializeGame();
      onConfigChange(window.elementSdk?.config || defaultConfig);
    }

    function initializeGame() {
      gameState.cards = [...cardSymbols, ...cardSymbols]
        .sort(() => Math.random() - 0.5)
        .map((symbol, index) => ({
          id: index,
          symbol: symbol,
          flipped: false,
          matched: false
        }));
      
      gameState.flippedCards = [];
      gameState.matchedPairs = 0;
      gameState.moves = 0;
      gameState.startTime = Date.now();

      renderGameBoard();
    }

    function renderGameBoard() {
      const board = document.getElementById('game-board');
      const config = window.elementSdk?.config || defaultConfig;
      
      board.innerHTML = gameState.cards.map(card => `
        <div class="card ${card.flipped ? 'flipped' : ''} ${card.matched ? 'matched' : ''}" data-id="${card.id}" onclick="flipCard(${card.id})">
          <div class="card-inner">
            <div class="card-front" style="background-color: ${config.card_color}; border: 3px solid ${config.accent_color};">
              ❓
            </div>
            <div class="card-back" style="background-color: ${config.secondary_color};">
              ${card.symbol}
            </div>
          </div>
        </div>
      `).join('');
    }

    window.flipCard = function(cardId) {
      if (gameState.flippedCards.length === 2) return;
      
      const card = gameState.cards[cardId];
      if (card.flipped || card.matched) return;

      card.flipped = true;
      gameState.flippedCards.push(card);
      
      const cardElement = document.querySelector(`[data-id="${cardId}"]`);
      cardElement.classList.add('flipped');

      if (gameState.flippedCards.length === 2) {
        gameState.moves++;
        document.getElementById('moves-count').textContent = gameState.moves;
        
        setTimeout(checkMatch, 1000);
      }
    };

    function checkMatch() {
      const [card1, card2] = gameState.flippedCards;

      if (card1.symbol === card2.symbol) {
        card1.matched = true;
        card2.matched = true;
        gameState.matchedPairs++;
        
        document.querySelector(`[data-id="${card1.id}"]`).classList.add('matched');
        document.querySelector(`[data-id="${card2.id}"]`).classList.add('matched');
        document.getElementById('pairs-count').textContent = `${gameState.matchedPairs}/8`;

        // Adiciona pontos ao jogador atual (100 pontos por par encontrado)
        if (currentPlayer) {
          currentPlayer.pontuacao = (currentPlayer.pontuacao || 0) + 100;
          
          // Atualiza a pontuação na tela
          const scoreElement = document.getElementById('score-count');
          if (scoreElement) {
            scoreElement.textContent = currentPlayer.pontuacao;
          }
          
          // Salva a pontuação atualizada no banco de dados imediatamente
          if (currentPlayer.__backendId) {
            window.dataSdk.update(currentPlayer).then(result => {
              if (result.isOk) {
                // Atualiza o localStorage com a nova pontuação
                localStorage.setItem('currentPlayer', JSON.stringify(currentPlayer));
              }
            });
          }
        }

        if (gameState.matchedPairs === 8) {
          setTimeout(() => endGame(), 500);
        }
      } else {
        card1.flipped = false;
        card2.flipped = false;
        
        document.querySelector(`[data-id="${card1.id}"]`).classList.remove('flipped');
        document.querySelector(`[data-id="${card2.id}"]`).classList.remove('flipped');
      }

      gameState.flippedCards = [];
    }

    async function endGame() {
      const timeElapsed = Math.floor((Date.now() - gameState.startTime) / 1000);

      // A pontuação já foi acumulada durante o jogo (100 pontos por par)
      currentPlayer.tempo_jogo = timeElapsed;

      const result = await window.dataSdk.update(currentPlayer);

      if (result.isOk) {
        // Atualiza o localStorage com o tempo final
        localStorage.setItem('currentPlayer', JSON.stringify(currentPlayer));
        showVictoryScreen(timeElapsed);
      }
    }

    function showVictoryScreen(timeElapsed) {
      const app = document.getElementById('app');
      app.innerHTML = `
        <div class="min-h-full flex items-center justify-center p-6">
          <div class="content-card w-full max-w-2xl rounded-2xl shadow-2xl p-8 space-y-6 text-center">
            <div class="text-6xl mb-4">🏆🎉</div>
            <h1 id="main-title" class="title-font tracking-wide text-5xl mb-4">PARABÉNS!</h1>
            
            <div class="space-y-3 mb-6">
              <p class="text-2xl font-bold">Você ganhou um</p>
              <p class="text-3xl font-bold" style="color: #ffa500;">BRINDE ESPECIAL</p>
              <p class="text-xl">da Cervejaria Colônia! 🎁</p>
            </div>

            <div class="flex justify-center my-6">
              <svg width="120" height="120" viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
                <!-- Copo -->
                <path d="M 30 25 L 35 75 Q 35 80 50 80 Q 65 80 65 75 L 70 25 Z" fill="#FFA500" stroke="#FF8C00" stroke-width="2"/>
                <!-- Cerveja -->
                <path d="M 32 30 L 36 70 Q 36 74 50 74 Q 64 74 64 70 L 68 30 Z" fill="#FFD700"/>
                <!-- Espuma -->
                <ellipse cx="50" cy="25" rx="20" ry="8" fill="#FFFAF0"/>
                <ellipse cx="45" cy="22" rx="8" ry="5" fill="#FFFFFF"/>
                <ellipse cx="55" cy="23" rx="7" ry="4" fill="#FFFFFF"/>
                <!-- Brilho no copo -->
                <path d="M 38 35 L 40 65 Q 40 68 45 68 L 43 38 Z" fill="#FFFFFF" opacity="0.3"/>
                <!-- Alça -->
                <path d="M 70 35 Q 80 40 80 50 Q 80 60 70 65" stroke="#FF8C00" stroke-width="3" fill="none"/>
              </svg>
            </div>

            <div class="p-6 rounded-lg" style="background: rgba(255, 165, 0, 0.1); border: 2px solid rgba(255, 165, 0, 0.3);">
              <p class="text-xl font-bold mb-3">📊 Suas Estatísticas:</p>
              <div class="space-y-2">
                <p><strong>Movimentos:</strong> ${gameState.moves}</p>
                <p><strong>Tempo:</strong> ${timeElapsed} segundos</p>
                <p><strong>Pontuação:</strong> ${currentPlayer.pontuacao}</p>
              </div>
            </div>

            <div class="p-4 rounded-lg" style="background: rgba(76, 175, 80, 0.1); border: 2px solid rgba(76, 175, 80, 0.3);">
              <p class="text-lg font-semibold">
                🎁 Confirme seus dados para retirar o brinde!
              </p>
            </div>

            <form id="winner-form" class="space-y-4 text-left">
              <div>
                <label for="winner-nome" class="block font-semibold mb-2">Nome Completo *</label>
                <input 
                  type="text" 
                  id="winner-nome" 
                  name="winner-nome" 
                  required 
                  value="${currentPlayer.nome}"
                  class="w-full px-4 py-3 rounded-lg bg-white bg-opacity-10 border-2 border-white border-opacity-20 focus:border-opacity-50 focus:outline-none transition-all"
                  placeholder="Digite seu nome completo"
                >
              </div>

              <div>
                <label for="winner-endereco" class="block font-semibold mb-2">Endereço Completo para Envio *</label>
                <textarea 
                  id="winner-endereco" 
                  name="winner-endereco" 
                  required 
                  rows="3"
                  class="w-full px-4 py-3 rounded-lg bg-white bg-opacity-10 border-2 border-white border-opacity-20 focus:border-opacity-50 focus:outline-none transition-all resize-none"
                  placeholder="Rua, número, complemento, bairro, cidade, estado, CEP"
                >${currentPlayer.endereco_envio || ''}</textarea>
              </div>

              <button 
                type="submit" 
                id="confirm-winner-btn"
                class="w-full py-4 rounded-lg font-bold text-white shadow-lg hover:shadow-xl transform hover:scale-105 transition-all duration-200 uppercase tracking-wide"
                style="background-color: #4CAF50;"
              >
                🎁 Resgatar Meu Brinde
              </button>
            </form>
          </div>
        </div>
      `;

      const winnerForm = document.getElementById('winner-form');
      winnerForm.addEventListener('submit', handleWinnerSubmit);

      onConfigChange(window.elementSdk?.config || defaultConfig);
    }

    async function handleWinnerSubmit(e) {
      e.preventDefault();
      
      const confirmBtn = document.getElementById('confirm-winner-btn');
      const originalText = confirmBtn.textContent;
      confirmBtn.textContent = 'Salvando...';
      confirmBtn.disabled = true;

      const formData = new FormData(e.target);
      const nomeCompleto = formData.get('winner-nome');
      const enderecoCompleto = formData.get('winner-endereco');

      currentPlayer.endereco_envio = enderecoCompleto;
      currentPlayer.nome = nomeCompleto;
      currentPlayer.premio_reivindicado = true;
      currentPlayer.data_reivindicacao = new Date().toISOString();

      const result = await window.dataSdk.update(currentPlayer);

      if (result.isOk) {
        // Atualiza o localStorage
        localStorage.setItem('currentPlayer', JSON.stringify(currentPlayer));
        showClaimSuccess();
      } else {
        confirmBtn.textContent = '❌ Erro! Tente novamente';
        confirmBtn.style.backgroundColor = '#e53e3e';
        setTimeout(() => {
          confirmBtn.textContent = originalText;
          confirmBtn.style.backgroundColor = '#4CAF50';
          confirmBtn.disabled = false;
        }, 3000);
      }
    }

    function showClaimSuccess() {
      const app = document.getElementById('app');
      app.innerHTML = `
        <div class="min-h-full flex items-center justify-center p-6">
          <div class="content-card w-full max-w-2xl rounded-2xl shadow-2xl p-8 space-y-6 text-center">
            <div class="text-6xl mb-4">✅🎉</div>
            <h1 id="main-title" class="title-font tracking-wide text-4xl mb-4">Brinde Confirmado!</h1>
            
            <div class="flex justify-center my-6">
              <svg width="120" height="120" viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
                <!-- Copo -->
                <path d="M 30 25 L 35 75 Q 35 80 50 80 Q 65 80 65 75 L 70 25 Z" fill="#FFA500" stroke="#FF8C00" stroke-width="2"/>
                <!-- Cerveja -->
                <path d="M 32 30 L 36 70 Q 36 74 50 74 Q 64 74 64 70 L 68 30 Z" fill="#FFD700"/>
                <!-- Espuma -->
                <ellipse cx="50" cy="25" rx="20" ry="8" fill="#FFFAF0"/>
                <ellipse cx="45" cy="22" rx="8" ry="5" fill="#FFFFFF"/>
                <ellipse cx="55" cy="23" rx="7" ry="4" fill="#FFFFFF"/>
                <!-- Brilho no copo -->
                <path d="M 38 35 L 40 65 Q 40 68 45 68 L 43 38 Z" fill="#FFFFFF" opacity="0.3"/>
                <!-- Alça -->
                <path d="M 70 35 Q 80 40 80 50 Q 80 60 70 65" stroke="#FF8C00" stroke-width="3" fill="none"/>
              </svg>
            </div>

            <div class="p-6 rounded-lg space-y-3" style="background: rgba(76, 175, 80, 0.15); border: 2px solid rgba(76, 175, 80, 0.5);">
              <p class="text-2xl font-bold" style="color: #4CAF50;">Seus dados foram registrados com sucesso!</p>
              <div class="mt-4 pt-4 border-t border-white border-opacity-20">
                <p class="font-bold mb-2">Dados Confirmados:</p>
                <div class="text-left space-y-1">
                  <p><strong>Nome:</strong> ${currentPlayer.nome}</p>
                  <p><strong>Email:</strong> ${currentPlayer.email}</p>
                  <p><strong>Telefone:</strong> ${currentPlayer.telefone}</p>
                  <p><strong>Endereço:</strong> ${currentPlayer.endereco_envio}</p>
                </div>
              </div>
            </div>

            <div class="p-4 rounded-lg" style="background: rgba(255, 165, 0, 0.1); border: 2px solid rgba(255, 165, 0, 0.3);">
              <p class="text-lg">
                🚚 <strong>Entraremos em contato em breve</strong><br>
                <span class="text-sm">para confirmar a entrega do seu brinde!</span>
              </p>
            </div>
          </div>
        </div>
      `;

      onConfigChange(window.elementSdk?.config || defaultConfig);
    }

    window.resetGame = function() {
      renderGameScreen();
    };

    window.showAdminLogin = function() {
      const app = document.getElementById('app');
      app.innerHTML = `
        <div class="min-h-full flex items-center justify-center p-6">
          <div class="content-card w-full max-w-md rounded-2xl shadow-2xl p-8 space-y-6">
            <div class="text-center space-y-3">
              <div class="text-5xl mb-3">🔐</div>
              <h1 id="main-title" class="title-font tracking-wide text-4xl">Acesso Restrito</h1>
              <p id="subtitle" class="text-lg">Digite a senha de administrador</p>
            </div>

            <form id="admin-login-form" class="space-y-5">
              <div>
                <label for="admin-password" class="block font-semibold mb-2">Senha</label>
                <input 
                  type="password" 
                  id="admin-password" 
                  name="admin-password" 
                  required 
                  class="w-full px-4 py-3 rounded-lg bg-white bg-opacity-10 border-2 border-white border-opacity-20 focus:border-opacity-50 focus:outline-none transition-all"
                  placeholder="Digite a senha"
                  autocomplete="off"
                >
              </div>
              
              <button 
                type="submit" 
                id="login-btn"
                class="w-full py-4 rounded-lg font-bold text-white shadow-lg hover:shadow-xl transform hover:scale-105 transition-all duration-200 uppercase tracking-wide"
                style="background-color: #4CAF50;"
              >
                🔓 Entrar no Painel
              </button>

              <button 
                type="button"
                class="w-full py-3 rounded-lg font-semibold text-white bg-opacity-50 hover:bg-opacity-70 transition-all"
                style="background-color: #666;"
                onclick="backToHome()"
              >
                ← Voltar
              </button>
            </form>

            <div id="login-error" class="hidden p-4 rounded-lg bg-red-600 bg-opacity-20 border-2 border-red-500 text-center">
              <p class="font-bold text-red-300">❌ Senha incorreta!</p>
            </div>
          </div>
        </div>
      `;

      const loginForm = document.getElementById('admin-login-form');
      loginForm.addEventListener('submit', handleAdminLogin);

      onConfigChange(window.elementSdk?.config || defaultConfig);
    };

    function handleAdminLogin(e) {
      e.preventDefault();
      
      const passwordInput = document.getElementById('admin-password');
      const password = passwordInput.value;
      const correctPassword = 'admin123';

      if (password === correctPassword) {
        isAdminMode = true;
        renderAdminPanel();
      } else {
        const errorDiv = document.getElementById('login-error');
        errorDiv.classList.remove('hidden');
        passwordInput.value = '';
        passwordInput.focus();

        setTimeout(() => {
          errorDiv.classList.add('hidden');
        }, 3000);
      }
    }

    function renderAdminPanel() {
      const app = document.getElementById('app');
      
      const pendingClaims = participants.filter(p => p.premio_reivindicado && p.endereco_envio);
      const allParticipants = participants.filter(p => p.nome);

      app.innerHTML = `
        <div class="min-h-full p-6">
          <div class="content-card w-full max-w-6xl mx-auto rounded-2xl shadow-2xl p-8 space-y-6">
            <div class="flex justify-between items-center mb-6">
              <div>
                <h1 id="main-title" class="title-font tracking-wide text-4xl">🔐 Painel Administrativo</h1>
                <p id="subtitle" class="mt-2">Gerenciamento de participantes e prêmios</p>
              </div>
              <button 
                onclick="backToHome()"
                class="px-6 py-3 rounded-lg font-semibold text-white hover:opacity-80 transition-all"
                style="background-color: #666;"
              >
                ← Voltar
              </button>
            </div>

            <div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
              <div class="p-6 rounded-lg text-center" style="background: rgba(76, 175, 80, 0.15); border: 2px solid rgba(76, 175, 80, 0.4);">
                <p class="text-3xl font-bold">${allParticipants.length}</p>
                <p class="text-sm opacity-75 mt-2">Total de Participantes</p>
              </div>
              <div class="p-6 rounded-lg text-center" style="background: rgba(255, 165, 0, 0.15); border: 2px solid rgba(255, 165, 0, 0.4);">
                <p class="text-3xl font-bold">${pendingClaims.length}</p>
                <p class="text-sm opacity-75 mt-2">Prêmios Reivindicados</p>
              </div>
              <div class="p-6 rounded-lg text-center" style="background: rgba(33, 150, 243, 0.15); border: 2px solid rgba(33, 150, 243, 0.4);">
                <p class="text-3xl font-bold">${participants.filter(p => p.tempo_jogo > 0).length}</p>
                <p class="text-sm opacity-75 mt-2">Jogos Completados</p>
              </div>
            </div>

            <div class="space-y-4">
              <div class="flex gap-4 mb-4">
                <button 
                  id="tab-claims"
                  onclick="showTab('claims')"
                  class="flex-1 py-3 rounded-lg font-bold transition-all"
                  style="background-color: #4CAF50;"
                >
                  🎁 Prêmios Reivindicados (${pendingClaims.length})
                </button>
                <button 
                  id="tab-all"
                  onclick="showTab('all')"
                  class="flex-1 py-3 rounded-lg font-bold transition-all"
                  style="background-color: #666;"
                >
                  📋 Todos Participantes (${allParticipants.length})
                </button>
              </div>

              <div id="admin-content">
              </div>
            </div>
          </div>
        </div>
      `;

      onConfigChange(window.elementSdk?.config || defaultConfig);
      showTab('claims');
    }

    window.showTab = function(tab) {
      const claimsBtn = document.getElementById('tab-claims');
      const allBtn = document.getElementById('tab-all');
      const content = document.getElementById('admin-content');

      if (tab === 'claims') {
        claimsBtn.style.backgroundColor = '#4CAF50';
        allBtn.style.backgroundColor = '#666';
        renderClaimsTable();
      } else {
        claimsBtn.style.backgroundColor = '#666';
        allBtn.style.backgroundColor = '#4CAF50';
        renderAllParticipantsTable();
      }
    };

    function renderClaimsTable() {
      const content = document.getElementById('admin-content');
      const pendingClaims = participants.filter(p => p.premio_reivindicado && p.endereco_envio);

      if (pendingClaims.length === 0) {
        content.innerHTML = `
          <div class="text-center p-12 rounded-lg" style="background: rgba(255, 255, 255, 0.05);">
            <p class="text-2xl opacity-75">📭 Nenhum prêmio reivindicado ainda</p>
          </div>
        `;
        return;
      }

      content.innerHTML = `
        <div class="space-y-4">
          ${pendingClaims.map((participant, index) => `
            <div class="p-6 rounded-lg" style="background: rgba(76, 175, 80, 0.1); border: 2px solid rgba(76, 175, 80, 0.3);">
              <div class="flex justify-between items-start mb-4">
                <div>
                  <h3 class="text-2xl font-bold mb-1">🏆 Prêmio #${index + 1}</h3>
                  <p class="text-sm opacity-75">Reivindicado em: ${new Date(participant.data_reivindicacao).toLocaleString('pt-BR')}</p>
                </div>
                <div class="text-right">
                  <p class="text-lg font-bold" style="color: #4CAF50;">✅ Confirmado</p>
                  <p class="text-lg font-bold" style="color: #ffa500;">⭐ ${participant.pontuacao || 0} pontos</p>
                </div>
              </div>
              
              <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                <div>
                  <p class="text-sm opacity-75 mb-1">Nome Completo:</p>
                  <p class="font-bold text-lg">${participant.nome}</p>
                </div>
                <div>
                  <p class="text-sm opacity-75 mb-1">E-mail:</p>
                  <p class="font-bold">${participant.email}</p>
                </div>
                <div>
                  <p class="text-sm opacity-75 mb-1">Telefone:</p>
                  <p class="font-bold">${participant.telefone}</p>
                </div>
                <div>
                  <p class="text-sm opacity-75 mb-1">Tempo de Jogo:</p>
                  <p class="font-bold">${participant.tempo_jogo} segundos</p>
                </div>
              </div>

              <div class="p-4 rounded-lg" style="background: rgba(255, 165, 0, 0.15); border: 2px solid rgba(255, 165, 0, 0.3);">
                <p class="text-sm opacity-75 mb-2">📦 Endereço para Envio Postal:</p>
                <p class="font-bold text-lg">${participant.endereco_envio}</p>
              </div>
            </div>
          `).join('')}
        </div>
      `;
    }

    function renderAllParticipantsTable() {
      const content = document.getElementById('admin-content');
      const allParticipants = participants.filter(p => p.nome);

      if (allParticipants.length === 0) {
        content.innerHTML = `
          <div class="text-center p-12 rounded-lg" style="background: rgba(255, 255, 255, 0.05);">
            <p class="text-2xl opacity-75">📭 Nenhum participante cadastrado ainda</p>
          </div>
        `;
        return;
      }

      const sortedParticipants = [...allParticipants].sort((a, b) => b.pontuacao - a.pontuacao);

      content.innerHTML = `
        <div class="overflow-x-auto">
          <table class="w-full">
            <thead>
              <tr style="background: rgba(255, 165, 0, 0.2); border-bottom: 2px solid rgba(255, 165, 0, 0.4);">
                <th class="p-3 text-left font-bold">#</th>
                <th class="p-3 text-left font-bold">Nome</th>
                <th class="p-3 text-left font-bold">E-mail</th>
                <th class="p-3 text-left font-bold">Telefone</th>
                <th class="p-3 text-left font-bold">Endereço</th>
                <th class="p-3 text-center font-bold">Pontuação</th>
                <th class="p-3 text-center font-bold">Tempo (s)</th>
                <th class="p-3 text-center font-bold">Prêmio</th>
              </tr>
            </thead>
            <tbody>
              ${sortedParticipants.map((participant, index) => `
                <tr style="border-bottom: 1px solid rgba(255, 255, 255, 0.1);">
                  <td class="p-3">${index + 1}</td>
                  <td class="p-3 font-semibold">${participant.nome}</td>
                  <td class="p-3">${participant.email}</td>
                  <td class="p-3">${participant.telefone}</td>
                  <td class="p-3 text-sm">${participant.endereco_envio || '-'}</td>
                  <td class="p-3 text-center font-bold">${participant.pontuacao || 0}</td>
                  <td class="p-3 text-center">${participant.tempo_jogo || '-'}</td>
                  <td class="p-3 text-center">
                    ${participant.premio_reivindicado 
                      ? '<span style="color: #4CAF50;">✅ Reivindicado</span>' 
                      : '<span style="color: #999;">-</span>'}
                  </td>
                </tr>
              `).join('')}
            </tbody>
          </table>
        </div>

        <div class="mt-6 p-4 rounded-lg text-center" style="background: rgba(33, 150, 243, 0.1); border: 2px solid rgba(33, 150, 243, 0.3);">
          <p class="text-sm">
            📊 <strong>Estatísticas:</strong> 
            ${allParticipants.length} participantes • 
            ${allParticipants.filter(p => p.tempo_jogo > 0).length} jogos completados • 
            ${allParticipants.filter(p => p.premio_reivindicado).length} prêmios reivindicados
          </p>
        </div>
      `;
    }

    window.backToHome = function() {
      isAdminMode = false;
      renderRegistrationForm();
    };

    async function handleSubmit(e) {
      e.preventDefault();
      
      if (isSubmitting) return;
      
      if (participants.length >= 999) {
        showMessage('Limite de 999 participantes atingido. Entre em contato com o organizador.', 'error');
        return;
      }
      
      isSubmitting = true;
      const submitBtn = document.getElementById('submit-btn');
      const originalText = submitBtn.textContent;
      submitBtn.textContent = 'Inscrevendo...';
      submitBtn.disabled = true;
      
      const formData = new FormData(e.target);
      const participant = {
        nome: formData.get('nome'),
        email: formData.get('email'),
        telefone: formData.get('telefone'),
        endereco_envio: formData.get('endereco_envio'),
        data_inscricao: new Date().toISOString(),
        pontuacao: 0,
        tempo_jogo: 0,
        premio_reivindicado: false
      };
      
      const result = await window.dataSdk.create(participant);
      
      if (result.isOk) {
        // Aguarda um momento para garantir que o participante foi criado e o onDataChanged foi chamado
        setTimeout(() => {
          // Encontra o participante criado na lista (agora com __backendId)
          const createdParticipant = participants.find(p => 
            p.email === participant.email && 
            p.nome === participant.nome
          );
          
          if (createdParticipant) {
            currentPlayer = createdParticipant;
            // Salva no localStorage
            localStorage.setItem('currentPlayer', JSON.stringify(createdParticipant));
          } else {
            currentPlayer = participant;
          }
          
          renderGameScreen();
        }, 1000);
      } else {
        showMessage('Erro ao realizar inscrição. Tente novamente.', 'error');
        submitBtn.textContent = originalText;
        submitBtn.disabled = false;
        isSubmitting = false;
      }
    }

    function showMessage(message, type) {
      const successDiv = document.getElementById('success-message');
      const messageElement = successDiv.querySelector('p');
      
      messageElement.textContent = type === 'error' ? `❌ ${message}` : `✅ ${message}`;
      successDiv.className = type === 'error' 
        ? 'p-4 rounded-lg bg-red-600 bg-opacity-20 border-2 border-red-500 text-center'
        : 'p-4 rounded-lg bg-green-600 bg-opacity-20 border-2 border-green-500 text-center';
      
      if (type === 'error') {
        messageElement.className = 'font-bold text-red-300';
      }
      
      successDiv.classList.remove('hidden');
      
      setTimeout(() => {
        successDiv.classList.add('hidden');
      }, 4000);
    }

    const dataHandler = {
      onDataChanged(data) {
        participants = data;
        if (currentPlayer && currentPlayer.__backendId) {
          const updated = data.find(p => p.__backendId === currentPlayer.__backendId);
          if (updated) {
            currentPlayer = updated;
          }
        }
        
        // Atualiza o contador de jogos completados no painel admin se estiver aberto
        if (isAdminMode) {
          const gamesCompletedElement = document.querySelector('.grid.grid-cols-1.md\\:grid-cols-3 > div:nth-child(3) .text-3xl');
          if (gamesCompletedElement) {
            const completedGames = participants.filter(p => p.tempo_jogo > 0).length;
            gamesCompletedElement.textContent = completedGames;
          }
        }
      }
    };

    async function initApp() {
      await window.dataSdk.init(dataHandler);
      
      // Restaura o jogador atual do localStorage se existir
      const savedPlayer = localStorage.getItem('currentPlayer');
      if (savedPlayer) {
        try {
          const playerData = JSON.parse(savedPlayer);
          // Verifica se o jogador ainda existe no banco de dados
          const existingPlayer = participants.find(p => p.__backendId === playerData.__backendId);
          if (existingPlayer) {
            currentPlayer = existingPlayer;
          } else {
            localStorage.removeItem('currentPlayer');
          }
        } catch (e) {
          localStorage.removeItem('currentPlayer');
        }
      }
      
      if (window.elementSdk) {
        window.elementSdk.init({
          defaultConfig,
          onConfigChange,
          mapToCapabilities: (config) => ({
            recolorables: [
              {
                get: () => config.background_color || defaultConfig.background_color,
                set: (value) => {
                  window.elementSdk.config.background_color = value;
                  window.elementSdk.setConfig({ background_color: value });
                }
              },
              {
                get: () => config.card_color || defaultConfig.card_color,
                set: (value) => {
                  window.elementSdk.config.card_color = value;
                  window.elementSdk.setConfig({ card_color: value });
                }
              },
              {
                get: () => config.text_color || defaultConfig.text_color,
                set: (value) => {
                  window.elementSdk.config.text_color = value;
                  window.elementSdk.setConfig({ text_color: value });
                }
              },
              {
                get: () => config.accent_color || defaultConfig.accent_color,
                set: (value) => {
                  window.elementSdk.config.accent_color = value;
                  window.elementSdk.setConfig({ accent_color: value });
                }
              },
              {
                get: () => config.secondary_color || defaultConfig.secondary_color,
                set: (value) => {
                  window.elementSdk.config.secondary_color = value;
                  window.elementSdk.setConfig({ secondary_color: value });
                }
              }
            ],
            borderables: [],
            fontEditable: {
              get: () => config.font_family || defaultConfig.font_family,
              set: (value) => {
                window.elementSdk.config.font_family = value;
                window.elementSdk.setConfig({ font_family: value });
              }
            },
            fontSizeable: {
              get: () => config.font_size || defaultConfig.font_size,
              set: (value) => {
                window.elementSdk.config.font_size = value;
                window.elementSdk.setConfig({ font_size: value });
              }
            }
          }),
          mapToEditPanelValues: (config) => new Map([
            ["titulo", config.titulo || defaultConfig.titulo],
            ["subtitulo", config.subtitulo || defaultConfig.subtitulo],
            ["botao_texto", config.botao_texto || defaultConfig.botao_texto],
            ["mensagem_sucesso", config.mensagem_sucesso || defaultConfig.mensagem_sucesso]
          ])
        });
      }
      
      renderRegistrationForm();
    }

    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', initApp);
    } else {
      initApp();
    }
  </script>
 <script>(function(){function c(){var b=a.contentDocument||a.contentWindow.document;if(b){var d=b.createElement('script');d.innerHTML="window.__CF$cv$params={r:'9ba687ade1ceb214',t:'MTc2NzgyMTE1MC4wMDAwMDA='};var a=document.createElement('script');a.nonce='';a.src='/cdn-cgi/challenge-platform/scripts/jsd/main.js';document.getElementsByTagName('head')[0].appendChild(a);";b.getElementsByTagName('head')[0].appendChild(d)}}if(document.body){var a=document.createElement('iframe');a.height=1;a.width=1;a.style.position='absolute';a.style.top=0;a.style.left=0;a.style.border='none';a.style.visibility='hidden';document.body.appendChild(a);if('loading'!==document.readyState)c();else if(window.addEventListener)document.addEventListener('DOMContentLoaded',c);else{var e=document.onreadystatechange||function(){};document.onreadystatechange=function(b){e(b);'loading'!==document.readyState&&(document.onreadystatechange=e,c())}}}})();</script></body>
</html>
