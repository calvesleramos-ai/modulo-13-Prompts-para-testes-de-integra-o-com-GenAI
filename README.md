<WritingBlock id="63827" variant="document"># Testes de Integração com GenAI – Módulo 13
1. Sobre o projeto
Este repositório apresenta um experimento de utilização de Inteligência Artificial Generativa para apoiar a elaboração de testes de integração de software.

A atividade está vinculada ao módulo 13 – Testes de Software Automatizados com Inteligência Artificial Generativa, da pós-graduação em IA Aplicada à Engenharia de Software.
2. Cenário utilizado
O cenário considera o desenvolvimento de uma ferramenta de inteligência para identificação de pesquisadores e análise de publicações científicas em colaboração com outras instituições.

A solução utiliza a API OpenAlex para recuperar informações sobre pesquisadores, publicações e vínculos institucionais.
3. Objetivo
Avaliar a utilização de duas técnicas de prompting para geração de casos de testes de integração:
- Zero-shot: geração de cenários sem exemplos prévios.
- Few-shot: geração de cenários orientada por exemplos previamente fornecidos.
4. Ferramenta de IA
- LLM: GitHub Copilot.
- Ambiente: Visual Studio Code.
- Linguagem considerada para automação: Python.
- Framework de testes: Pytest.
- Integração HTTP: Requests.
5. Estrutura do repositório
Pasta	Conteúdo
docs/	Contexto e análise comparativa
prompts/	Prompts utilizados nos experimentos
resultados/	Respostas originais geradas pelo Copilot
testes/	Casos de testes de integração organizados