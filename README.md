<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Plataforma de IA - Projetos Varejo</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Nunito', sans-serif; }
        .hidden { display: none; }
        .modal-enter { animation: fadeIn 0.3s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: scale(0.95); } to { opacity: 1; transform: scale(1); } }
        /* Efeito de cursor piscando para a caixa de resposta */
        .typing::after {
            content: '▋';
            animation: blink 1s step-end infinite;
            margin-left: 0.25rem;
            color: #10B981;
        }
        @keyframes blink {
            from, to { color: transparent; }
            50% { color: inherit; }
        }
        /* Custom scrollbar */
        #chat-history::-webkit-scrollbar { width: 6px; }
        #chat-history::-webkit-scrollbar-track { background: #F3F4F6; }
        #chat-history::-webkit-scrollbar-thumb { background: #D1D5DB; border-radius: 3px;}
        #chat-history::-webkit-scrollbar-thumb:hover { background: #9CA3AF; }
    </style>
</head>
<body class="bg-white text-gray-800 min-h-screen font-sans">

    <div id="app-container" class="h-screen flex flex-col">
        
        <header class="flex justify-between items-center p-4 border-b border-gray-200 bg-white flex-shrink-0">
            <div class="flex items-center space-x-3">
                <div class="p-2 bg-orange-100 rounded-full">
                    <svg class="w-6 h-6 text-orange-600" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z"></path></svg>
                </div>
                <h1 class="text-xl font-bold text-gray-800">IA Projetos Varejo</h1>
            </div>
            <div class="flex items-center space-x-4">
                <div class="flex items-center space-x-2">
                    <label for="doc-select" class="text-sm font-semibold text-gray-600">Projeto:</label>
                    <select id="doc-select" class="p-2 bg-gray-100 border-2 border-gray-200 rounded-lg focus:ring-2 focus:ring-orange-500 focus:border-orange-500"></select>
                </div>
                <button id="mode-toggle-button" class="bg-gray-800 hover:bg-gray-700 text-white font-semibold py-2 px-4 rounded-lg transition-colors duration-300 shadow-sm">
                    Admin
                </button>
            </div>
        </header>

        <main id="chat-history" class="flex-1 p-6 overflow-y-auto bg-gray-100">
            </main>

        <footer class="p-4 bg-white border-t border-gray-200 flex-shrink-0">
            <div class="flex items-center bg-gray-100 rounded-lg p-2">
                <input id="question" type="text" placeholder="Faça uma pergunta sobre o projeto selecionado..." class="w-full p-2 bg-transparent focus:outline-none">
                <button id="ask-button" class="p-2 bg-orange-500 hover:bg-orange-600 text-white rounded-full transition-colors duration-300 disabled:bg-gray-400">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"></path></svg>
                </button>
            </div>
        </footer>

    </div>

    <div id="modal-container" class="hidden fixed inset-0 bg-black bg-opacity-60 flex items-center justify-center p-4 z-50">
        <div id="modal-content" class="bg-white rounded-lg shadow-2xl w-full max-w-4xl modal-enter text-gray-800">
            </div>
    </div>


    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getFirestore, collection, onSnapshot, addDoc, deleteDoc, doc, query } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";

        const firebaseConfig = {
            apiKey: "AIzaSyCwRjgwJZe1_FK3-LRt2XSuaZippduH9gc",
            authDomain: "iafranquias20.firebaseapp.com",
            projectId: "iafranquias20",
            storageBucket: "iafranquias20.firebasestorage.app",
            messagingSenderId: "368055182817",
            appId: "1:368055182817:web:83e1560f97c1c39103dea0",
            measurementId: "G-KJE8VFLZBB"
        };

        const ADMIN_PASSWORD = "admin";
        
        let db, auth, docCollectionRef;
        let documents = [];
        let chatHistoryContainer = document.getElementById('chat-history');
        let isWaitingForFollowUp = false; 

        const docSelect = document.getElementById('doc-select');
        const askButton = document.getElementById('ask-button');
        const questionInput = document.getElementById('question');
        const modalContainer = document.getElementById('modal-container');
        const modalContent = document.getElementById('modal-content');

        // --- LÓGICA DO CHAT ---
        function addMessageToChat(sender, message) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `flex items-start gap-3 my-4 ${sender === 'user' ? 'justify-end' : ''}`;

            const iconDiv = document.createElement('div');
            iconDiv.className = `w-8 h-8 rounded-full flex-shrink-0 ${sender === 'user' ? 'bg-blue-500' : 'bg-orange-500'} flex items-center justify-center text-white font-bold`;
            iconDiv.textContent = sender === 'user' ? 'V' : 'IA';

            const textDiv = document.createElement('div');
            textDiv.className = `p-3 rounded-lg max-w-lg ${sender === 'user' ? 'bg-blue-100 text-gray-800' : 'bg-white text-gray-800 border'}`;
            textDiv.id = `msg-${Date.now()}`;

            if (sender === 'user') {
                messageDiv.appendChild(textDiv);
                messageDiv.appendChild(iconDiv);
            } else {
                messageDiv.appendChild(iconDiv);
                messageDiv.appendChild(textDiv);
            }

            chatHistoryContainer.appendChild(messageDiv);
            
            if (message) {
                const formattedMessage = message.replace(/\n/g, '<br>');
                typeWriterEffect(textDiv, formattedMessage);
            }

            chatHistoryContainer.scrollTop = chatHistoryContainer.scrollHeight;
            return textDiv;
        }

        function typeWriterEffect(element, text, speed = 15) {
            let i = 0;
            element.innerHTML = "";
            element.classList.add('typing');
            function type() {
                const nextChunk = text.substring(i);
                if (nextChunk.startsWith('<br>')) {
                    element.innerHTML += '<br>';
                    i += 4;
                } else {
                    element.innerHTML += text.charAt(i);
                    i++;
                }

                if (i < text.length) {
                    setTimeout(type, speed);
                } else {
                    element.classList.remove('typing');
                }
            }
            type();
        }

        // --- LÓGICA DOS MODAIS ---
        function showModal(htmlContent) { modalContent.innerHTML = htmlContent; modalContainer.classList.remove('hidden'); }
        function hideModal() { modalContainer.classList.add('hidden'); modalContent.innerHTML = ''; }
        
        function showAlert(message) {
            const html = `<div class="p-6"><h3 class="text-xl font-bold text-yellow-500 mb-4">Aviso</h3><p class="text-gray-700 mb-6">${message}</p><div class="flex justify-end"><button id="modal-ok-btn" class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg">OK</button></div></div>`;
            showModal(html);
            document.getElementById('modal-ok-btn').onclick = hideModal;
        }

        function showConfirm(message, onConfirm) {
            const html = `<div class="p-6"><h3 class="text-xl font-bold text-red-600 mb-4">Confirmação</h3><p class="text-gray-700 mb-6">${message}</p><div class="flex justify-end space-x-4"><button id="modal-cancel-btn" class="bg-gray-200 hover:bg-gray-300 font-bold py-2 px-4 rounded-lg">Cancelar</button><button id="modal-confirm-btn" class="bg-red-600 hover:bg-red-700 text-white font-bold py-2 px-4 rounded-lg">Confirmar</button></div></div>`;
            showModal(html);
            document.getElementById('modal-cancel-btn').onclick = hideModal;
            document.getElementById('modal-confirm-btn').onclick = () => {
                hideModal();
                onConfirm();
            };
        }
        
        function showAdminPanel() {
            const adminHTML = `
                <div class="p-6 bg-white rounded-lg">
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
                        <div>
                            <h2 class="text-2xl font-bold mb-4 text-gray-800">Adicionar Novo Projeto</h2>
                            <div class="space-y-4">
                                <input type="text" id="doc-title" placeholder="Título do Projeto" class="w-full p-3 bg-gray-50 border-2 border-gray-300 rounded-lg">
                                <textarea id="doc-content" rows="10" placeholder="Cole aqui o conteúdo..." class="w-full p-3 bg-gray-50 border-2 border-gray-300 rounded-lg"></textarea>
                                <button id="add-doc-button" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-lg">Salvar Projeto</button>
                            </div>
                        </div>
                        <div>
                            <div class="flex justify-between items-center mb-4">
                                <h2 class="text-2xl font-bold text-gray-800">Projetos Salvos</h2>
                                <button id="close-modal-btn" class="font-bold text-gray-500 hover:text-gray-800 text-2xl">&times;</button>
                            </div>
                            <ul id="doc-list" class="space-y-3 max-h-[22rem] overflow-y-auto pr-2"></ul>
                        </div>
                    </div>
                </div>
            `;
            showModal(adminHTML);
            document.getElementById('close-modal-btn').onclick = hideModal;
            renderDocList(document.getElementById('doc-list'));
            document.getElementById('add-doc-button').addEventListener('click', handleAddDocument);
            document.getElementById('doc-list').addEventListener('click', handleDeleteDocument);
        }

        function showAdminPasswordPrompt() {
            const html = `<div class="p-6"><h3 class="text-xl font-bold text-gray-800 mb-4">Acesso Restrito</h3><p class="text-gray-600 mb-4">Digite a senha de administrador.</p><input type="password" id="password-input" class="w-full p-3 bg-gray-50 border-2 border-gray-300 rounded-lg mb-6"><div class="flex justify-end space-x-4"><button id="modal-cancel-btn" class="bg-gray-200 hover:bg-gray-300 font-bold py-2 px-4 rounded-lg">Cancelar</button><button id="modal-login-btn" class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg">Entrar</button></div></div>`;
            showModal(html);
            const passInput = document.getElementById('password-input');
            passInput.focus();
            document.getElementById('modal-cancel-btn').onclick = hideModal;
            const loginAction = () => { if (passInput.value === ADMIN_PASSWORD) { hideModal(); showAdminPanel(); } else { showAlert('Senha incorreta!'); } };
            document.getElementById('modal-login-btn').onclick = loginAction;
            passInput.onkeyup = (e) => { if (e.key === 'Enter') loginAction(); };
        }

        // --- Funções de Renderização e Lógica ---
        function renderDocList(listElement) {
            const sortedDocuments = [...documents].sort((a, b) => a.title.localeCompare(b.title));
            listElement.innerHTML = ''; 
            docSelect.innerHTML = '';
            if (sortedDocuments.length === 0) {
                listElement.innerHTML = '<p class="text-gray-500">Nenhum projeto encontrado.</p>';
                docSelect.innerHTML = '<option disabled>Nenhum projeto disponível</option>';
                return;
            }
            sortedDocuments.forEach(doc => {
                const li = document.createElement('li');
                li.className = 'flex justify-between items-center bg-gray-100 p-3 rounded-lg border';
                li.innerHTML = `<span class="truncate pr-4">${doc.title}</span><button data-id="${doc.id}" class="delete-button text-red-500 hover:text-red-700 p-1 rounded-full text-xl">&times;</button>`;
                listElement.appendChild(li);
                const option = document.createElement('option');
                option.value = doc.id;
                option.textContent = doc.title;
                docSelect.appendChild(option);
            });
        }
        
        async function handleAddDocument() {
            const title = document.getElementById('doc-title').value.trim();
            const content = document.getElementById('doc-content').value.trim();
            if (!title || !content) { showAlert('Título e conteúdo são obrigatórios.'); return; }
            const button = document.getElementById('add-doc-button');
            button.disabled = true; button.textContent = 'Salvando...';
            try {
                await addDoc(docCollectionRef, { title, content, createdAt: new Date() });
                document.getElementById('doc-title').value = '';
                document.getElementById('doc-content').value = '';
            } catch (error) { console.error("Erro ao adicionar documento:", error); showAlert('Falha ao salvar o documento.'); } 
            finally { button.disabled = false; button.textContent = 'Salvar Projeto'; }
        }

        async function handleDeleteDocument(e) {
            if (e.target.classList.contains('delete-button')) {
                const id = e.target.getAttribute('data-id');
                showConfirm('Tem certeza que deseja excluir este projeto?', async () => {
                    try { 
                        await deleteDoc(doc(db, "documentos", id));
                    } catch (error) { 
                        console.error("Erro ao excluir:", error); 
                        showAlert('Falha ao excluir o projeto.'); 
                    }
                });
            }
        }
        
        document.getElementById('mode-toggle-button').addEventListener('click', showAdminPasswordPrompt);
        
        askButton.addEventListener('click', async () => {
            const question = questionInput.value.trim();
            if (!question) return;

            addMessageToChat('user', question);
            questionInput.value = '';

            const lowerCaseQuestion = question.toLowerCase().replace(/[?.,!]/g, ''); // Remove pontuação para ajudar

            if (lowerCaseQuestion.startsWith('quem é v') || lowerCaseQuestion.startsWith('quem e v')) {
                const whoAmIResponse = `Olá! Eu sou a IA do time de Projetos Varejo da Natura. Estou aqui para responder suas dúvidas sobre os projetos que estamos tocando atualmente: desde cronogramas e entregas até KPIs e status de implantação.
Posso te ajudar a:

• Verificar o andamento de cada iniciativa
• Encontrar documentos e relatórios atualizados
• Esclarecer prazos, responsáveis e próximos passos
• Apresentar indicadores de performance

Basta me perguntar sobre qualquer projeto, e eu trago a informação que você precisa na hora. Vamos começar? 😊`;
                addMessageToChat('ia', whoAmIResponse);
                return;
            }

            if (lowerCaseQuestion === 'oi', 'oi tudo bem?, 'opa', 'ola') {
                addMessageToChat('ia', 'Oi, tudo bem? Em que posso ajudar hoje?');
                return; 
            }

            // MUDANÇA AQUI: Condição mais flexível para capturar todas as variações
            const isGreetingResponse = lowerCaseQuestion.startsWith('estou bem e v') ||
                                       lowerCaseQuestion.startsWith('tudo bem e v') ||
                                       lowerCaseQuestion.startsWith('td bem e v') ||
                                       lowerCaseQuestion.startsWith('tudo e v');

            if (isGreetingResponse) {
                addMessageToChat('ia', 'Estou bem melhor agora falando com você. Como posso te ajudar hoje?');
                return;
            }

            const negativeResponses = ['não', 'nao', 'não obrigado', 'nao obrigado', 'não, obrigado', 'não obrigada', 'nao obrigada', 'não, obrigada', 'só isso'];
            if (isWaitingForFollowUp && negativeResponses.includes(lowerCaseQuestion)) {
                addMessageToChat('ia', 'Foi um prazer te ajudar, volte sempre!');
                isWaitingForFollowUp = false;
                return;
            }

            isWaitingForFollowUp = false; 
            const aiMessageBubble = addMessageToChat('ia', null);
            typeWriterEffect(aiMessageBubble, 'Pensando...');

            const selectedDocId = docSelect.value;
            const selectedDoc = documents.find(d => d.id === selectedDocId);
            if (!selectedDoc) {
                typeWriterEffect(aiMessageBubble, 'Por favor, selecione um projeto primeiro.');
                return;
            }

            const promptText = `Você é um assistente de extração de dados. Sua única função é responder perguntas usando APENAS o texto fornecido no documento. Você é proibido de usar qualquer conhecimento externo ou fazer suposições. REGRAS OBRIGATÓRIAS: 1. **ESTILO DIRETO:** Responda diretamente à pergunta. **NÃO** use frases introdutórias como "O documento informa..." ou "De acordo com o texto...". Vá direto ao ponto. 2. **FIDELIDADE TOTAL AO TEXTO:** A sua resposta deve ser 100% baseada no documento. Se o documento menciona um tópico (ex: "Piloto") mas não dá detalhes, a sua resposta deve ser "O documento menciona o tópico 'Piloto', mas não fornece mais detalhes sobre ele.". NUNCA invente detalhes que não estão no texto. 3. **RESPOSTA COMPLETA:** Se o documento contém múltiplos detalhes sobre a pergunta, sintetize todos eles numa resposta coesa e bem estruturada. 4. **SE NÃO ESTIVER LÁ, DIGA:** Se o tópico ou a resposta para a pergunta não existir DE TODO no documento, responda: "Não encontrei nenhuma informação sobre '${question}' no documento fornecido." --- DOCUMENTO --- Título: "${selectedDoc.title}" Conteúdo: ${selectedDoc.content} --- FIM DO DOCUMENTO --- --- PERGUNTA DO USUÁRIO --- ${question} --- FIM DA PERGUNTA ---`;

            try {
                const apiKey = firebaseConfig.apiKey;
                const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;
                const payload = { contents: [{ role: "user", parts: [{ text: promptText }] }] };
                const response = await fetch(apiUrl, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) });
                
                if (!response.ok) {
                    const errorBody = await response.json();
                    console.error("API Error Body:", errorBody);
                    throw new Error(`API Error: ${response.status} - ${errorBody.error?.message || 'Erro desconhecido'}`);
                }
                
                const result = await response.json();
                const text = result.candidates?.[0]?.content?.parts?.[0]?.text || "Não consegui processar a resposta.";
                typeWriterEffect(aiMessageBubble, text);

                setTimeout(() => {
                    addMessageToChat('ia', 'Posso ajudar com mais alguma coisa?');
                    isWaitingForFollowUp = true;
                }, 1000);

            } catch (error) {
                console.error("Erro ao chamar a IA:", error);
                typeWriterEffect(aiMessageBubble, `Ocorreu um erro técnico. Verifique a sua ligação ou as permissões da API.`);
            }
        });

        questionInput.addEventListener('keyup', (e) => { if (e.key === 'Enter') askButton.click(); });
        
        async function main() {
            try {
                await signInAnonymously(auth);
                onSnapshot(query(docCollectionRef), (snapshot) => {
                    documents = snapshot.docs.map(doc => ({...doc.data(), id: doc.id }));
                    const dummyList = document.createElement('ul');
                    renderDocList(dummyList);
                    const adminList = document.getElementById('doc-list');
                    if(adminList) {
                        renderDocList(adminList);
                    }
                }, (error) => console.error("Erro ao ler dados:", error) );
                
                addMessageToChat('ia', 'Olá! Sou o assistente de Projetos Varejo. Faça-me qualquer pergunta sobre o projeto selecionado.');

            } catch(error) {
                console.error("Erro na autenticação:", error);
                addMessageToChat('ia', 'Erro: Não foi possível autenticar com o serviço. Verifique as configurações do Firebase.');
            }
        }
        
        document.addEventListener('DOMContentLoaded', () => {
            try {
                if (!firebaseConfig.apiKey || !firebaseConfig.apiKey.startsWith("AIza")) {
                    throw new Error("Configuração da API Key do Firebase ausente ou inválida.");
                }
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                docCollectionRef = collection(db, "documentos");
                main();
            } catch (e) {
                console.error("ERRO CRÍTICO:", e);
                addMessageToChat('ia', `ERRO CRÍTICO: ${e.message}`);
            }
        });

    </script>
</body>
</html>
