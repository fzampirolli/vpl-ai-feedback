*[Read in English](README.en.md)*

# VPL AI Feedback

Sistema que gera **feedback qualitativo e socrático** (via LLM), complementar à correção automática por casos de teste do **VPL (Virtual Programming Lab)** no Moodle. Para cada aluno, o código submetido é enviado a um provedor de IA junto com uma rubrica definida pelo professor; a IA identifica o tipo de questão sorteada, aplica a rubrica correspondente e produz uma avaliação detalhada — sem nunca substituir a nota objetiva do VPL ou o julgamento do professor.

O projeto cobre tanto atividades VPL de **uma única questão** (ex.: exercícios formativos, quizzes) quanto de **múltiplas questões paramétricas por aluno** (ex.: provas com 3 questões sorteadas entre tipos A/B/C).

Quando a atividade VPL tem **apenas um código submetido**, a questão pode ser **estática** (mesmo enunciado para todos os alunos) — nesse caso não há sorteio de variantes e a ferramenta MCTest não é necessária; o enunciado oficial extraído do VPL simplesmente não varia de aluno para aluno. Já quando a atividade envolve questões **paramétricas** (uma variante distinta por aluno, como nas provas), as questões são geradas pela ferramenta MCTest e entregues tanto em PDF impresso quanto como arquivo anexo à própria atividade VPL — é desse anexo/enunciado, extraído do `execution.txt` do VPL, que `core/utils.py` obtém o **enunciado oficial** usado como evidência primária para a IA identificar corretamente o tipo sorteado de cada questão.

> Referência: Zampirolli et al. (2026), *"Intelligent Feedback for Individualized Introductory Programming Exercises"* (ver [Referências](#referências)).

## Como funciona

1. As submissões dos alunos são exportadas do Moodle/VPL como uma pasta por aluno (`<Nome> - <login>/<timestamp>/` com o código, e `<timestamp>.ceg/execution.txt` com o resultado real da execução dos casos de teste).
2. `main.py` percorre cada pasta de aluno, monta um prompt por questão (código do aluno + enunciado oficial da questão sorteada + evidência real de execução do VPL + os demais códigos do aluno como contexto de estilo) e envia para o provedor de LLM configurado.
3. A IA responde com o tipo de questão identificado, a confiança nessa identificação e a avaliação segundo a rubrica correspondente, incluindo uma nota estruturada (`NOTA FINAL: X/Y`).
4. Os resultados são gravados por aluno (`rubrica.txt`, com cache — reexecuções não pagam API de novo) e consolidados em um CSV comparando nota do Moodle vs. nota da IA, por questão.
5. `enviar_email.py` envia a cada aluno seu relatório individual por e-mail, sinalizando quando alguma questão teve baixa confiança na identificação de tipo (recomendando revisão manual do professor).

Em nenhum momento a nota da IA substitui a nota do VPL: ela é informativa/formativa, e a rubrica/prompt usados são sempre definidos e aprovados pelo professor (mesmo que rascunhados com auxílio de IA).

## Estrutura do projeto

```
main.py                  Orquestra a correção em lote (assíncrona, com concorrência limitada)
core/
  grader.py              Monta os prompts por questão, chama a LLM, gera rubrica.txt/CSV/ALL.txt
  utils.py               Extração de notas da IA, leitura de submissões e do execution.txt do VPL
providers/
  __init__.py            Factory: escolhe o cliente de LLM conforme config.yaml (llm.provider)
  base.py                Lógica comum: fallback entre modelos, retries, backoff em erro 429
  groq.py                Cliente para a API da Groq (OpenAI-compatible)
  deepseek.py            Cliente para a API da DeepSeek (OpenAI-compatible)
  gemini.py              Cliente para a API do Google Gemini
enviar_email.py           Envia por e-mail o rubrica.txt de cada aluno, via SMTP (com fallback de TLS)
atualizar_conceitos.py    Atualiza conceitos (A–F) numa planilha .xls a partir do CSV de notas do Moodle
busca.sh                  Lista submissões que ainda não têm rubrica.txt gerado
renomear_pastas.sh        Renomeia pastas exportadas do Moodle para o padrão "Nome - login"
run.sh                    Wrapper de main.py com log opcional e estatísticas de tempo
config.yaml.example       Modelo de configuração (copiar para config.yaml, nunca versionar)
promptP1.txt / promptP2.txt / promptP3.txt
                           Prompts para atividades de UMA questão por aluno (quizzes)
promptProva1.txt / promptProva2.txt
                           Prompts para provas de MÚLTIPLAS questões por aluno (tipos A/B/C)
p1moodle0/                 Exemplo de dado exportado (mantido no repositório como referência)
old/                       Versões anteriores dos scripts (mantidas para histórico)
```

## Instalação

```bash
git clone https://github.com/fzampirolli/vpl-ai-feedback.git
cd vpl-ai-feedback
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

`atualizar_conceitos.py` tem dependências próprias (`xlrd==1.2.0`, `xlwt`, `xlutils`, `pandas`) que ele mesmo instala sob demanda na primeira execução.

## Configuração

```bash
cp config.yaml.example config.yaml
```

Edite `config.yaml` (nunca commitado — já está no `.gitignore`) com:

- **`llm.provider`**: `groq`, `deepseek` ou `gemini`.
- **`<provider>.api_key`** e **`<provider>.models`**: chave da API e lista de modelos (o primeiro é tentado primeiro; em falha, tenta o próximo — útil para contornar rate limit/indisponibilidade).
- **`grading.prompt_file`**: qual arquivo de prompt/rubrica usar (ex.: `promptP3.txt` para uma questão, `promptProva2.txt` para prova com 3 questões).
- **`grading.weights`**: peso de cada questão, ex. `q1: 50` para uma atividade de questão única, ou `q1: 33` / `q2: 33` / `q3: 34` para uma prova de 3 questões.
- **`paths.student_base_dir`**: pasta com as submissões exportadas do Moodle.
- **`email.*`** e **`templates.*`**: servidor SMTP e o texto do e-mail enviado a cada aluno.

## Uso

**Rodar a correção por IA** (gera `rubrica.txt` por aluno + CSV + arquivo consolidado):

```bash
bash run.sh config.yaml --log     # --log grava a saída em log_correcao_<data>.txt
# ou diretamente:
python3 main.py --config config.yaml --concurrent 3
```

Saídas geradas ao lado da pasta de alunos:
- `<student_base_dir>/<aluno>/<submissão>/rubrica.txt` — relatório individual (também serve de cache: se já existe, não reconsulta a API).
- `<student_base_dir>_relatorio.csv` — comparação Moodle vs. IA, por questão, com colunas de tipo identificado, confiança e uma coluna `Revisar_Manualmente`.
- `<student_base_dir>_ALL.txt` — todos os relatórios individuais concatenados.

**Verificar pendências** (submissões sem `rubrica.txt` ainda):

```bash
./busca.sh p1moodle
```

**Enviar os relatórios por e-mail:**

```bash
python3 enviar_email.py
```

**Renomear pastas exportadas do Moodle** para o padrão `Nome - login` esperado pelo restante do pipeline:

```bash
./renomear_pastas.sh /caminho/para/pasta_exportada
```

**Atualizar conceitos (A–F) numa planilha de notas** a partir do CSV de submissões do Moodle:

```bash
python3 atualizar_conceitos.py notas_turma.xls Prova1_submissions.csv -o notas_atualizado.xls
```

## Atividades de uma questão vs. múltiplas questões

O mesmo pipeline (`main.py` → `core/grader.py`) atende os dois formatos; o que muda é o arquivo de prompt e o dicionário `weights`:

- **Uma questão** (`promptP1.txt`, `promptP2.txt`, `promptP3.txt`): `grading.weights` tem uma única chave (ex.: `q1: 50`). O prompt descreve os tipos possíveis dessa questão e a rubrica de cada um.
- **Múltiplas questões / prova** (`promptProva1.txt`, `promptProva2.txt`): `grading.weights` tem uma chave por questão sorteada (ex.: `q1`, `q2`, `q3`). Cada questão é avaliada isoladamente pela IA (uma chamada de API por questão, todas do mesmo aluno), mas o `rubrica.txt` final e o CSV consolidam as três notas e a nota total do aluno.

Em ambos os casos, o prompt exige que a IA declare explicitamente o **tipo identificado** e a **confiança** dessa identificação antes de aplicar a rubrica — usado depois para marcar automaticamente, no CSV, quais alunos merecem revisão manual do professor.

## Provedores de LLM

`providers/__init__.py` implementa uma factory simples: `get_client(config)` lê `llm.provider` e instancia o cliente correspondente. Todos herdam de `providers/base.py`, que cuida de:

- fallback entre múltiplos modelos configurados;
- até 3 tentativas por modelo, com backoff exponencial (respeitando `retry-after` em erros 429);
- interrupção imediata em erros de autenticação/saldo (401/402).

Para adicionar um novo provedor, basta criar uma subclasse de `BaseLLMClient` implementando `_raw_call()` e registrá-la no `mapping` de `providers/__init__.py`.

## Segurança

- `config.yaml` (com chaves de API e senha SMTP) está no `.gitignore` — nunca deve ser commitado. Use `config.yaml.example` como modelo.
- Dados de alunos (pastas de submissão, CSVs de notas, relatórios) também são ignorados por padrão, exceto o exemplo `p1moodle0/`.

## Referências

- ZAMPIROLLI, Francisco de Assis; TEUBL, Fernando; PISANI, Paulo Henrique; SILVA, Thiago Alexandre Paiares e. **Intelligent Feedback for Individualized Introductory Programming Exercises**. *Computer Applications in Engineering Education*, v. 34, n. 1, p. e70132, 2026. DOI: [10.1002/cae.70132](https://doi.org/10.1002/cae.70132).
- ZAMPIROLLI, Francisco de Assis. **MCTest: como criar e corrigir exames parametrizados automaticamente**. 1ª ed. 2023. ISBN: 978-65-00-79086-3. Disponível em: <https://github.com/fzampirolli/mctest/tree/master/book>.
