Visão Geral do Sistema
O BioLib é um protótipo de sistema projetado para gerenciar acervos científicos e didáticos e rastrear a alocação temporária (empréstimo) desses materiais para pesquisadores acadêmicos. O projeto serve como uma ponte de aplicação prática voltada à indissociabilidade acadêmica, permitindo que as atividades de extensão universitária (atendimento externo e parcerias com institutos) e pesquisa gerenciem seus recursos de forma rastreável.

A interface gráfica foi construída com Tailwind CSS focando na alta scannability (escaneabilidade) visual, simulando um console ou terminal de gerenciamento de banco de dados moderno.

2. Modelagem Relacional de Dados (Arquitetura de Banco)
O banco de dados simulado é composto por três tabelas físicas principais estruturadas sob uma cardinalidade de Muitos para Muitos (N:M), onde a tabela emprestimos atua como uma tabela de junção/associativa.

       [pesquisadores] 1 -------- N [emprestimos] N -------- 1 [itens_acervo]
2.1. Dicionário de Dados
Tabela: pesquisadores
Armazena a entidade dos cientistas ou extensionistas autorizados a retirar materiais.

id_pesquisador (Integer, Primary Key): Identificador único sequencial do pesquisador.

nome (Varchar): Nome completo do usuário.

instituicao (Varchar): Sigla ou nome da instituição vinculada (ex: UFPR, Fiocruz).

email (Varchar): Endereço eletrônico de contato.

Tabela: itens_acervo
Armazena os objetos físicos, livros ou espécimes catalogados.

id_item (Integer, Primary Key): Identificador único sequencial físico.

codigo_catalogo (Varchar, Unique): Código alfa-numérico exclusivo de identificação visual (ex: BIO-001).

titulo_identificacao (Varchar): Descrição ou nome científico do item.

tipo_item (Varchar, Check Constraint): Restrição de domínio aceitando apenas os valores: Livro, Amostra Botanica, Amostra Zoologica ou Fossil.

localizacao_fisica (Varchar): Indicação de gaveta, estante ou vitrine onde o objeto reside.

status_item (Varchar): Estado atual do item (Disponivel ou Emprestado).

Tabela: emprestimos (Tabela Associativa)
Registra a transação e o vínculo histórico entre um pesquisador e um item.

id_emprestimo (Integer, Primary Key): Identificador único da transação.

id_pesquisador (Integer, Foreign Key): Aponta para pesquisadores(id_pesquisador).

id_item (Integer, Foreign Key): Aponta para itens_acervo(id_item).

data_retirada (Date): Registrada automaticamente com a data corrente do sistema.

data_devolucao_prevista (Date): Prazo limite estipulado pelo operador.

data_devolucao_real (Date, Nullable): Registra null enquanto o item estiver em posse do pesquisador, sendo atualizada com a data atual no momento da devolução.

3. Engenharia de Operações SQL (Simuladas)
Embora a aplicação execute códigos em JavaScript no ecossistema cliente (frontend), a lógica algorítmica imita estritamente as regras de álgebra relacional das seguintes queries SQL:

3.1. Inserção de Novo Item (DML - INSERT)
Ao submeter o formulário de cadastro de itens, o sistema executa uma validação idêntica à restrição de unicidade (UNIQUE KEY):

SQL
-- Lógica abstrata de inserção validada pelo script
INSERT INTO itens_acervo (codigo_catalogo, titulo_identificacao, tipo_item, localizacao_fisica, status_item)
VALUES ('BIO-401', 'Crânio de Panthera onca', 'Amostra Zoologica', 'Armário Biologia - Gaveta 3', 'Disponivel');
Tratamento de Exceção: Se .some() encontrar correspondência para o codigo_catalogo, o motor Javascript bloqueia a operação disparando um alerta de violação de integridade ([INTEGRITY CONSTRAINT VIOLATION]).

3.2. Registro de Empréstimo (Transação em Cascata)
A criação de um registro em emprestimos dispara simultaneamente uma atualização de estado (simulando uma Trigger ou uma transação atômica encapsulada):

SQL
BEGIN TRANSACTION;
  INSERT INTO emprestimos (id_pesquisador, id_item, data_retirada, data_devolucao_prevista, data_devolucao_real)
  VALUES (2, 1, CURRENT_DATE, '2026-06-15', NULL);

  UPDATE itens_acervo 
  SET status_item = 'Emprestado' 
  WHERE id_item = 1;
COMMIT;
3.3. Consulta Estruturada Avançada (Multi-JOIN)
O painel inferior renderiza o resultado de uma consulta relacional complexa que unifica dados das três tabelas físicas com base em chaves estrangeiras, filtrando apenas as pendências ativas:

SQL
SELECT p.nome, p.instituicao, i.codigo_catalogo, e.data_devolucao_prevista 
FROM emprestimos e
INNER JOIN pesquisadores p ON e.id_pesquisador = p.id_pesquisador
INNER JOIN itens_acervo i ON e.id_item = i.id_item
WHERE e.data_devolucao_real IS NULL;
Mapeamento em Código: O SGBD Virtual resolve essa busca usando métodos .filter() para a cláusula WHERE e .find() para interceptar os registros correspondentes do INNER JOIN.

4. Persistência de Dados e Ciclo de Vida
O ciclo de vida dos dados é gerenciado por uma camada de persistência local simulada no navegador:

Boot / Inicialização (obterSGBD): Verifica se a chave biolib_sgbd já existe no localStorage. Caso esteja vazia, ela injeta os registros de semente (DML Seed) contidos no objeto esquemaPadrao, garantindo que a aplicação nunca inicie sem dados de teste.

Sincronização de Estado (salvarSGBD): Toda alteração de estado (Inserção de itens, registros de empréstimos ou atualizações de devoluções) converte o banco de dados JSON em string e o grava no disco simulado do navegador (localStorage.setItem).

Auditoria / Reset (reiniciarBanco): Executa uma operação destrutiva controlada, limpando o armazenamento local para forçar o sistema a recriar as tabelas físicas em seu estado padrão na próxima atualização.
