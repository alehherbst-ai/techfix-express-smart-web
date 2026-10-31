# Cenário Make — Triagem Inteligente (Gemini)

Fluxo: **Custom webhook → HTTP (Gemini) → Webhook response**

## 0. Chave do Gemini
1. Acesse https://aistudio.google.com/apikey e crie uma chave (cota gratuita).
2. Guarde só no Make (campo da URL/headers do módulo HTTP). Nunca no JS do site.

## 1. Módulo 1 — Webhooks > Custom webhook
- Crie o webhook (ex.: `triagem-techfix`) e copie a URL.
- Clique em "Run once" e envie um teste (veja o curl no fim) para o Make aprender a estrutura: `nome, aparelho, modelo, descricao`.

## 2. Módulo 2 — HTTP > Make a request
- URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent` (modelo rápido e disponível na conta; o gemini-2.5-flash retornou 404 e o gemini-3.8-flash estourou o tempo por excesso de demanda)
- Method: `POST`
- Headers:
  - `Content-Type: application/json`
  - `x-goog-api-key: SUA_CHAVE`
- Body type: `Raw` / Content type: `JSON (application/json)`
- Request content:

```json
{
  "systemInstruction": {
    "parts": [{
      "text": "Você é o assistente de triagem da TechFix Express, assistência técnica de celulares e informática no Brasil. Com base no aparelho e no defeito descrito, devolva um pré-diagnóstico em português do Brasil. Não invente certezas: é uma estimativa, o diagnóstico final é feito na bancada. Valores em reais (número inteiro). Categorias possíveis: Tela, Bateria, Placa/Não liga, Água/Oxidação, Software/Sistema, Vírus/Malware, Armazenamento/Lentidão, Conectividade, Outro. urgencia deve ser exatamente: alta, media ou baixa. proximos_passos: 3 itens curtos e acionáveis."
    }]
  },
  "contents": [{
    "role": "user",
    "parts": [{
      "text": "Aparelho: {{1.aparelho}}\nModelo: {{1.modelo}}\nCliente: {{1.nome}}\nDescrição do defeito: {{1.descricao}}"
    }]
  }],
  "generationConfig": {
    "temperature": 0.3,
    "responseMimeType": "application/json",
    "responseSchema": {
      "type": "OBJECT",
      "properties": {
        "categoria": {"type": "STRING"},
        "urgencia": {"type": "STRING", "enum": ["alta", "media", "baixa"]},
        "estimativa_min": {"type": "INTEGER"},
        "estimativa_max": {"type": "INTEGER"},
        "prazo": {"type": "STRING"},
        "resumo": {"type": "STRING"},
        "proximos_passos": {"type": "ARRAY", "items": {"type": "STRING"}}
      },
      "required": ["categoria", "urgencia", "estimativa_min", "estimativa_max", "prazo", "resumo", "proximos_passos"]
    }
  }
}
```

- Marque **Parse response** = Yes.
- Atenção: se `descricao` tiver aspas ou quebras de linha, o JSON quebra. Se acontecer, aplique `replace()`/`encodeURL` ou monte o body com o módulo **Tools > Set variable** + função `toString`. Para o trabalho, os testes simples funcionam.

## 3. Módulo 3 — Webhooks > Webhook response
- Status: `200`
- Body: `{{2.data.candidates[].content.parts[].text}}`
  (o Gemini devolve o JSON como texto dentro dessa propriedade, que já é o JSON final)
- Custom headers:
  - `Content-Type: application/json`
  - `Access-Control-Allow-Origin: *`

## 4. Ligar no site
- Ative o cenário (ON, "Immediately as data arrives").
- Cole a URL do webhook em `WEBHOOK_URL` no `triagem-embed.html`.
- Troque `WHATSAPP` pelo seu número (formato `55DDDNUMERO`).

## 5. Teste rápido pelo terminal
```bash
curl -X POST "URL_DO_WEBHOOK" \
  -d "nome=Maria" -d "aparelho=Celular" -d "modelo=iPhone 11" \
  -d "descricao=Tela quebrou depois de uma queda e o toque falha"
```
Resposta esperada: JSON com `categoria`, `urgencia`, `estimativa_min/max`, `prazo`, `resumo`, `proximos_passos`.

## Evidências para o trabalho (prints)
- Cenário no Make com os 3 módulos.
- Prompt do módulo HTTP.
- Execução bem-sucedida (histórico do cenário).
- A triagem funcionando no site.
