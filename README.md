# API da Biblioteca

API de empréstimos de livros em FastAPI, usada nas aulas de Engenharia de Software.

## Rodar localmente

```bash
pip install -r requirements-dev.txt
pytest -v                          # testes
ruff check .                       # lint
fastapi dev biblioteca/main.py     # sobe a API: http://127.0.0.1:8000/docs
```

## Rodar com Docker

```bash
docker build -t biblioteca .
docker run -p 8000:8000 biblioteca
```

## Estrutura

| Pasta | Conteúdo |
|---|---|
| `biblioteca/` | Código da API |
| `tests/` | Testes unitários (`test_regras.py`) e de integração (`test_api.py`) |
| `scripts/gerar_docs.py` | Gera a página de documentação publicada pelo pipeline |
| `scripts/conferir_pipeline.py` | Confere as missões do laboratório no seu workflow |

O laboratório de CI/CD está em [ROTEIRO.md](ROTEIRO.md).
