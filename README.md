 //autor: Claudio A. R. Betoni//
//--------------------------//

#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <LoRa.h>
#include <Preferences.h>
#include <time.h>

// ======================================================
// CONFIGURAÇÃO DO WI-FI
// ======================================================

const char* SSID_WIFI = "senha wifi";
const char* SENHA_WIFI = "nome wifi";

// ======================================================
// PINOS DO MÓDULO LORA SX1278
// =========================================

const uint8_t PINO_LORA_SCK  = 18;
const uint8_t PINO_LORA_MISO = 19;
const uint8_t PINO_LORA_MOSI = 23;
const uint8_t PINO_LORA_SS   = 5;
const uint8_t PINO_LORA_RST  = 14;
const uint8_t PINO_LORA_DIO0 = 26;

// Frequência do módulo SX1278
const long FREQUENCIA_LORA = 433E6;

// ======================================================
// CALIBRAÇÃO DA CAIXA D'ÁGUA
// ======================================

/*
  Distância entre o sensor e a água
  quando a caixa estiver cheia.
*/
const float DISTANCIA_CAIXA_CHEIA_CM = 20.0;

/*
  Distância entre o sensor e o fundo
  quando a caixa estiver vazia.
*/
const float DISTANCIA_CAIXA_VAZIA_CM = 70.0;

/*
  Capacidade total da caixa.

  Exemplo:
  500 litros
  1000 litros
  2000 litros
*/
const float CAPACIDADE_CAIXA_LITROS = 1000.0;

// ======================================================
// SERVIDOR WEB
// ========================================

WebServer servidor(80);

// ======================================================
// MEMÓRIA INTERNA
// ==========================================

Preferences memoria;

// ======================================================
// DADOS RECEBIDOS PELO LORA
// ==============================================

float distanciaAtualCm = 0.0;
float nivelAtualPercentual = 0.0;
float litrosAtuais = 0.0;

long numeroPacote = 0;

int rssiAtual = 0;
float snrAtual = 0.0;

unsigned long momentoUltimoPacote = 0;

bool recebeuPrimeiroPacote = false;

// ======================================================
// CONFIGURAÇÃO DA COMUNICAÇÃO
// ===========================================

/*
  Se o receptor ficar mais de 10 segundos
  sem receber dados, o site mostrará
  comunicação interrompida.
*/
const unsigned long TEMPO_SEM_COMUNICACAO_MS = 10000;

// ======================================================
// HISTÓRICO SEMANAL
// ========================================

/*
  Uma semana possui:

  7 dias × 24 horas = 168 horas
*/
const int TOTAL_HORAS_HISTORICO = 168;

/*
  O cálculo do consumo será feito
  a cada cinco minutos.
*/
const unsigned long INTERVALO_CONSUMO_MS = 300000;

/*
  Diferenças menores que 0,20% serão
  tratadas como oscilação do sensor.
*/
const float CONSUMO_MINIMO_PERCENTUAL = 0.20;

/*
  Uma queda maior que 20% em apenas
  cinco minutos será tratada como
  leitura suspeita.
*/
const float CONSUMO_MAXIMO_PERCENTUAL = 20.0;

// ======================================================
// ESTRUTURA DO HISTÓRICO
// ========================================

struct RegistroHora {
    uint32_t horaEpoch;
    float consumoPercentual;
};

RegistroHora historico[TOTAL_HORAS_HISTORICO];

// Nível usado como referência para calcular o consumo
float nivelReferenciaConsumo = -1.0;

unsigned long momentoUltimoCalculo = 0;
unsigned long momentoUltimaGravacao = 0;

bool horarioSincronizado = false;

// ======================================================
// PÁGINA HTML
// =======================================

const char PAGINA_HTML[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="pt-BR">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Monitoramento da Caixa d'Água</title>

    <style>
        * {
            box-sizing: border-box;
        }

        body {
            margin: 0;
            padding: 20px;
            font-family: Arial, Helvetica, sans-serif;
            background: #eef5f9;
            color: #263238;
        }

        .pagina {
            width: min(100%, 1050px);
            margin: 0 auto;
        }

        h1 {
            margin-top: 5px;
            margin-bottom: 8px;
            text-align: center;
            color: #126ca8;
        }

        .subtitulo {
            margin-top: 0;
            margin-bottom: 25px;
            text-align: center;
            color: #607d8b;
        }

        .grade-principal {
            display: grid;
            grid-template-columns:
                repeat(auto-fit, minmax(220px, 1fr));
            gap: 15px;
            margin-bottom: 20px;
        }

        .cartao {
            background: #ffffff;
            border-radius: 16px;
            padding: 20px;
            text-align: center;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.07);
        }

        .cartao span {
            display: block;
            margin-bottom: 8px;
            font-size: 14px;
            color: #607d8b;
        }

        .cartao strong {
            display: block;
            font-size: 27px;
            color: #126ca8;
        }

        .cartao small {
            display: block;
            margin-top: 6px;
            color: #78909c;
        }

        .painel-nivel {
            display: grid;
            grid-template-columns: 220px 1fr;
            gap: 25px;
            align-items: center;
            margin-bottom: 20px;
            padding: 25px;
            background: #ffffff;
            border-radius: 18px;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.07);
        }

        .reservatorio {
            position: relative;
            width: 150px;
            height: 260px;
            margin: auto;
            overflow: hidden;
            border: 7px solid #455a64;
            border-radius: 25px 25px 35px 35px;
            background: #e9f2f7;
        }

        .agua {
            position: absolute;
            left: 0;
            right: 0;
            bottom: 0;
            height: 0%;
            background:
                linear-gradient(
                    180deg,
                    #55c7ff 0%,
                    #168bd2 100%
                );
            transition: height 1s ease;
        }

        .onda {
            position: absolute;
            top: -8px;
            left: -10%;
            width: 120%;
            height: 18px;
            border-radius: 50%;
            background: rgba(255, 255, 255, 0.35);
        }

        .nivel-centro {
            position: absolute;
            z-index: 2;
            top: 50%;
            left: 50%;
            width: 100%;
            transform: translate(-50%, -50%);
            text-align: center;
            font-size: 31px;
            font-weight: bold;
            color: #ffffff;
            text-shadow: 0 1px 4px rgba(0, 0, 0, 0.65);
        }

        .informacoes-nivel h2 {
            margin-top: 0;
            color: #126ca8;
        }

        .barra {
            width: 100%;
            height: 25px;
            overflow: hidden;
            border-radius: 20px;
            background: #dfe9ee;
        }

        .barra-preenchimento {
            width: 0%;
            height: 100%;
            border-radius: 20px;
            background:
                linear-gradient(
                    90deg,
                    #168bd2,
                    #4fc3f7
                );
            transition: width 1s ease;
        }

        .situacao {
            display: inline-block;
            margin-top: 15px;
            padding: 8px 15px;
            border-radius: 20px;
            font-weight: bold;
        }

        .situacao-online {
            color: #136b2c;
            background: #d9f7df;
        }

        .situacao-offline {
            color: #a52727;
            background: #ffdede;
        }

        .historico {
            margin-top: 25px;
        }

        .historico h2 {
            text-align: center;
            color: #126ca8;
        }

        .resumo-historico {
            display: grid;
            grid-template-columns:
                repeat(auto-fit, minmax(210px, 1fr));
            gap: 14px;
            margin-bottom: 20px;
        }

        .cartao-resumo {
            padding: 18px;
            border-radius: 15px;
            background: #ffffff;
            text-align: center;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.07);
        }

        .cartao-resumo span {
            display: block;
            margin-bottom: 8px;
            font-size: 14px;
            color: #607d8b;
        }

        .cartao-resumo strong {
            display: block;
            font-size: 21px;
            color: #126ca8;
        }

        .cartao-resumo small {
            display: block;
            margin-top: 6px;
            color: #607d8b;
        }

        .grafico-container {
            margin-bottom: 20px;
            padding: 20px;
            border-radius: 16px;
            background: #ffffff;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.07);
        }

        .grafico-container h3 {
            margin-top: 0;
            text-align: center;
            color: #37474f;
        }

        canvas {
            display: block;
            width: 100%;
            height: 310px;
        }

        .observacao {
            padding: 15px;
            border-left: 5px solid #168bd2;
            border-radius: 8px;
            background: #ffffff;
            color: #546e7a;
        }

        @media (max-width: 680px) {
            body {
                padding: 12px;
            }

            .painel-nivel {
                grid-template-columns: 1fr;
            }

            .reservatorio {
                width: 135px;
                height: 230px;
            }

            canvas {
                height: 260px;
            }
        }
    </style>
</head>

<body>

<div class="pagina">

    <h1>Monitoramento da Caixa d'Água</h1>

    <p class="subtitulo">
        Sistema ESP32, sensor ultrassônico e comunicação LoRa
    </p>

    <section class="painel-nivel">

        <div class="reservatorio">

            <div class="agua" id="agua">
                <div class="onda"></div>
            </div>

            <div class="nivel-centro" id="nivelCentro">
                0%
            </div>

        </div>

        <div class="informacoes-nivel">

            <h2>Nível atual</h2>

            <div class="barra">
                <div
                    class="barra-preenchimento"
                    id="barraNivel"
                ></div>
            </div>

            <p>
                A caixa está com
                <strong id="nivelTexto">0%</strong>
                da capacidade.
            </p>

            <div
                class="situacao situacao-offline"
                id="situacaoComunicacao"
            >
                Aguardando dados
            </div>

        </div>

    </section>

    <section class="grade-principal">

        <div class="cartao">
            <span>Distância medida</span>
            <strong id="distancia">-- cm</strong>
        </div>

        <div class="cartao">
            <span>Quantidade aproximada</span>
            <strong id="litros">-- litros</strong>
        </div>

        <div class="cartao">
            <span>Potência do sinal</span>
            <strong id="rssi">-- dBm</strong>
        </div>

        <div class="cartao">
            <span>Qualidade SNR</span>
            <strong id="snr">-- dB</strong>
        </div>

        <div class="cartao">
            <span>Número do pacote</span>
            <strong id="pacote">--</strong>
        </div>

        <div class="cartao">
            <span>Último recebimento</span>
            <strong id="ultimoPacote">--</strong>
        </div>

    </section>

    <section class="historico">

        <h2>Consumo dos últimos 7 dias</h2>

        <div class="resumo-historico">

            <div class="cartao-resumo">
                <span>Dia de maior consumo</span>
                <strong id="maiorDia">--</strong>
                <small id="maiorDiaValor">--</small>
            </div>

            <div class="cartao-resumo">
                <span>Dia de menor consumo</span>
                <strong id="menorDia">--</strong>
                <small id="menorDiaValor">--</small>
            </div>

            <div class="cartao-resumo">
                <span>Horário de maior consumo</span>
                <strong id="maiorHorario">--</strong>
                <small id="maiorHorarioValor">--</small>
            </div>

            <div class="cartao-resumo">
                <span>Horário de menor consumo</span>
                <strong id="menorHorario">--</strong>
                <small id="menorHorarioValor">--</small>
            </div>

        </div>

        <div class="grafico-container">
            <h3>Consumo diário</h3>
            <canvas id="graficoDias"></canvas>
        </div>

        <div class="grafico-container">
            <h3>Consumo por faixa de horário</h3>
            <canvas id="graficoHorarios"></canvas>
        </div>

    </section>

    <div class="observacao">
        O consumo é estimado pela redução do nível da água.
        Quando a caixa é abastecida e o nível aumenta, esse
        aumento não é contabilizado como consumo.
    </div>

</div>

<script>
    let historicoRecebido = null;

    function limitar(valor, minimo, maximo) {
        return Math.min(
            Math.max(valor, minimo),
            maximo
        );
    }

    function formatarHorario(hora) {
        const inicio =
            String(hora).padStart(2, "0") + ":00";

        const fim =
            String((hora + 1) % 24).padStart(2, "0") +
            ":00";

        return inicio + " às " + fim;
    }

    async function atualizarDadosAtuais() {
        try {
            const resposta = await fetch(
                "/dados?t=" + Date.now()
            );

            const dados = await resposta.json();

            const nivel = limitar(
                Number(dados.nivel),
                0,
                100
            );

            document.getElementById(
                "agua"
            ).style.height = nivel + "%";

            document.getElementById(
                "barraNivel"
            ).style.width = nivel + "%";

            document.getElementById(
                "nivelCentro"
            ).textContent = nivel.toFixed(1) + "%";

            document.getElementById(
                "nivelTexto"
            ).textContent = nivel.toFixed(1) + "%";

            document.getElementById(
                "distancia"
            ).textContent =
                Number(dados.distancia).toFixed(1) +
                " cm";

            document.getElementById(
                "litros"
            ).textContent =
                Number(dados.litros).toFixed(0) +
                " litros";

            document.getElementById(
                "rssi"
            ).textContent =
                dados.rssi + " dBm";

            document.getElementById(
                "snr"
            ).textContent =
                Number(dados.snr).toFixed(1) +
                " dB";

            document.getElementById(
                "pacote"
            ).textContent = dados.pacote;

            document.getElementById(
                "ultimoPacote"
            ).textContent =
                dados.tempoSemPacote + " s";

            const situacao =
                document.getElementById(
                    "situacaoComunicacao"
                );

            if (dados.comunicacao) {
                situacao.textContent =
                    "Comunicação normal";

                situacao.className =
                    "situacao situacao-online";
            } else {
                situacao.textContent =
                    "Comunicação interrompida";

                situacao.className =
                    "situacao situacao-offline";
            }

        } catch (erro) {
            console.error(
                "Erro ao carregar dados:",
                erro
            );

            const situacao =
                document.getElementById(
                    "situacaoComunicacao"
                );

            situacao.textContent =
                "Falha ao acessar o ESP32";

            situacao.className =
                "situacao situacao-offline";
        }
    }

    function desenharGrafico(
        canvasId,
        rotulos,
        valores,
        tipo
    ) {
        const canvas =
            document.getElementById(canvasId);

        const contexto =
            canvas.getContext("2d");

        const larguraCSS =
            canvas.clientWidth;

        const alturaCSS =
            canvas.clientHeight;

        const escala =
            window.devicePixelRatio || 1;

        canvas.width =
            larguraCSS * escala;

        canvas.height =
            alturaCSS * escala;

        contexto.setTransform(
            escala,
            0,
            0,
            escala,
            0,
            0
        );

        const largura = larguraCSS;
        const altura = alturaCSS;

        const margemEsquerda = 54;
        const margemDireita = 16;
        const margemSuperior = 30;
        const margemInferior = 52;

        const larguraUtil =
            largura -
            margemEsquerda -
            margemDireita;

        const alturaUtil =
            altura -
            margemSuperior -
            margemInferior;

        contexto.clearRect(
            0,
            0,
            largura,
            altura
        );

        let maiorValor =
            Math.max(...valores);

        if (
            !Number.isFinite(maiorValor) ||
            maiorValor <= 0
        ) {
            maiorValor = 1;
        }

        maiorValor *= 1.15;

        contexto.font = "12px Arial";
        contexto.textBaseline = "middle";

        for (let linha = 0; linha <= 5; linha++) {
            const percentual = linha / 5;

            const y =
                margemSuperior +
                alturaUtil -
                percentual * alturaUtil;

            const valorEixo =
                maiorValor * percentual;

            contexto.strokeStyle = "#dce6eb";
            contexto.lineWidth = 1;

            contexto.beginPath();

            contexto.moveTo(
                margemEsquerda,
                y
            );

            contexto.lineTo(
                largura - margemDireita,
                y
            );

            contexto.stroke();

            contexto.fillStyle = "#607d8b";
            contexto.textAlign = "right";

            contexto.fillText(
                valorEixo.toFixed(1) + "%",
                margemEsquerda - 7,
                y
            );
        }

        const espaco =
            larguraUtil / valores.length;

        if (tipo === "linha") {
            contexto.strokeStyle = "#168bd2";
            contexto.lineWidth = 3;
            contexto.lineJoin = "round";

            contexto.beginPath();

            valores.forEach(
                (valor, indice) => {
                    const x =
                        margemEsquerda +
                        indice * espaco +
                        espaco / 2;

                    const y =
                        margemSuperior +
                        alturaUtil -
                        (valor / maiorValor) *
                        alturaUtil;

                    if (indice === 0) {
                        contexto.moveTo(x, y);
                    } else {
                        contexto.lineTo(x, y);
                    }
                }
            );

            contexto.stroke();

            valores.forEach(
                (valor, indice) => {
                    const x =
                        margemEsquerda +
                        indice * espaco +
                        espaco / 2;

                    const y =
                        margemSuperior +
                        alturaUtil -
                        (valor / maiorValor) *
                        alturaUtil;

                    contexto.fillStyle = "#168bd2";

                    contexto.beginPath();

                    contexto.arc(
                        x,
                        y,
                        5,
                        0,
                        Math.PI * 2
                    );

                    contexto.fill();

                    contexto.fillStyle = "#263238";
                    contexto.textAlign = "center";
                    contexto.textBaseline = "bottom";
                    contexto.font = "11px Arial";

                    contexto.fillText(
                        valor.toFixed(1) + "%",
                        x,
                        y - 8
                    );
                }
            );
        } else {
            valores.forEach(
                (valor, indice) => {
                    const larguraBarra =
                        Math.max(
                            3,
                            espaco * 0.65
                        );

                    const alturaBarra =
                        (valor / maiorValor) *
                        alturaUtil;

                    const x =
                        margemEsquerda +
                        indice * espaco +
                        (
                            espaco -
                            larguraBarra
                        ) / 2;

                    const y =
                        margemSuperior +
                        alturaUtil -
                        alturaBarra;

                    contexto.fillStyle = "#168bd2";

                    contexto.fillRect(
                        x,
                        y,
                        larguraBarra,
                        alturaBarra
                    );
                }
            );
        }

        contexto.fillStyle = "#455a64";
        contexto.font = "11px Arial";
        contexto.textAlign = "center";
        contexto.textBaseline = "top";

        rotulos.forEach(
            (rotulo, indice) => {
                if (
                    rotulos.length > 12 &&
                    indice % 2 !== 0
                ) {
                    return;
                }

                const x =
                    margemEsquerda +
                    indice * espaco +
                    espaco / 2;

                contexto.fillText(
                    rotulo,
                    x,
                    margemSuperior +
                    alturaUtil +
                    12
                );
            }
        );
    }

    async function atualizarHistorico() {
        try {
            const resposta = await fetch(
                "/historico?t=" + Date.now()
            );

            const dados = await resposta.json();

            historicoRecebido = dados;

            const rotulosHorarios = [];

            for (
                let hora = 0;
                hora < 24;
                hora++
            ) {
                rotulosHorarios.push(
                    String(hora).padStart(2, "0") +
                    "h"
                );
            }

            desenharGrafico(
                "graficoDias",
                dados.dias,
                dados.consumoDias,
                "linha"
            );

            desenharGrafico(
                "graficoHorarios",
                rotulosHorarios,
                dados.consumoHorarios,
                "barra"
            );

            document.getElementById(
                "maiorDia"
            ).textContent =
                dados.maiorDia;

            document.getElementById(
                "maiorDiaValor"
            ).textContent =
                Number(
                    dados.maiorDiaValor
                ).toFixed(2) +
                "% · " +
                Number(
                    dados.maiorDiaLitros
                ).toFixed(0) +
                " litros";

            document.getElementById(
                "menorDia"
            ).textContent =
                dados.menorDia;

            document.getElementById(
                "menorDiaValor"
            ).textContent =
                Number(
                    dados.menorDiaValor
                ).toFixed(2) +
                "% · " +
                Number(
                    dados.menorDiaLitros
                ).toFixed(0) +
                " litros";

            document.getElementById(
                "maiorHorario"
            ).textContent =
                formatarHorario(
                    dados.maiorHorario
                );

            document.getElementById(
                "maiorHorarioValor"
            ).textContent =
                Number(
                    dados.maiorHorarioValor
                ).toFixed(2) +
                "% na semana";

            document.getElementById(
                "menorHorario"
            ).textContent =
                formatarHorario(
                    dados.menorHorario
                );

            document.getElementById(
                "menorHorarioValor"
            ).textContent =
                Number(
                    dados.menorHorarioValor
                ).toFixed(2) +
                "% na semana";

        } catch (erro) {
            console.error(
                "Erro ao carregar histórico:",
                erro
            );
        }
    }

    atualizarDadosAtuais();
    atualizarHistorico();

    setInterval(
        atualizarDadosAtuais,
        1000
    );

    setInterval(
        atualizarHistorico,
        300000
    );

    window.addEventListener(
        "resize",
        function() {
            if (historicoRecebido) {
                const horarios = [];

                for (
                    let hora = 0;
                    hora < 24;
                    hora++
                ) {
                    horarios.push(
                        String(hora)
                            .padStart(2, "0") +
                        "h"
                    );
                }

                desenharGrafico(
                    "graficoDias",
                    historicoRecebido.dias,
                    historicoRecebido.consumoDias,
                    "linha"
                );

                desenharGrafico(
                    "graficoHorarios",
                    horarios,
                    historicoRecebido
                        .consumoHorarios,
                    "barra"
                );
            }
        }
    );
</script>

</body>
</html>
)rawliteral";

// ======================================================
// LIMITAR UM VALOR
// ======================================================

float limitarFloat(
    float valor,
    float minimo,
    float maximo
) {
    if (valor < minimo) {
        return minimo;
    }

    if (valor > maximo) {
        return maximo;
    }

    return valor;
}

// ======================================================
// CALCULAR O NÍVEL DA CAIXA
// ======================================================

float calcularNivelPercentual(float distanciaCm) {
    float nivel =
        (
            DISTANCIA_CAIXA_VAZIA_CM -
            distanciaCm
        ) /
        (
            DISTANCIA_CAIXA_VAZIA_CM -
            DISTANCIA_CAIXA_CHEIA_CM
        ) *
        100.0;

    return limitarFloat(
        nivel,
        0.0,
        100.0
    );
}

// ======================================================
// CALCULAR QUANTIDADE EM LITROS
// ======================================================

float calcularLitros(float nivelPercentual) {
    return
        CAPACIDADE_CAIXA_LITROS *
        nivelPercentual /
        100.0;
}

// ======================================================
// SINCRONIZAR DATA E HORA
// ======================================================

void configurarHorario() {
    /*
      Alta Floresta e Cuiabá utilizam UTC-4.

      O ESP32 consulta os servidores NTP
      pela internet.
    */
    configTime(
        -4 * 3600,
        0,
        "pool.ntp.org",
        "time.google.com",
        "time.cloudflare.com"
    );

    Serial.print(
        "Sincronizando data e hora"
    );

    unsigned long inicio = millis();

    time_t agora = time(nullptr);

    while (
        agora < 1700000000 &&
        millis() - inicio < 15000
    ) {
        Serial.print(".");
        delay(500);

        agora = time(nullptr);
    }

    Serial.println();

    if (agora >= 1700000000) {
        horarioSincronizado = true;

        struct tm horarioAtual;

        localtime_r(
            &agora,
            &horarioAtual
        );

        Serial.printf(
            "Horário sincronizado: "
            "%02d/%02d/%04d "
            "%02d:%02d:%02d\n",
            horarioAtual.tm_mday,
            horarioAtual.tm_mon + 1,
            horarioAtual.tm_year + 1900,
            horarioAtual.tm_hour,
            horarioAtual.tm_min,
            horarioAtual.tm_sec
        );
    } else {
        horarioSincronizado = false;

        Serial.println(
            "Não foi possível sincronizar o horário."
        );
    }
}

// ======================================================
// CARREGAR O HISTÓRICO
// ======================================================

void carregarHistorico() {
    memoria.begin(
        "caixa-agua",
        false
    );

    size_t tamanhoEsperado =
        sizeof(historico);

    size_t tamanhoSalvo =
        memoria.getBytesLength(
            "historico"
        );

    if (tamanhoSalvo == tamanhoEsperado) {
        memoria.getBytes(
            "historico",
            historico,
            tamanhoEsperado
        );

        Serial.println(
            "Histórico carregado da memória."
        );
    } else {
        memset(
            historico,
            0,
            sizeof(historico)
        );

        Serial.println(
            "Novo histórico criado."
        );
    }
}

// ======================================================
// SALVAR O HISTÓRICO
// ======================================================

void salvarHistorico() {
    memoria.putBytes(
        "historico",
        historico,
        sizeof(historico)
    );

    momentoUltimaGravacao = millis();

    Serial.println(
        "Histórico salvo na memória."
    );
}

// ======================================================
// REGISTRAR CONSUMO
// ======================================================

void registrarConsumo(float nivelAtual) {
    if (!horarioSincronizado) {
        return;
    }

    /*
      A primeira leitura apenas cria
      uma referência.
    */
    if (nivelReferenciaConsumo < 0.0) {
        nivelReferenciaConsumo =
            nivelAtual;

        momentoUltimoCalculo =
            millis();

        return;
    }

    /*
      Aguarda o intervalo de cinco minutos.
    */
    if (
        millis() - momentoUltimoCalculo <
        INTERVALO_CONSUMO_MS
    ) {
        return;
    }

    momentoUltimoCalculo = millis();

    /*
      Exemplo:

      nível anterior = 80%
      nível atual = 77%

      consumo = 80 - 77 = 3%
    */
    float consumo =
        nivelReferenciaConsumo -
        nivelAtual;

    if (
        consumo >=
            CONSUMO_MINIMO_PERCENTUAL &&
        consumo <=
            CONSUMO_MAXIMO_PERCENTUAL
    ) {
        time_t agora = time(nullptr);

        /*
          Cada valor representa uma hora
          desde 1º de janeiro de 1970.
        */
        uint32_t horaEpoch =
            (uint32_t)(agora / 3600);

        /*
          O operador % mantém o índice
          entre 0 e 167.
        */
        int indice =
            horaEpoch %
            TOTAL_HORAS_HISTORICO;

        /*
          Quando o índice pertence a uma
          hora antiga, seus dados são
          substituídos.
        */
        if (
            historico[indice].horaEpoch !=
            horaEpoch
        ) {
            historico[indice].horaEpoch =
                horaEpoch;

            historico[indice]
                .consumoPercentual = 0.0;
        }

        historico[indice]
            .consumoPercentual += consumo;

        Serial.printf(
            "Consumo registrado: %.2f%% "
            "na posição %d\n",
            consumo,
            indice
        );
    }

    /*
      Se o nível aumentar, isso representa
      abastecimento e não consumo.

      Mesmo assim, o novo nível passa a ser
      a próxima referência.
    */
    nivelReferenciaConsumo =
        nivelAtual;

    /*
      Grava o histórico aproximadamente
      uma vez por hora.
    */
    if (
        millis() - momentoUltimaGravacao >=
        3600000
    ) {
        salvarHistorico();
    }
}

// ======================================================
// NOMES DOS DIAS
// ======================================================

String nomeDiaSemana(int diaSemana) {
    const char* nomes[] = {
        "Dom",
        "Seg",
        "Ter",
        "Qua",
        "Qui",
        "Sex",
        "Sáb"
    };

    if (
        diaSemana < 0 ||
        diaSemana > 6
    ) {
        return "--";
    }

    return String(
        nomes[diaSemana]
    );
}

// ======================================================
// VERIFICAR SE DUAS DATAS SÃO IGUAIS
// ==============================================

bool mesmaData(
    const struct tm& data1,
    const struct tm& data2
) {
    return
        data1.tm_year == data2.tm_year &&
        data1.tm_mon == data2.tm_mon &&
        data1.tm_mday == data2.tm_mday;
}

// ======================================================
// ENVIAR A PÁGINA
// ===========================================

void enviarPagina() {
    servidor.send_P(
        200,
        "text/html; charset=utf-8",
        PAGINA_HTML
    );
}

// ======================================================
// ENVIAR DADOS ATUAIS EM JSON
// =======================================

void enviarDadosAtuais() {
    unsigned long tempoSemPacote;

    if (recebeuPrimeiroPacote) {
        tempoSemPacote =
            (
                millis() -
                momentoUltimoPacote
            ) /
            1000;
    } else {
        tempoSemPacote = 0;
    }

    bool comunicacaoNormal =
        recebeuPrimeiroPacote &&
        (
            millis() -
            momentoUltimoPacote
        ) <=
        TEMPO_SEM_COMUNICACAO_MS;

    String json = "{";

    json += "\"nivel\":";
    json += String(
        nivelAtualPercentual,
        1
    );
    json += ",";

    json += "\"distancia\":";
    json += String(
        distanciaAtualCm,
        1
    );
    json += ",";

    json += "\"litros\":";
    json += String(
        litrosAtuais,
        1
    );
    json += ",";

    json += "\"rssi\":";
    json += String(rssiAtual);
    json += ",";

    json += "\"snr\":";
    json += String(
        snrAtual,
        1
    );
    json += ",";

    json += "\"pacote\":";
    json += String(numeroPacote);
    json += ",";

    json += "\"tempoSemPacote\":";
    json += String(tempoSemPacote);
    json += ",";

    json += "\"comunicacao\":";
    json +=
        comunicacaoNormal ?
        "true" :
        "false";

    json += "}";

    servidor.send(
        200,
        "application/json",
        json
    );
}

// ======================================================
// ENVIAR HISTÓRICO EM JSON
// ====================================

void enviarHistorico() {
    float consumoDias[7] = {
        0, 0, 0, 0, 0, 0, 0
    };

    float consumoHorarios[24] = {0};

    String rotulosDias[7];

    struct tm datasDosDias[7];

    time_t agora = time(nullptr);

    uint32_t horaAtualEpoch =
        (uint32_t)(agora / 3600);

    /*
      Cria as datas dos últimos sete dias.

      posição 0 = seis dias atrás
      posição 6 = hoje
    */
    for (int dia = 0; dia < 7; dia++) {
        time_t data =
            agora -
            (6 - dia) * 86400;

        localtime_r(
            &data,
            &datasDosDias[dia]
        );

        rotulosDias[dia] =
            nomeDiaSemana(
                datasDosDias[dia].tm_wday
            ) +
            " " +
            String(
                datasDosDias[dia].tm_mday
            ) +
            "/" +
            String(
                datasDosDias[dia].tm_mon + 1
            );
    }

    /*
      Percorre todas as 168 posições.
    */
    for (
        int i = 0;
        i < TOTAL_HORAS_HISTORICO;
        i++
    ) {
        if (
            historico[i].horaEpoch == 0
        ) {
            continue;
        }

        if (
            historico[i].horaEpoch >
            horaAtualEpoch
        ) {
            continue;
        }

        uint32_t idadeHoras =
            horaAtualEpoch -
            historico[i].horaEpoch;

        if (
            idadeHoras >=
            TOTAL_HORAS_HISTORICO
        ) {
            continue;
        }

        time_t momentoRegistro =
            (time_t)
            historico[i].horaEpoch *
            3600;

        struct tm dataRegistro;

        localtime_r(
            &momentoRegistro,
            &dataRegistro
        );

        /*
          Soma todos os consumos ocorridos
          na mesma faixa de horário.
        */
        consumoHorarios[
            dataRegistro.tm_hour
        ] +=
            historico[i]
                .consumoPercentual;

        /*
          Descobre a qual dia o registro
          pertence.
        */
        for (int dia = 0; dia < 7; dia++) {
            if (
                mesmaData(
                    dataRegistro,
                    datasDosDias[dia]
                )
            ) {
                consumoDias[dia] +=
                    historico[i]
                        .consumoPercentual;

                break;
            }
        }
    }

    int indiceMaiorDia = 0;
    int indiceMenorDia = -1;

    for (int i = 0; i < 7; i++) {
        if (
            consumoDias[i] >
            consumoDias[indiceMaiorDia]
        ) {
            indiceMaiorDia = i;
        }

        /*
          Para o menor consumo, ignoramos
          dias ainda sem nenhum registro.
        */
        if (
            consumoDias[i] > 0 &&
            (
                indiceMenorDia < 0 ||
                consumoDias[i] <
                consumoDias[indiceMenorDia]
            )
        ) {
            indiceMenorDia = i;
        }
    }

    if (indiceMenorDia < 0) {
        indiceMenorDia = 0;
    }

    int indiceMaiorHorario = 0;
    int indiceMenorHorario = -1;

    for (int i = 0; i < 24; i++) {
        if (
            consumoHorarios[i] >
            consumoHorarios[
                indiceMaiorHorario
            ]
        ) {
            indiceMaiorHorario = i;
        }

        /*
          Horários sem nenhum consumo
          também são ignorados na escolha
          do menor consumo.
        */
        if (
            consumoHorarios[i] > 0 &&
            (
                indiceMenorHorario < 0 ||
                consumoHorarios[i] <
                consumoHorarios[
                    indiceMenorHorario
                ]
            )
        ) {
            indiceMenorHorario = i;
        }
    }

    if (indiceMenorHorario < 0) {
        indiceMenorHorario = 0;
    }

    float maiorDiaLitros =
        consumoDias[indiceMaiorDia] *
        CAPACIDADE_CAIXA_LITROS /
        100.0;

    float menorDiaLitros =
        consumoDias[indiceMenorDia] *
        CAPACIDADE_CAIXA_LITROS /
        100.0;

    String json;

    /*
      Reserva espaço para reduzir
      fragmentação da memória.
    */
    json.reserve(1800);

    json = "{";

    json += "\"dias\":[";

    for (int i = 0; i < 7; i++) {
        if (i > 0) {
            json += ",";
        }

        json += "\"";
        json += rotulosDias[i];
        json += "\"";
    }

    json += "],";

    json += "\"consumoDias\":[";

    for (int i = 0; i < 7; i++) {
        if (i > 0) {
            json += ",";
        }

        json += String(
            consumoDias[i],
            2
        );
    }

    json += "],";

    json += "\"consumoHorarios\":[";

    for (int i = 0; i < 24; i++) {
        if (i > 0) {
            json += ",";
        }

        json += String(
            consumoHorarios[i],
            2
        );
    }

    json += "],";

    json += "\"maiorDia\":\"";
    json += rotulosDias[indiceMaiorDia];
    json += "\",";

    json += "\"maiorDiaValor\":";
    json += String(
        consumoDias[indiceMaiorDia],
        2
    );
    json += ",";

    json += "\"maiorDiaLitros\":";
    json += String(
        maiorDiaLitros,
        1
    );
    json += ",";

    json += "\"menorDia\":\"";
    json += rotulosDias[indiceMenorDia];
    json += "\",";

    json += "\"menorDiaValor\":";
    json += String(
        consumoDias[indiceMenorDia],
        2
    );
    json += ",";

    json += "\"menorDiaLitros\":";
    json += String(
        menorDiaLitros,
        1
    );
    json += ",";

    json += "\"maiorHorario\":";
    json += String(
        indiceMaiorHorario
    );
    json += ",";

    json += "\"maiorHorarioValor\":";
    json += String(
        consumoHorarios[
            indiceMaiorHorario
        ],
        2
    );
    json += ",";

    json += "\"menorHorario\":";
    json += String(
        indiceMenorHorario
    );
    json += ",";

    json += "\"menorHorarioValor\":";
    json += String(
        consumoHorarios[
            indiceMenorHorario
        ],
        2
    );

    json += "}";

    servidor.send(
        200,
        "application/json",
        json
    );
}

// ======================================================
// INTERPRETAR O PACOTE LORA
// ===========================================

bool interpretarPacote(
    const String& mensagem,
    long& pacote,
    float& distancia
) {
    /*
      Formato esperado:

      NIVEL,25,83.4
    */

    int primeiraVirgula =
        mensagem.indexOf(',');

    int segundaVirgula =
        mensagem.indexOf(
            ',',
            primeiraVirgula + 1
        );

    if (
        primeiraVirgula < 0 ||
        segundaVirgula < 0
    ) {
        return false;
    }

    String tipo =
        mensagem.substring(
            0,
            primeiraVirgula
        );

    if (tipo != "NIVEL") {
        return false;
    }

    String textoPacote =
        mensagem.substring(
            primeiraVirgula + 1,
            segundaVirgula
        );

    String textoDistancia =
        mensagem.substring(
            segundaVirgula + 1
        );

    pacote =
        textoPacote.toInt();

    distancia =
        textoDistancia.toFloat();

    if (
        distancia <= 0 ||
        distancia > 1000
    ) {
        return false;
    }

    return true;
}

// ======================================================
// RECEBER PACOTE LORA
// ======================================

void verificarLoRa() {
    int tamanhoPacote =
        LoRa.parsePacket();

    if (tamanhoPacote == 0) {
        return;
    }

    String mensagem;

    while (LoRa.available()) {
        mensagem +=
            (char)LoRa.read();
    }

    mensagem.trim();

    long pacoteRecebido;
    float distanciaRecebida;

    if (
        !interpretarPacote(
            mensagem,
            pacoteRecebido,
            distanciaRecebida
        )
    ) {
        Serial.print(
            "Pacote inválido: "
        );

        Serial.println(mensagem);

        return;
    }

    numeroPacote =
        pacoteRecebido;

    distanciaAtualCm =
        distanciaRecebida;

    nivelAtualPercentual =
        calcularNivelPercentual(
            distanciaAtualCm
        );

    litrosAtuais =
        calcularLitros(
            nivelAtualPercentual
        );

    rssiAtual =
        LoRa.packetRssi();

    snrAtual =
        LoRa.packetSnr();

    momentoUltimoPacote =
        millis();

    recebeuPrimeiroPacote =
        true;

    registrarConsumo(
        nivelAtualPercentual
    );

    Serial.println(
        "--------------------------------"
    );

    Serial.print(
        "Mensagem: "
    );
    Serial.println(mensagem);

    Serial.print(
        "Pacote: "
    );
    Serial.println(numeroPacote);

    Serial.print(
        "Distância: "
    );
    Serial.print(distanciaAtualCm);
    Serial.println(" cm");

    Serial.print(
        "Nível: "
    );
    Serial.print(
        nivelAtualPercentual
    );
    Serial.println("%");

    Serial.print(
        "Quantidade: "
    );
    Serial.print(litrosAtuais);
    Serial.println(" litros");

    Serial.print(
        "RSSI: "
    );
    Serial.print(rssiAtual);
    Serial.println(" dBm");

    Serial.print(
        "SNR: "
    );
    Serial.print(snrAtual);
    Serial.println(" dB");
}

// ======================================================
// CONECTAR AO WI-FI
// =======================================

void conectarWiFi() {
    WiFi.mode(WIFI_STA);

    WiFi.begin(
        SSID_WIFI,
        SENHA_WIFI
    );

    Serial.print(
        "Conectando ao Wi-Fi"
    );

    while (
        WiFi.status() != WL_CONNECTED
    ) {
        Serial.print(".");
        delay(500);
    }

    Serial.println();

    Serial.println(
        "Wi-Fi conectado."
    );

    Serial.print(
        "Endereço do site: http://"
    );

    Serial.println(
        WiFi.localIP()
    );
}

// ======================================================
// CONFIGURAR O LORA
// ===========================================

void configurarLoRa() {
    SPI.begin(
        PINO_LORA_SCK,
        PINO_LORA_MISO,
        PINO_LORA_MOSI,
        PINO_LORA_SS
    );

    LoRa.setPins(
        PINO_LORA_SS,
        PINO_LORA_RST,
        PINO_LORA_DIO0
    );

    if (
        !LoRa.begin(
            FREQUENCIA_LORA
        )
    ) {
        Serial.println(
            "Falha ao iniciar o LoRa."
        );

        while (true) {
            delay(1000);
        }
    }

    /*
      Estes parâmetros precisam ser iguais
      no transmissor e no receptor.
    */
    LoRa.setSpreadingFactor(7);
    LoRa.setSignalBandwidth(125E3);
    LoRa.setCodingRate4(5);
    LoRa.enableCrc();

    Serial.println(
        "LoRa iniciado em 433 MHz."
    );
}

// ======================================================
// CONFIGURAR O SERVIDOR
// =============================

void configurarServidor() {
    servidor.on(
        "/",
        HTTP_GET,
        enviarPagina
    );

    servidor.on(
        "/dados",
        HTTP_GET,
        enviarDadosAtuais
    );

    servidor.on(
        "/historico",
        HTTP_GET,
        enviarHistorico
    );

    servidor.onNotFound(
        []() {
            servidor.send(
                404,
                "text/plain",
                "Página não encontrada."
            );
        }
    );

    servidor.begin();

    Serial.println(
        "Servidor web iniciado."
    );
}

// ======================================================
// SETUP
// ===========================

void setup() {
    Serial.begin(115200);

    delay(1000);

    Serial.println();
    Serial.println(
        "RECEPTOR - MONITORAMENTO DA CAIXA"
    );

    configurarLoRa();

    conectarWiFi();

    configurarHorario();

    carregarHistorico();

    configurarServidor();
}

// ======================================================
// LOOP
// =========================

void loop() {
    /*
      Verifica continuamente se chegou
      algum pacote pelo LoRa.
    */
    verificarLoRa();

    /*
      Atende as solicitações feitas
      pelo navegador.
    */
    servidor.handleClient();

    /*
      Caso o Wi-Fi caia, tenta reconectar.
    */
    if (
        WiFi.status() != WL_CONNECTED
    ) {
        WiFi.reconnect();
    }

    delay(2);
}
