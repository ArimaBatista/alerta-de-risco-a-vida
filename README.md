# alerta-de-risco-a-vida
GitHub destinado ao desenvolvimento de um sistema automatizado de alerta para prevenção de riscos à vida, utilizando os conhecimentos adquiridos ao longo da disciplina de IoT da Unicamp, com o uso de Bluetooth, redes mesh e Wi-Fi.
##  Criando o server no UBUNTU

```
mkdir ~/servidor_esp32 && cd ~/servidor_esp32
python3 -m venv venv
source venv/bin/activate
pip install flask
```
criando o arquivo do site:
```
nano app.py
```
copie o codido para o arquivo criado
```
from flask import Flask, request, jsonify, render_template_string
import time

app = Flask(__name__)

# Dados recebidos do ESP32
dados_esp32 = {
    "leitura_ir": 0,
    "distancia_aprox": "Aguardando leitura...",
    "status_execucao": "Inativo",
    "ultima_atualizacao": "Nunca"
}

# Fila de comandos pendentes para o ESP32
fila_comandos = []

HTML_PAGE = """
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Controle ESP32 - IR & Fila de Ações</title>
    <style>
        body { font-family: sans-serif; max-width: 700px; margin: 30px auto; padding: 20px; line-height: 1.6; }
        .card { border: 1px solid #ccc; border-radius: 8px; padding: 15px; margin-bottom: 20px; box-shadow: 2px 2px 8px #eee; }
        label { display: block; margin-top: 8px; }
        input, select, button { padding: 8px; margin-top: 4px; font-size: 15px; }
        button { cursor: pointer; background: #007bff; color: white; border: none; border-radius: 4px; }
        button:hover { background: #0056b3; }
        ul { padding-left: 20px; }
    </style>
</head>
<body>
    <h1>Painel de Controle ESP32</h1>

    <div class="card">
        <h3>Telemetria do ESP32</h3>
        <p><strong>Reflexão IR (Diferencial):</strong> {{ dados.leitura_ir }} (0 a 4095)</p>
        <p><strong>Status de Proximidade:</strong> {{ dados.distancia_aprox }}</p>
        <p><strong>Executando no ESP32:</strong> {{ dados.status_execucao }}</p>
        <p><small>Última atualização: {{ dados.ultima_atualizacao }}</small></p>
    </div>

    <div class="card">
        <h3>Adicionar Comando à Fila</h3>
        <form method="POST" action="/adicionar_comando">
            <label>Ação / Comando:
                <select name="nome">
                    <option value="LIGAR_LED">LIGAR_LED (Prova de conceito)</option>
                    <option value="MOTOR_ATIVO">MOTOR_ATIVO (Comando genérico)</option>
                    <option value="ALARME_SONORO">ALARME_SONORO (Comando genérico)</option>
                </select>
            </label>
            <label>Duração ativa (segundos):
                <input type="number" name="duracao" value="5" min="1" max="60" required>
            </label>
            <br>
            <button type="submit">Enviar para Fila</button>
        </form>
    </div>

    <div class="card">
        <h3>Fila Pendente para o ESP32 ({{ fila|length }} itens)</h3>
        {% if fila %}
            <ul>
                {% for cmd in fila %}
                    <li><strong>{{ cmd.nome }}</strong> por {{ cmd.duracao }}s</li>
                {% endfor %}
            </ul>
        {% else %}
            <p>Nenhum comando na fila aguardando sincronização.</p>
        {% endif %}
    </div>
</body>
</html>
"""

@app.route("/", methods=["GET"])
def index():
    return render_template_string(HTML_PAGE, dados=dados_esp32, fila=fila_comandos)

@app.route("/adicionar_comando", methods=["POST"])
def adicionar_comando():
    nome = request.form.get("nome", "LIGAR_LED")
    duracao = int(request.form.get("duracao", 5))
    fila_comandos.append({"nome": nome, "duracao": duracao})
    return index()

@app.route("/api/sincronizar", methods=["POST"])
def api_sincronizar():
    global fila_comandos
    body = request.get_json(silent=True) or {}

    # Atualiza as métricas recebidas do ESP32
    if "leitura_ir" in body:
        val = body["leitura_ir"]
        dados_esp32["leitura_ir"] = val
        if val > 1500:
            dados_esp32["distancia_aprox"] = "Objeto Muito Próximo"
        elif val > 500:
            dados_esp32["distancia_aprox"] = "Objeto a Média Distância"
        else:
            dados_esp32["distancia_aprox"] = "Livre / Sem Objeto Próximo"

    if "status_execucao" in body:
        dados_esp32["status_execucao"] = body["status_execucao"]

    dados_esp32["ultima_atualizacao"] = time.strftime("%H:%M:%S")

    # Envia os comandos enfileirados e esvazia a fila
    comandos_para_enviar = list(fila_comandos)
    fila_comandos.clear()

    return jsonify({
        "status": "ok",
        "comandos": comandos_para_enviar
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
```
agora a main do esp32
```
#include <Arduino.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <vector>

// Credenciais de Wi-Fi e IP do seu computador Ubuntu
const char* ssid = "NOME_DO_SEU_WIFI";
const char* password = "SENHA_DO_SEU_WIFI";
const char* serverUrl = "http://SEU_IP:5000/api/sincronizar";

// Pinout
#define IR_EMITTER_PIN 32
#define IR_RECEIVER_PIN 33
#define LED_PLACA 2

// Estrutura para fila de comandos
struct Comando {
  String nome;
  int duracao_ms;
};

std::vector<Comando> filaComandos;
bool executandoAcao = false;
String comandoAtual = "Nenhum";
unsigned long tempoFimAcao = 0;

unsigned long ultimoEnvioHttp = 0;
const unsigned long intervaloHttp = 2000; // Sincroniza a cada 2s

// Função para medir IR com supressão de luz ambiente (leitura diferencial)
int lerDistanciaIR() {
  // 1. Leitura de luz ambiente (emissor apagado)
  digitalWrite(IR_EMITTER_PIN, LOW);
  delayMicroseconds(300);
  int leituraAmbiente = analogRead(IR_RECEIVER_PIN);

  // 2. Leitura com emissor aceso (reflexão + ambiente)
  digitalWrite(IR_EMITTER_PIN, HIGH);
  delayMicroseconds(300);
  int leituraComSinal = analogRead(IR_RECEIVER_PIN);

  digitalWrite(IR_EMITTER_PIN, LOW); // desliga após o pulso

  // Diferença: quanto maior, mais perto está o obstáculo
  int sinalRefletido = abs(leituraComSinal - leituraAmbiente);
  return sinalRefletido;
}

void processarFilaComandos() {
  unsigned long agora = millis();

  // Verifica se o comando atual terminou o tempo de execução
  if (executandoAcao && agora >= tempoFimAcao) {
    if (comandoAtual == "LIGAR_LED") {
      digitalWrite(LED_PLACA, LOW);
    }
    Serial.println("Comando finalizado: " + comandoAtual);
    executandoAcao = false;
    comandoAtual = "Nenhum";
  }

  // Se não tem ação rodando e há novos comandos na fila, inicia o próximo
  if (!executandoAcao && !filaComandos.empty()) {
    Comando proximo = filaComandos.front();
    filaComandos.erase(filaComandos.begin());

    comandoAtual = proximo.nome;
    tempoFimAcao = agora + proximo.duracao_ms;
    executandoAcao = true;

    Serial.println("Iniciando comando: " + comandoAtual + " por " + String(proximo.duracao_ms / 1000) + "s");

    if (comandoAtual == "LIGAR_LED") {
      digitalWrite(LED_PLACA, HIGH);
    }
  }
}

void sincronizarComServidor(int leituraIR) {
  if (WiFi.status() != WL_CONNECTED) return;

  HTTPClient http;
  http.begin(serverUrl);
  http.addHeader("Content-Type", "application/json");

  // Status de execução para enviar ao servidor
  String statusMsg = executandoAcao ? (comandoAtual + " (" + String((tempoFimAcao - millis()) / 1000) + "s restantes)") : "Pronto / Livre";

  // Monta JSON de envio
  JsonDocument docEnvio;
  docEnvio["leitura_ir"] = leituraIR;
  docEnvio["status_execucao"] = statusMsg;

  String payloadEnvio;
  serializeJson(docEnvio, payloadEnvio);

  int httpCode = http.POST(payloadEnvio);

  if (httpCode == HTTP_CODE_OK) {
    String resposta = http.getString();
    
    // Processa lista de comandos recebida
    JsonDocument docResposta;
    DeserializationError erro = deserializeJson(docResposta, resposta);
    if (!erro) {
      JsonArray lista = docResposta["comandos"].as<JsonArray>();
      for (JsonObject item : lista) {
        Comando c;
        c.nome = item["nome"].as<String>();
        c.duracao_ms = item["duracao"].as<int>() * 1000;
        filaComandos.push_back(c);
        Serial.println("Enfileirado: " + c.nome + " (" + String(c.duracao_ms / 1000) + "s)");
      }
    }
  } else {
    Serial.printf("Falha HTTP: %d\n", httpCode);
  }

  http.end();
}

void setup() {
  Serial.begin(115200);

  pinMode(LED_PLACA, OUTPUT);
  pinMode(IR_EMITTER_PIN, OUTPUT);
  pinMode(IR_RECEIVER_PIN, INPUT);

  digitalWrite(LED_PLACA, LOW);
  digitalWrite(IR_EMITTER_PIN, LOW);

  WiFi.begin(ssid, password);
  Serial.print("Conectando ao Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConectado! IP: " + WiFi.localIP().toString());
}

void loop() {
  // Gerencia o temporizador das ações em execução
  processarFilaComandos();

  // Envia telemetria e busca comandos na fila do servidor periodicamente
  if (millis() - ultimoEnvioHttp >= intervaloHttp) {
    ultimoEnvioHttp = millis();
    int irVal = lerDistanciaIR();
    sincronizarComServidor(irVal);
  }
}
```
agora o platformio.ini caso esteja usando
```
[env:esp32doit-devkit-v1]
platform = espressif32
board = esp32doit-devkit-v1
framework = arduino
monitor_speed = 115200
lib_deps =
    bblanchon/ArduinoJson @ ^7.0.0
```
