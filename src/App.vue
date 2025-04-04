<template>
  <div class="modern-wizard">

    <div class="connection-status" :class="connectionStatus">
      {{
        connectionStatus === 'connected' ? 'Conectado' :
          connectionStatus === 'reconnecting' ? 'Reconectando...' :
            connectionStatus === 'error' ? 'Erro de conexão' : 'Desconectado'
      }}
    </div>

    <div v-if="error" class="error-message">
      {{ error }}
      <button @click="error = null" class="modern-btn secondary small">
        Tentar novamente
      </button>
    </div>

    <div class="steps-indicator">
      <div v-for="step in 3" :key="step"
        :class="['step', { 'active': currentStep >= step, 'completed': currentStep > step }]">
        <div class="step-number">{{ step }}</div>
        <div class="step-label">
          {{
            step === 1 ? 'Informações' :
              step === 2 ? 'Processamento' :
                'Download'
          }}
        </div>
      </div>
      <div class="progress-bar" :style="{ width: `${(currentStep - 1) * 50}%` }"></div>
    </div>

    <transition name="fade-slide" mode="out-in">
      <div v-if="currentStep === 1" class="wizard-step step-1">
        <h2 class="step-title">👋 Bem-vindo!</h2>
        <p class="step-subtitle">Por favor, informe seu nome para continuar</p>

        <div class="input-group">
          <input v-model="userName" type="text" placeholder="Seu nome completo" class="modern-input"
            @keyup.enter="goToNextStep">
          <span class="input-icon">👤</span>
        </div>

        <button @click="goToNextStep" :disabled="!userName.trim()" class="modern-btn primary">
          Continuar <span class="icon">→</span>
        </button>
      </div>
    </transition>

    <transition name="fade-slide" mode="out-in">
      <div v-if="currentStep === 2" class="wizard-step step-2">
        <h2 class="step-title">⚙️ {{ processingTitle }}</h2>
        <p class="step-subtitle">{{ statusMessage || 'Isso pode levar alguns instantes...' }}</p>

        <div class="progress-container">
          <div class="progress-track">
            <div class="progress-fill" :style="{ width: `${progress}%` }"></div>
          </div>
          <span class="progress-text">{{ progress }}%</span>
        </div>

        <div v-if="progress < 100" class="loading-animation">
          <div class="loading-dots">
            <div class="dot"></div>
            <div class="dot"></div>
            <div class="dot"></div>
          </div>
        </div>

        <div v-else class="completion-message">
          <p>Processamento concluído! Preparando seus arquivos...</p>
        </div>
      </div>
    </transition>

    <transition name="fade-slide" mode="out-in">
      <div v-if="currentStep === 3" class="wizard-step step-3">
        <h2 class="step-title">🎉 Pronto, {{ userName }}!</h2>
        <p class="step-subtitle">Seus arquivos estão disponíveis para download</p>

        <div class="download-cards">
          <div class="download-card" @click="downloadFile('zip1')">
            <div class="card-icon">📦</div>
            <h3>Download anexos I e II</h3>
            <p>Contém os anexos I e II em formato PDF</p>
            <button class="modern-btn download" :disabled="downloading">
              <span class="icon">↓</span>
              {{ downloading === 'zip1' ? 'Preparando...' : 'Baixar ZIP' }}
            </button>
          </div>

          <div class="download-card" @click="downloadFile('zip2')">
            <div class="card-icon">📊</div>
            <h3>Download tabela Rol</h3>
            <p>Contém a tabela Rol em formato CSV</p>
            <button class="modern-btn download" :disabled="downloading">
              <span class="icon">↓</span>
              {{ downloading === 'zip2' ? 'Preparando...' : 'Baixar ZIP' }}
            </button>
          </div>
        </div>

        <button @click="restartProcess" class="modern-btn secondary">
          <span class="icon">↻</span> Começar novamente
        </button>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const currentStep = ref(1);
const userName = ref('');
const connectionStatus = ref('connected');
const error = ref(null);
const progress = ref(0);
const processingTitle = ref('Processando seus arquivos...');
const statusMessage = ref('');
const downloading = ref(null);

const goToNextStep = async () => {
  if (currentStep.value === 1 && userName.value.trim()) {
    try {
      currentStep.value = 2;

      const response = await fetch('http://localhost:3001/api/v1/file/compress-csv-files-to-zip', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          name: userName.value.trim()
        })
      });

      const interval = setInterval(() => {
        if (progress.value < 90) {
          progress.value += 10;
        }
      }, 500);

      if (response.ok) {
        clearInterval(interval);
        progress.value = 100;
        statusMessage.value = 'Processamento concluído com sucesso!';

        setTimeout(() => {
          currentStep.value = 3;
        }, 1000);
      } else {
        throw new Error('Falha ao processar os arquivos');
      }
    } catch (err) {
      error.value = err.message;
      currentStep.value = 1;
      progress.value = 0;
    }
  }
};

const downloadFile = async (type) => {
  downloading.value = type;
  try {
    let url;

    if (type === 'zip1') {
      url = 'http://localhost:3001/api/v1/file/download/attachments';
    } else if (type === 'zip2') {
      url = 'http://localhost:3001/api/v1/file/download/csv?name=' + encodeURIComponent(userName.value.trim());
    }

    const response = await fetch(url);

    if (!response.ok) {
      throw new Error('Falha ao baixar os arquivos');
    }

    const contentDisposition = response.headers.get('Content-Disposition');
    console.log('Content-Disposition:', contentDisposition);

    let fileName = type === 'zip1' ? 'anexos_I_II.zip' : 'tabela_rol.zip';

    if (contentDisposition) {
      const fileNameMatch = contentDisposition.match(/filename\*?=['"]?(?:UTF-\d['"]*)?([^;\r\n"']*)['"]?;?/i);

      if (fileNameMatch && fileNameMatch[1]) {
        fileName = fileNameMatch[1].trim();
        console.log('Nome do arquivo extraído:', fileName);
      } else {
        const fallbackMatch = contentDisposition.split('filename=')[1];
        if (fallbackMatch) {
          fileName = fallbackMatch.replace(/['"]/g, '').trim();
        }
      }
    }

    const blob = await response.blob();
    const downloadUrl = window.URL.createObjectURL(blob);

    const a = document.createElement('a');
    a.href = downloadUrl;
    a.download = fileName;
    document.body.appendChild(a);
    a.click();

    window.URL.revokeObjectURL(downloadUrl);
    document.body.removeChild(a);

  } catch (err) {
    error.value = err.message;
  } finally {
    downloading.value = null;
  }
};


const restartProcess = () => {
  currentStep.value = 1;
  userName.value = '';
  progress.value = 0;
  statusMessage.value = '';
};
</script>

<style scoped>
.modern-wizard {
  max-width: 600px;
  margin: 2rem auto;
  padding: 2rem;
  background: white;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.08);
  font-family: 'Segoe UI', system-ui, sans-serif;
  position: relative;
}

.connection-status {
  position: fixed;
  bottom: 10px;
  right: 10px;
  padding: 5px 10px;
  border-radius: 3px;
  background-color: #ff6b6b;
  color: white;
  font-size: 0.8rem;
  z-index: 1000;
}

.connection-status.connected {
  background-color: #51cf66;
}

.connection-status.reconnecting {
  background-color: #fcc419;
}

.connection-status.error {
  background-color: #ff6b6b;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% {
    opacity: 1;
  }

  50% {
    opacity: 0.5;
  }

  100% {
    opacity: 1;
  }
}

.error-message {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: #FFEBEE;
  color: #C62828;
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  z-index: 100;
  display: flex;
  align-items: center;
  gap: 1rem;
  max-width: 90%;
}

.completion-message {
  color: #4CAF50;
  font-weight: 600;
  margin-top: 1rem;
  text-align: center;
}

.steps-indicator {
  display: flex;
  justify-content: space-between;
  position: relative;
  margin-bottom: 3rem;
  padding-bottom: 1.5rem;
}

.steps-indicator::before {
  content: '';
  position: absolute;
  top: 20px;
  left: 0;
  right: 0;
  height: 4px;
  background: #f0f0f0;
  z-index: 1;
  border-radius: 2px;
}

.progress-bar {
  position: absolute;
  top: 20px;
  left: 0;
  height: 4px;
  background: linear-gradient(90deg, #4361ee, #3a0ca3);
  z-index: 2;
  border-radius: 2px;
  transition: width 0.4s ease;
}

.step {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
  z-index: 3;
}

.step-number {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: #f0f0f0;
  color: #888;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  margin-bottom: 0.5rem;
  border: 3px solid #f0f0f0;
  transition: all 0.3s ease;
}

.step-label {
  font-size: 0.9rem;
  color: #888;
  font-weight: 500;
  text-align: center;
}

.step.active .step-number {
  background: white;
  border-color: #4361ee;
  color: #4361ee;
}

.step.completed .step-number {
  background: #4361ee;
  border-color: #4361ee;
  color: white;
}

.step.completed .step-label,
.step.active .step-label {
  color: #333;
}

.wizard-step {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  align-items: center;
}

.step-title {
  font-size: 1.8rem;
  font-weight: 700;
  color: #333;
  margin: 0;
  text-align: center;
}

.step-subtitle {
  font-size: 1rem;
  color: #666;
  text-align: center;
  max-width: 80%;
  margin: 0 auto 1rem;
}

.input-group {
  position: relative;
  width: 100%;
  max-width: 400px;
}

.modern-input {
  width: 100%;
  padding: 1rem 1rem 1rem 3rem;
  border: 2px solid #e0e0e0;
  border-radius: 12px;
  font-size: 1rem;
  transition: all 0.3s ease;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
}

.modern-input:focus {
  outline: none;
  border-color: #4361ee;
  box-shadow: 0 0 0 3px rgba(67, 97, 238, 0.2);
}

.input-icon {
  position: absolute;
  left: 1rem;
  top: 50%;
  transform: translateY(-50%);
  font-size: 1.2rem;
  color: #888;
}

.progress-container {
  width: 100%;
  max-width: 400px;
  margin: 1rem 0;
}

.progress-track {
  height: 8px;
  background: #f0f0f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #4361ee, #3a0ca3);
  border-radius: 4px;
  transition: width 0.3s ease;
}

.progress-text {
  display: block;
  text-align: center;
  font-weight: 600;
  color: #4361ee;
}

.loading-animation {
  margin: 2rem 0;
}

.loading-dots {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
}

.dot {
  width: 12px;
  height: 12px;
  background: #4361ee;
  border-radius: 50%;
  animation: bounce 1.4s infinite ease-in-out;
}

.dot:nth-child(1) {
  animation-delay: -0.32s;
}

.dot:nth-child(2) {
  animation-delay: -0.16s;
}

@keyframes bounce {

  0%,
  80%,
  100% {
    transform: translateY(0);
    opacity: 0.5;
  }

  40% {
    transform: translateY(-15px);
    opacity: 1;
  }
}

.download-cards {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1.5rem;
  width: 100%;
  margin: 1.5rem 0;
}

.download-card {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  text-align: center;
  border: 1px solid #e0e0e0;
  transition: all 0.3s ease;
  cursor: pointer;
}

.download-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
  border-color: #4361ee;
}

.card-icon {
  font-size: 2.5rem;
  margin-bottom: 1rem;
}

.download-card h3 {
  margin: 0 0 0.5rem;
  color: #333;
}

.download-card p {
  margin: 0 0 1.5rem;
  color: #666;
  font-size: 0.9rem;
}

.modern-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.8rem 1.8rem;
  border-radius: 12px;
  font-weight: 600;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
  border: none;
  gap: 0.5rem;
}

.modern-btn .icon {
  font-size: 1.2rem;
}

.primary {
  background: linear-gradient(135deg, #4361ee, #3a0ca3);
  color: white;
  box-shadow: 0 4px 10px rgba(67, 97, 238, 0.3);
}

.primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 15px rgba(67, 97, 238, 0.4);
}

.primary:disabled {
  background: #ccc;
  transform: none;
  box-shadow: none;
  cursor: not-allowed;
}

.secondary {
  background: white;
  color: #4361ee;
  border: 2px solid #e0e0e0;
}

.secondary:hover {
  border-color: #4361ee;
  background: #f8f9ff;
}

.secondary.small {
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
}

.download {
  background: white;
  color: #4361ee;
  border: 2px solid #e0e0e0;
  width: 100%;
}

.download:hover {
  background: #f8f9ff;
  border-color: #4361ee;
}

.download:disabled {
  opacity: 0.7;
  cursor: wait;
}

.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.5s ease;
}

.fade-slide-enter-from {
  opacity: 0;
  transform: translateX(30px);
}

.fade-slide-leave-to {
  opacity: 0;
  transform: translateX(-30px);
}

@media (max-width: 640px) {
  .modern-wizard {
    padding: 1.5rem;
    margin: 1rem;
  }

  .steps-indicator {
    margin-bottom: 2rem;
  }

  .step-label {
    font-size: 0.8rem;
  }

  .error-message {
    width: 90%;
    top: 10px;
    padding: 0.8rem;
  }
}
</style>