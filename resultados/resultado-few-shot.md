# Experimento 2 — Resultado Few-shot

## Casos de teste de integração

Os cenários mantêm a estrutura dos exemplos do prompt. Os contratos HTTP do back-end e a normalização dos identificadores OpenAlex devem ser ajustados para a aplicação real. A referência pode conter mais de uma instituição; uma publicação conta uma única vez quando ao menos uma instituição de autoria não pertence ao conjunto de referência.

### Caso FT-01 — Pesquisador sem publicações

- **Cenário:** pesquisador existente sem publicações cadastradas na OpenAlex.
- **Entrada:** identificador do pesquisador `A100`; resposta de trabalhos vazia (`results: []`).
- **Execução:** o teste envia uma requisição HTTP à rota de análise do back-end; a chamada de saída à OpenAlex é simulada para retornar uma coleção vazia.
- **Resultado esperado:** HTTP 200, zero publicações em colaboração e lista de publicações colaborativas vazia; a interface/cliente apresenta estado vazio, não erro.

### Caso FT-02 — Publicação sem afiliação institucional

- **Cenário:** pesquisador possui uma publicação cuja resposta não informa instituições nas autorias.
- **Entrada:** trabalho `W1` com autoria sem `institutions` (ou com lista vazia), referência `I10`.
- **Execução:** o teste consulta a análise pelo back-end e simula a resposta da OpenAlex com o trabalho incompleto.
- **Resultado esperado:** HTTP 200, `W1` não é contado como colaboração sem evidência de instituição externa e é identificado como dado incompleto para diagnóstico.

### Caso FT-03 — Pesquisador associado a múltiplas instituições

- **Cenário:** pesquisador tem vínculos com mais de uma instituição de referência.
- **Entrada:** instituições de referência `I10` e `I11`; `W1` possui apenas `I10` e `I11`; `W2` possui `I11` e instituição externa `I20`.
- **Execução:** solicitar a análise informando as instituições de referência; simular os dois trabalhos com suas afiliações.
- **Resultado esperado:** somente `W2` é colaboração externa; total igual a 1 e publicação listada uma única vez.

### Caso FT-04 — Falha de comunicação ou timeout na OpenAlex

- **Cenário:** API externa indisponível ou sem resposta dentro do limite configurado.
- **Entrada:** pesquisador `A100`; mock da OpenAlex lança `ConnectionError` ou `Timeout` (executar como duas variantes).
- **Execução:** enviar uma requisição HTTP à rota de análise e simular a falha no cliente HTTP usado pelo back-end.
- **Resultado esperado:** resposta de erro controlada conforme contrato (por exemplo, 502 para indisponibilidade e 504 para timeout); não retornar sucesso com total zero ou indicador parcial.

### Caso FT-05 — Paginação de publicações

- **Cenário:** publicações do pesquisador estão distribuídas em mais de uma página.
- **Entrada:** página 1 contém `W1` colaborativa e cursor para página 2; página 2 contém `W2` local e `W3` colaborativa, sem cursor seguinte.
- **Execução:** solicitar a análise; simular as páginas da OpenAlex e verificar os parâmetros de cursor enviados pelo back-end.
- **Resultado esperado:** back-end percorre todas as páginas, considera `W1`, `W2` e `W3`, retorna duas publicações colaborativas (`W1`, `W3`) e não duplica registros.

## Código de referência automatizado

O exemplo abaixo é autocontido: disponibiliza uma rota Flask mínima para ilustrar o contrato e executa requisições HTTP reais ao back-end local usando `requests`. As chamadas de saída à OpenAlex são simuladas, portanto os testes não dependem da rede externa. Em uma aplicação existente, substitua `app` e a rota de exemplo pelo back-end real, preservando a estratégia de mock no limite HTTP externo.

Instalação das dependências:

```bash
python -m pip install Flask requests pytest
```

Salve como `test_collaborations.py` e execute `pytest -q test_collaborations.py`.

```python
from threading import Thread

import pytest
import requests
from flask import Flask, jsonify, request
from werkzeug.serving import make_server


OPENALEX_WORKS_URL = "https://api.openalex.org/works"
app = Flask(__name__)


def _institution_ids(work):
    ids = set()
    for authorship in work.get("authorships") or []:
        for institution in authorship.get("institutions") or []:
            institution_id = institution.get("id")
            if institution_id:
                ids.add(institution_id)
    return ids


@app.post("/api/researchers/<author_id>/collaborations")
def collaborations(author_id):
    reference_ids = set(request.get_json()["reference_institution_ids"])
    cursor = "*"
    seen_work_ids = set()
    collaborative_ids = []
    incomplete_ids = []

    try:
        while cursor:
            response = requests.get(
                OPENALEX_WORKS_URL,
                params={
                    "filter": f"author.id:{author_id}",
                    "per-page": 100,
                    "cursor": cursor,
                },
                timeout=5,
            )
            response.raise_for_status()
            payload = response.json()

            for work in payload.get("results", []):
                work_id = work.get("id")
                if not work_id or work_id in seen_work_ids:
                    continue
                seen_work_ids.add(work_id)

                institutions = _institution_ids(work)
                if not institutions:
                    incomplete_ids.append(work_id)
                elif institutions - reference_ids:
                    collaborative_ids.append(work_id)

            cursor = (payload.get("meta") or {}).get("next_cursor")
    except requests.Timeout:
        return jsonify({"error": "OpenAlex timeout"}), 504
    except requests.RequestException:
        return jsonify({"error": "OpenAlex unavailable"}), 502

    return jsonify(
        {
            "author_id": author_id,
            "collaboration_count": len(collaborative_ids),
            "collaborative_work_ids": collaborative_ids,
            "incomplete_work_ids": incomplete_ids,
        }
    )


@pytest.fixture
def backend_url():
    server = make_server("127.0.0.1", 0, app, threaded=True)
    thread = Thread(target=server.serve_forever, daemon=True)
    thread.start()
    try:
        yield f"http://127.0.0.1:{server.server_port}"
    finally:
        server.shutdown()
        thread.join(timeout=2)


def work(work_id, *institution_ids):
    return {
        "id": work_id,
        "authorships": [
            {"institutions": [{"id": institution_id} for institution_id in institution_ids]}
        ],
    }


class FakeResponse:
    def __init__(self, payload):
        self.payload = payload

    def raise_for_status(self):
        return None

    def json(self):
        return self.payload


def mock_openalex(monkeypatch, responses):
    calls = []

    def fake_get(url, *, params, timeout):
        assert url == OPENALEX_WORKS_URL
        assert timeout == 5
        calls.append(params.copy())
        result = responses.pop(0)
        if isinstance(result, BaseException):
            raise result
        return FakeResponse(result)

    monkeypatch.setattr(
        requests,
        "get",
        fake_get,
    )
    return calls


def test_author_without_works_returns_empty_result(backend_url, monkeypatch):
    calls = mock_openalex(
        monkeypatch, [{"results": [], "meta": {"next_cursor": None}}]
    )
    response = requests.post(
        f"{backend_url}/api/researchers/A100/collaborations",
        json={"reference_institution_ids": ["I10"]},
        timeout=2,
    )

    assert response.status_code == 200
    assert response.json()["collaboration_count"] == 0
    assert response.json()["collaborative_work_ids"] == []
    assert calls[0]["filter"] == "author.id:A100"


def test_missing_affiliations_are_not_counted(backend_url, monkeypatch):
    calls = mock_openalex(
        monkeypatch,
        [{"results": [{"id": "W1"}], "meta": {"next_cursor": None}}],
    )

    response = requests.post(
        f"{backend_url}/api/researchers/A100/collaborations",
        json={"reference_institution_ids": ["I10"]},
        timeout=2,
    )

    assert response.status_code == 200
    assert response.json()["collaboration_count"] == 0
    assert response.json()["incomplete_work_ids"] == ["W1"]
    assert len(calls) == 1


def test_multiple_reference_institutions_count_external_work_once(
    backend_url, monkeypatch
):
    calls = mock_openalex(
        monkeypatch,
        [
            {
                "results": [
                    work("W1", "I10", "I11"),
                    work("W2", "I11", "I20"),
                ],
                "meta": {"next_cursor": None},
            }
        ],
    )

    response = requests.post(
        f"{backend_url}/api/researchers/A100/collaborations",
        json={"reference_institution_ids": ["I10", "I11"]},
        timeout=2,
    )

    assert response.status_code == 200
    assert response.json()["collaboration_count"] == 1
    assert response.json()["collaborative_work_ids"] == ["W2"]
    assert len(calls) == 1


@pytest.mark.parametrize(
    ("failure", "expected_status"),
    [(requests.ConnectionError("offline"), 502), (requests.Timeout("slow"), 504)],
)
def test_openalex_connection_failure_or_timeout_is_not_success(
    backend_url, monkeypatch, failure, expected_status
):
    mock_openalex(monkeypatch, [failure])

    response = requests.post(
        f"{backend_url}/api/researchers/A100/collaborations",
        json={"reference_institution_ids": ["I10"]},
        timeout=2,
    )

    assert response.status_code == expected_status
    assert "collaboration_count" not in response.json()


def test_all_pages_are_processed_and_duplicate_work_is_counted_once(
    backend_url, monkeypatch
):
    calls = mock_openalex(
        monkeypatch,
        [
            {
                "results": [work("W1", "I10", "I20"), work("W2", "I10")],
                "meta": {"next_cursor": "page-2"},
            },
            {
                "results": [work("W1", "I10", "I20"), work("W3", "I10", "I30")],
                "meta": {"next_cursor": None},
            },
        ],
    )

    response = requests.post(
        f"{backend_url}/api/researchers/A100/collaborations",
        json={"reference_institution_ids": ["I10"]},
        timeout=2,
    )

    assert response.status_code == 200
    assert response.json()["collaboration_count"] == 2
    assert response.json()["collaborative_work_ids"] == ["W1", "W3"]
    assert [call["cursor"] for call in calls] == ["*", "page-2"]
```

### Observações sobre adaptação

- O exemplo pressupõe Flask, um endpoint de análise ilustrativo e os campos OpenAlex `results`, `meta.next_cursor`, `id` e `authorships[].institutions[].id`. Confirme os campos, filtros e a paginação na documentação OpenAlex e no contrato efetivo da aplicação.
- Neste exemplo, os identificadores curtos (`I10`, `I20`) são fixtures didáticas; em produção, normalize os IDs de instituição para um único formato antes de comparar.
- O mock substitui `requests.get` usado pelo back-end, enquanto `requests.post` continua fazendo HTTP para o servidor local. Em back-ends executados em outro processo/container, injete um cliente HTTP mockável ou direcione o serviço a um servidor HTTP simulado.
- A rota responde com `502`/`504` conforme o exemplo. Se a aplicação usa outros códigos ou formato de erro, ajuste as asserções ao contrato existente; o requisito essencial é não apresentar uma falha externa como indicador válido.