# Ponderada de Computação - Semana 07

Escolhi a **Anamnai** como empresa pra abordar na ponderada, uma aplicação que desenvolvi com meu sócio da Medicina da USP. Nosso produto é um software que transcreve automaticamente consultas médicas e gera documentos estruturados para profissionais de saúde.

Como o front-end mobile consome essas APIs em tempo real durante a consulta, a arquitetura do backend precisa ser bem resiliente. Abaixo, está a forma como aplicamos os padrões de integração vistos em aula para suportar essa operação.

---

### **1) Documentar os requisitos em código**

**Requisito Funcional (RF01): Processamento e Transcrição de Áudio**
O sistema deve receber o pacote de áudio da consulta médica e integrá-lo com um serviço de terceiros (API de Transcrição) para devolver o texto estruturado ao prontuário do paciente.

Pra implementar isso de forma segura e não travar o aplicativo se a IA externa demorar a responder, o padrão **Circuit Breaker** com um *fallback* poderia ser implementado como uma fila de contingência (padrão *Message Channel*), exatamente como estudamos no caso da SEFAZ que vimos na aula.
Abaixo está a forma que esse padrão poderia ser implementado:

```python
# app/services/transcricao_service.py
from circuitbreaker import circuit_breaker
import fila_contingencia

def transcrever_consulta(dados_audio):
    """
    RF: Envia o áudio para a API de transcrição.
    RNF embutido: Em caso de falha ou timeout, o circuito abre e envia para contingência.
    """
    # Circuit Breaker com limite de 3 falhas e 30s de reset
    @circuit_breaker(failure_threshold=3, reset_timeout=30)
    def call_ai_api():
        try:
            # Integração com o serviço de IA externo
            response = ai_client.post("/transcribe", json=dados_audio, timeout=5)
            return response.json()
        except Timeout:
            # Em caso de timeout, a operação vai para a fila
            fila_contingencia.enqueue(dados_audio)
            return {"status": "contingencia", "msg": "Áudio na fila para processamento assíncrono"}

    return call_ai_api()
```

---

### **2) Aferir a qualidade dos requisitos envolvidos no sistema**

**Requisito Não Funcional (RNF01): Resiliência e Tolerância a Falhas**
O sistema não pode perder os dados da consulta médica caso a API de IA caia ou sofra degradação de performance. A qualidade desse RNF é aferida através de testes BDD (Behavior-Driven Development), simulando a falha do serviço externo e garantindo que o roteamento de mensagens assuma o controle.

Abaixo está o código que poderiamos ter usado pra testar e validar o comportamento do sistema quando um problema ocorre:

**Arquivo Gherkin (`resiliencia_ia.feature`):**
```gherkin
@rnf @resiliencia @falha
Feature: Resiliência na transcrição automática de consultas

  Scenario: Circuit Breaker abre com API de IA em manutenção
    Given a API de IA externa está em manutenção ou lenta
    And o circuit breaker está CLOSED
    When 3 tentativas de transcrição falham por timeout
    Then o CB muda para o estado OPEN
    And o áudio da consulta é enviado com sucesso para a fila de contingência
```

**Step Definitions em Python (`steps.py`):**
```python
# app/tests/steps.py
from behave import given, when, then
from unittest.mock import MagicMock, patch

@given('a API de IA externa está em manutenção ou lenta')
def step_ia_manutencao(context):
    # Poderiamos mockar o client da IA para forçar um erro de timeout e testar a resiliência
    context.ai_client = MagicMock()
    context.ai_client.post.side_effect = TimeoutError("API de IA indisponível - Timeout")

@then('o áudio da consulta é enviado com sucesso para a fila de contingência')
def step_fila_contingencia(context):
    # Seria importante aferir a qualidade do RNF garantindo que o método de fila foi chamado corretamente
    context.fila_contingencia.enqueue.assert_called_once()
    print("Sucesso: Nenhuma consulta foi perdida, dados salvos na fila!")
```

**Explicação da Aferição:** Ao rodar estes códigos de teste, temos uma garantia de que a arquitetura cumpre o requisito de **Disponibilidade**. Se o provedor de IA sair do ar, o *Circuit Breaker* isola a falha, evitando o esgotamento dos recursos do nosso backend, e o *Message Channel* (Fila) garante que o documento estruturado do médico será gerado assim que o sistema normalizar.