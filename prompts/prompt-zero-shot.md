<WritingBlock id="63822" variant="document"># Experimento 1 – Prompt Zero-shot
Objetivo
Gerar cenários de testes de integração para uma ferramenta de identificação de pesquisadores e análise de colaboração científica utilizando a API OpenAlex, sem fornecer exemplos prévios de testes.
Prompt utilizado
Atue como um engenheiro de software especialista em testes automatizados de integração de APIs REST.

Considere uma ferramenta que utiliza a API da OpenAlex para identificar pesquisadores relevantes e analisar suas publicações em colaboração com outras instituições.

A arquitetura possui três componentes: um front-end, um back-end responsável pelo processamento das informações e integração com a API externa da OpenAlex.

O sistema deve permitir pesquisar pesquisadores, recuperar suas publicações, identificar as instituições associadas aos autores e calcular a quantidade de publicações realizadas em colaboração com instituições diferentes daquela de referência.

Com base nesse cenário, gere pelo menos 10 casos de testes de integração que contemplem:
1. Comunicação entre o front-end e o back-end.
2. Comunicação entre o back-end e a API OpenAlex.
3. Recuperação e processamento de publicações e vínculos institucionais.
4. Cálculo correto da quantidade de publicações em colaboração, considerando publicações únicas e evitando duplicidades.
5. Tratamento de respostas vazias e dados incompletos.
6. Tratamento de falhas de comunicação, erros HTTP, timeout e limitação de requisições da API externa.
7. Tratamento de paginação dos resultados da OpenAlex.
Para cada caso de teste, apresente: identificador, objetivo, pré-condições, dados de entrada, etapas de execução, resultado esperado e critério de aprovação.

Organize os resultados em uma tabela e priorize cenários que possam ser automatizados utilizando Python, Pytest e Requests.

Considere a utilização de mocks para simular respostas da API externa quando necessário.

Não invente endpoints ou campos da OpenAlex. Quando necessário, indique as premissas que precisam ser confirmadas na documentação oficial da API.</WritingBlock>