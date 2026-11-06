# Laboratório de CI/CD: o pipeline da API da Biblioteca

Hoje a API da biblioteca ganha um pipeline de verdade no GitHub Actions. A cada push, ele vai rodar o lint, rodar os
testes em três versões do Python, construir a imagem Docker e testar se ela sobe. Quando tudo estiver verde na
`main`, ele publica a imagem e a documentação da API na internet, sem ninguém apertar botão.

São 6 missões. Façam na ordem: cada uma usa a anterior.

| Missão | O que entrega | Tempo |
|---|---|---|
| 0 | Repositório no GitHub com o código | 10 min |
| 1 | Primeiro pipeline: testes a cada push e PR | 10 min |
| 2 | Quebrar de propósito e ler o log | 5 min |
| 3 | Lint, matriz de versões e cobertura mínima | 15 min |
| 4 | `main` protegida: só entra com pipeline verde | 5 min |
| 5 | Imagem Docker + teste de fumaça | 10 min |
| 6 | Entrega contínua: imagem no GHCR e site no GitHub Pages | 15 min |
| ★ | Desafio: release por tag | se sobrar tempo |

A qualquer momento, rodem o conferidor para ver o que o YAML de vocês já cobre:

```bash
python scripts/conferir_pipeline.py
```

Ele só lê o arquivo; quem diz se o pipeline está verde é a aba **Actions** do GitHub.

Se travarem, cada missão tem uma dica escondida no final. Tentem antes de abrir.

---

## Missão 0 · Repositório no GitHub

1. Instalem e rodem tudo localmente uma vez:
   ```bash
   python -m venv .venv
   source .venv/bin/activate        # Windows: .venv\Scripts\activate
   pip install -r requirements-dev.txt
   ruff check .
   pytest -v
   ```
   Tem que dar **42 testes passando** e o ruff sem erros.
2. Criem um repositório **público** no GitHub chamado `biblioteca-ci` (repositórios públicos têm Actions,
   GitHub Pages e o registro de imagens de graça).
3. Enviem o código:
   ```bash
   git init -b main
   git add .
   git commit -m "API da biblioteca"
   git remote add origin https://github.com/SEU_USUARIO/biblioteca-ci.git
   git push -u origin main
   ```
   Com o GitHub CLI dá para fazer os passos 2 e 3 de uma vez:
   `gh repo create biblioteca-ci --public --source . --push`.

✅ **Checkpoint:** o código aparece na página do repositório.

---

## Missão 1 · Primeiro pipeline

Criem o arquivo `.github/workflows/ci.yml`. O GitHub executa automaticamente todo YAML que estiver nessa pasta.

```yaml
name: CI

on:                        # QUANDO roda
  push:
    branches: [main]
  pull_request:

jobs:                      # O QUE roda
  testes:
    runs-on: ubuntu-latest # ONDE roda: uma máquina virtual nova a cada execução
    steps:
      - uses: actions/checkout@v7          # baixa o código do repositório
      - uses: actions/setup-python@v7      # instala o Python
        with:
          python-version: "3.13"
      - run: pip install -r requirements-dev.txt
      - run: pytest -v
```

Façam commit e push. Abram a aba **Actions** do repositório e acompanhem a execução ao vivo.

Perguntas para o grupo responder olhando a tela:
- Quanto tempo levou? Qual passo foi o mais demorado?
- A máquina que rodou os testes é a de vocês? O que acontece com ela quando o job termina?

✅ **Checkpoint:** um ✔ verde ao lado do último commit.

---

## Missão 2 · Quebrar de propósito

Um pipeline que nunca fica vermelho pode não estar testando nada. Vamos ver ele falhar.

1. Criem uma branch: `git switch -c quebra`
2. Em `biblioteca/regras.py`, troquem `MULTA_POR_DIA = 2.00` por `3.00`.
3. Commit, `git push -u origin quebra`, e abram um **pull request** para a `main`.
4. Esperem o ✗ vermelho. Cliquem em **Details** e encontrem, no log, **qual teste falhou e por quê**.
5. Fechem o PR sem fazer merge e voltem para a main: `git switch main`.

✅ **Checkpoint:** o nome do teste que falhou e o valor esperado × obtido.

---

## Missão 3 · Qualidade

Agora o pipeline vai cobrar mais do que "os testes passam". Editem o `ci.yml` para:

1. Ter um **job `lint`** separado, que roda `ruff check .` e `ruff format --check .`.
2. Rodar os testes numa **matriz** com Python **3.11, 3.12 e 3.13**. A biblioteca promete funcionar a partir do
   3.11 (`requires-python` no `pyproject.toml`), então o pipeline tem que provar isso.
3. **Falhar se a cobertura cair abaixo de 90%**: `pytest --cov --cov-fail-under=90`.

Dica de sintaxe para a matriz:
```yaml
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
```
e, no `setup-python`, `python-version: ${{ matrix.python-version }}`.

Na aba Actions, vocês devem ver **4 jobs** rodando em paralelo: lint e três de testes.

✅ **Checkpoint:** 4 jobs verdes, e o conferidor marcando a Missão 3.

---

## Missão 4 · Proteger a main

De que adianta o pipeline se alguém pode dar push direto na `main` com tudo vermelho?

1. No repositório: **Settings › Rules › Rulesets › New ruleset › New branch ruleset**.
2. Nome: `proteger-main`. Enforcement status: **Active**.
3. Em *Target branches*, **Add target › Include default branch**.
4. Marquem:
   - **Require a pull request before merging** (podem deixar 0 aprovações obrigatórias para o laboratório);
   - **Require status checks to pass**, e adicionem os checks `lint` e os três `testes (...)`.
     Eles só aparecem na busca depois de terem rodado pelo menos uma vez (Missão 3).
5. Salvem e testem: tentem dar `git push` direto na `main`. Deve ser **recusado**.

A partir daqui, todo trabalho vai em branch + pull request.

✅ **Checkpoint:** o push direto na main é recusado com uma mensagem sobre a regra.

---

## Missão 5 · Imagem Docker e teste de fumaça

O repositório já tem um `Dockerfile`. Adicionem ao workflow um **job `imagem`** que:

1. Só roda **depois** que `lint` e `testes` passarem (`needs: [lint, testes]`).
2. Constrói a imagem: `docker build -t biblioteca:ci .`
3. Faz um **teste de fumaça**: sobe o container e confere se `GET /saude` responde.
   Se não responder em 30 segundos, mostra os logs do container e falha.

Por que testar de novo se os testes já passaram? Porque os testes rodam no ambiente de desenvolvimento, com
`requirements-dev.txt`. O container usa só `requirements.txt`. São ambientes diferentes, e a sabotagem do fim da
aula vai mostrar por que isso importa.

Façam isso numa branch e abram um PR: com a main protegida, é o único caminho.

✅ **Checkpoint:** no PR, o job `imagem` aparece depois dos outros e fica verde.

---

## Missão 6 · Entrega contínua

Até aqui o pipeline **verifica**. Agora ele vai **entregar**. Quando um PR entra na `main`, dois jobs novos rodam:

**6a. Publicar a imagem no GitHub Container Registry (ghcr.io)**

```yaml
  publicar-imagem:
    needs: imagem
    if: github.event_name == 'push'      # nunca em pull request
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write                    # permissão para escrever no registro
    steps:
      - uses: actions/checkout@v7
      - name: Nome da imagem (o registro exige minúsculas)
        run: echo "IMAGEM=ghcr.io/${GITHUB_REPOSITORY,,}" >> "$GITHUB_ENV"
      - run: docker build -t "$IMAGEM:latest" -t "$IMAGEM:${GITHUB_SHA::7}" .
      - name: Login no registro
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin
      - run: docker push --all-tags "$IMAGEM"
```

`secrets.GITHUB_TOKEN` é criado automaticamente pelo GitHub para cada execução. Ninguém precisa digitar senha, e
ela nunca aparece no código.

**6b. Publicar a documentação da API no GitHub Pages**

Primeiro: **Settings › Pages › Build and deployment › Source: GitHub Actions**.

Depois escrevam o job `publicar-docs`. Ele precisa:
- rodar só em push na `main`;
- ter as permissões `pages: write` e `id-token: write`;
- gerar o site: `pytest -q --cov --cov-report=html:site/cobertura` e `python scripts/gerar_docs.py site`;
- enviar a pasta `site` com `actions/upload-pages-artifact@v5` e publicar com `actions/deploy-pages@v5`.

Façam merge do PR e acompanhem a execução na `main`.

✅ **Checkpoint:**
1. `https://SEU_USUARIO.github.io/biblioteca-ci/` abre a documentação da API, com o commit atual no topo.
2. A imagem aparece em **Packages**, no perfil de vocês. Quem tiver Docker pode rodar:
   `docker run -p 8000:8000 ghcr.io/SEU_USUARIO/biblioteca-ci:latest`

---

## ★ Desafio · Releases por tag

Façam o pipeline criar uma **release** quando alguém enviar uma tag de versão:

```bash
git tag v1.0.0
git push origin v1.0.0
```

Pistas:
- o gatilho `push` aceita `tags: ["v*"]`;
- o job só deve rodar se `startsWith(github.ref, 'refs/tags/v')`;
- o comando `gh release create "$TAG" --generate-notes` já vem instalado nas máquinas do Actions; ele precisa de
  `GH_TOKEN` e da permissão `contents: write`.

Bônus: usem `cache: pip` no `setup-python` e comparem o tempo de instalação antes e depois.

---

## Dicas escondidas

<details>
<summary>Missão 3: o workflow inteiro</summary>

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.13"
      - run: pip install ruff
      - run: ruff check .
      - run: ruff format --check .

  testes:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements-dev.txt
      - run: pytest -v --cov --cov-report=term-missing --cov-fail-under=90
```
</details>

<details>
<summary>Missão 5: o job imagem</summary>

```yaml
  imagem:
    needs: [lint, testes]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: docker build -t biblioteca:ci .
      - name: Sobe o container e chama /saude
        run: |
          docker run -d --name api -p 8000:8000 biblioteca:ci
          for _ in $(seq 1 30); do
            if curl -fsS http://localhost:8000/saude; then exit 0; fi
            sleep 1
          done
          docker logs api
          exit 1
```
</details>

<details>
<summary>Missão 6b: o job publicar-docs</summary>

```yaml
  publicar-docs:
    needs: [lint, testes]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deploy.outputs.page_url }}
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.13"
      - run: pip install -r requirements-dev.txt
      - name: Gera o site (docs da API + relatório de cobertura)
        run: |
          pytest -q --cov --cov-report=html:site/cobertura
          python scripts/gerar_docs.py site
      - uses: actions/upload-pages-artifact@v5
        with:
          path: site
      - id: deploy
        uses: actions/deploy-pages@v5
```
</details>

## Quando algo der errado

| Sintoma | Causa provável |
|---|---|
| O workflow nem aparece na aba Actions | O arquivo não está em `.github/workflows/` ou o YAML tem erro de indentação. |
| `Error: Process completed with exit code 1` | Leiam as linhas **acima** dessa mensagem: o erro de verdade está lá. |
| O check não aparece na busca do ruleset | Ele precisa ter rodado ao menos uma vez. |
| `denied: permission_denied` no push da imagem | Faltou `packages: write` no job. |
| Deploy do Pages falha com erro de permissão ou de ambiente | Faltou Settings › Pages › Source: **GitHub Actions**, ou `pages: write` / `id-token: write`. |
| `invalid reference format` no docker | O nome da imagem tem letra maiúscula: usem `${GITHUB_REPOSITORY,,}`. |
