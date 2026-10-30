<WritingBlock id="63824" variant="document"># Experimento 2 – Prompt Few-shot
Objetivo
Gerar novos cenários de testes de integração utilizando exemplos previamente definidos para orientar o formato e a lógica dos casos de teste.
Prompt utilizado
Atue como um engenheiro de software especialista em testes automatizados de integração utilizando Python e Pytest.

Estou desenvolvendo uma ferramenta que consulta a API OpenAlex para identificar pesquisadores e analisar suas publicações em colaboração com outras instituições.

O back-end consulta a API externa, processa os dados de autoria e instituições e disponibiliza os indicadores para o front-end.

Utilize os exemplos abaixo como referência de estrutura e nível de detalhamento para gerar novos casos de teste.
Exemplo 1 – Integração com a API OpenAlex
- Cenário: pesquisa de um pesquisador existente.
- Entrada: identificador OpenAlex válido de um pesquisador.
- Execução: o back-end realiza uma requisição à API OpenAlex para recuperar os dados do pesquisador.
- Resultado esperado: resposta HTTP 200, contendo o identificador, nome e informações disponíveis sobre o pesquisador.
Exemplo 2 – Cálculo de publicações em colaboração
- Cenário: pesquisador com publicações realizadas em parceria com outras instituições.
- Entrada: conjunto de três publicações, sendo duas com participação de autores vinculados a instituições diferentes da instituição de referência e uma publicação exclusivamente vinculada à instituição de referência.
- Execução: o back-end processa as publicações e identifica os vínculos institucionais dos autores.
- Resultado esperado: o sistema contabiliza duas publicações em colaboração, sem duplicar registros, mesmo quando uma publicação possui vários autores ou instituições parceiras.