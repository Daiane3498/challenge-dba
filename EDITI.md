
+A Educação - Desafio PostgreSQL
===================

[![N|Solid](https://maisaedu.com.br/hubfs/site-grupo-a/logo-mais-a-educacao.svg)](https://maisaedu.com.br/) 

O objetivo deste desafio é avaliar algumas competências técnicas consideradas fundamentais para candidatos ao cargo de DBA na Maior Plataforma de Educação do Brasil.

Será solicitado ao candidato que realize algumas tarefas baseadas em estrutura incompleta de tabelas relacionadas neste documento. Considere o PostgreSQL como SGDB ao aplicar conceitos e validações.

## Contexto

Você está lidando com um banco de dados multi-tenant que armazena informações acadêmicas sobre pessoas, instituições, cursos e matrículas de diferentes clientes (tenants). Sua tarefa será estruturar as tabelas, criar chaves primárias e estrangeiras, otimizar consultas, entre outras atividades. O desafio também envolve manipulação de dados em formato `jsonb`, com a necessidade implícita de construir índices apropriados.

## Estrutura das Tabelas

### 1. Tabela `tenant`
Representa diferentes clientes que utilizam o sistema. Pode ter cerca de 100 registros.

```sql
CREATE TABLE tenant (
    id SERIAL,
    name VARCHAR(100),
    description VARCHAR(255)
);
```
### 2.	Tabela `person` 
Contém informações sobre indivíduos. Esta tabela não está associada diretamente a um tenant. Estima-se ter 5.000.000 registros.

```sql
CREATE TABLE person (
    id SERIAL,
    name VARCHAR(100),
    birth_date DATE,
    metadata JSONB
);
```
### 3.	Tabela `institution`
Armazena detalhes sobre instituições associadas a diferentes tenants. Esta tabela terá aproximadamente 1.000 registros.

```sql
CREATE TABLE institution (
    id SERIAL,
    tenant_id INTEGER,
    name VARCHAR(100),
    location VARCHAR(100),
    details JSONB
);
```
### 4.	Tabela `course` 
Contém informações sobre cursos oferecidos por instituições, também associadas a um tenant. Deve ter cerca de 5.000 registros.

```sql
CREATE TABLE course (
    id SERIAL,
    tenant_id INTEGER,
    institution_id INTEGER,
    name VARCHAR(100),
    duration INTEGER,
    details JSONB
);
```

### 5.	Tabela `enrollment`
Armazena informações de matrículas, associadas a um tenant. Esta é a tabela com maior volume de dados, com cerca de 100.000.000 registros.

```sql
CREATE TABLE enrollment (
    id SERIAL,
    tenant_id INTEGER,
    institution_id INTEGER,
    person_id INTEGER,
    enrollment_date DATE,
    status VARCHAR(20)
);
```

## Tarefas

1. Identifique as chaves primárias e estrangeiras necessárias para garantir a integridade referencial. Defina-as corretamente. 

RESPOSTA:
--Tabela tenant
Chave primária: id (do tipo SERIAL).

--Tabela person
Chave primária: id (do tipo SERIAL).

--Tabela institution
Chave primária: id (do tipo SERIAL).
Chave estrangeira: tenant_id (relaciona-se com a tabela tenant).

--Tabela course
Chave primária: id (do tipo SERIAL).
Chave estrangeira: tenant_id (relaciona-se com a tabela tenant).
Chave estrangeira: institution_id (relaciona-se com a tabela institution).

--Tabela enrollment
Chave primária: id (do tipo SERIAL).
Chaves estrangeiras:
tenant_id (relaciona-se com a tabela tenant).
institution_id (relaciona-se com a tabela institution).
person_id (relaciona-se com a tabela person).
course_id (relaciona-se com a tabela course).

2. Construa índices que consideras essenciais para operações básicas do banco e de consultas possíveis para a estrutura sugerida.

RESPOSTA:Índices para a tabela person:
Índice GIN no campo metadata (para otimizar a pesquisa por dados JSONB):

CREATE INDEX idx_person_metadata ON person USING GIN(metadata);

--Índices para a tabela enrollment:
Considerando o grande volume de dados da tabela enrollment (aproximadamente 100 milhões de registros), é importante adicionar índices para facilitar a busca por tenant_id, institution_id, person_id e status (que serão frequentemente filtrados nas consultas).

--Índices sobre as chaves estrangeiras
CREATE INDEX idx_enrollment_tenant_institution_person ON enrollment(tenant_id, institution_id, person_id);
CREATE INDEX idx_enrollment_status ON enrollment(status);
CREATE INDEX idx_enrollment_enrollment_date ON enrollment(enrollment_date);
CREATE INDEX idx_enrollment_is_deleted ON enrollment(is_deleted);

3. Considere que em enollment só pode existir um único person_id por tenant e institution. Mas institution poderá ser nulo. Como garantir a integridade desta regra?

RESPOSTA:CONSTRAINT unique_enrollment UNIQUE (tenant_id, institution_id, person_id): Esta restrição garante que a combinação de tenant_id, institution_id e person_id seja única na tabela enrollment, evitando que um aluno seja matriculado mais de uma vez em um mesmo curso dentro de uma instituição para o mesmo tenant.

CREATE TABLE enrollment (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    institution_id INTEGER,
    person_id INTEGER NOT NULL,
    course_id INTEGER NOT NULL,
    enrollment_date DATE,
    status VARCHAR(20),
    is_deleted BOOLEAN DEFAULT FALSE,  -- Exclusão lógica
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE,
    FOREIGN KEY (institution_id) REFERENCES institution(id) ON DELETE CASCADE,
    FOREIGN KEY (person_id) REFERENCES person(id) ON DELETE CASCADE,
    FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,
    CONSTRAINT unique_enrollment UNIQUE (tenant_id, institution_id, person_id),  -- Garantia de unicidade
    CHECK (is_deleted IN (FALSE, TRUE))  -- Garantir valores válidos para is_deleted
);


Exclusão lógica: O campo is_deleted permite marcar registros como excluídos logicamente, sem removê-los fisicamente da base de dados.

4. Caso eu queira incluir conceitos de exclusão lógica na tabela enrollment. Como eu poderia fazer? Quais as alterações necessárias nas definições anteriores?

RESPOSTA:is_deleted BOOLEAN DEFAULT FALSE: Marca a matrícula como excluída logicamente. O valor padrão é FALSE (não excluído).
Exclusão lógica: Ao invés de deletar fisicamente o registro, você só altera o valor de is_deleted para TRUE.

CREATE TABLE enrollment (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    institution_id INTEGER,
    person_id INTEGER NOT NULL,
    course_id INTEGER NOT NULL,
    enrollment_date DATE,
    status VARCHAR(20),
    is_deleted BOOLEAN DEFAULT FALSE,  -- Exclusão lógica
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE,
    FOREIGN KEY (institution_id) REFERENCES institution(id) ON DELETE CASCADE,
    FOREIGN KEY (person_id) REFERENCES person(id) ON DELETE CASCADE,
    FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,
    CONSTRAINT unique_enrollment UNIQUE (tenant_id, institution_id, person_id),
    CHECK (is_deleted IN (FALSE, TRUE))  -- Garantir valores válidos para is_deleted
);



5. Construa uma consulta que retorne o número de matrículas por curso em uma determinada instituição.Filtre por tenant_id e institution_id obrigatoriamente. Filtre também por uma busca qualquer -full search - no campo metadata da tabela person que contém informações adicionais no formato JSONB. Considere aqui também a exclusão lógica e exiba somente registros válidos.

RESPOSTA: tenant_id e institution_id são obrigatórios.
Exclui registros de alunos deletados (p.deleted = FALSE).
Busca no metadata: Filtra com uma expressão JSONB no campo metadata da tabela person.
Contagem e agrupamento: Conta o número de matrículas por curso (COUNT(m.id)) e agrupa os resultados pelo nome do curso (c.name).

SELECT
    c.name AS curso,
    COUNT(m.id) AS num_matriculas
FROM
    matricula m
JOIN
    curso c ON c.id = m.course_id
JOIN
    person p ON p.id = m.person_id
WHERE
    m.tenant_id = :tenant_id            -- Filtro por tenant_id
    AND m.institution_id = :institution_id  -- Filtro por institution_id
    AND p.deleted = FALSE                 -- Exclui registros deletados logicamente
    AND p.metadata @> :search_term       -- Busca no campo metadata (substituir por expressão JSONB)
GROUP BY
    c.name
ORDER BY
    num_matriculas DESC;

6. Construa uma consulta que retorne os alunos de um curso em uma tenant e institution específicos. Esta é uma consulta para atender a requisição que tem por objetivo alimentar uma listagem de alunos em determinado curso. Tenha em mente que poderá retornar um número grande de registros por se tratar de um curso EAD. Use boas práticas. Considere aqui também a exclusão lógica e exiba somente registros válidos.

RESPOSTA:Filtros: Filtra por tenant_id, institution_id e course_id específicos.
Exclusão lógica: Filtra registros excluídos logicamente usando p.deleted = FALSE.
Ordenação: Ordena os alunos por nome.

SELECT
    p.id AS aluno_id,
    p.nome AS aluno_nome,
    c.name AS curso_nome
FROM
    matricula m
JOIN
    person p ON p.id = m.person_id
JOIN
    curso c ON c.id = m.course_id
WHERE
    m.tenant_id = :tenant_id            -- Filtro por tenant_id
    AND m.institution_id = :institution_id  -- Filtro por institution_id
    AND m.course_id = :course_id          -- Filtro por curso_id
    AND p.deleted = FALSE                 -- Exclui registros deletados logicamente
ORDER BY
    p.nome;                              -- Ordena por nome (ou outro critério)

7. Suponha que decidimos particionar a tabela enrollment. Desenvolva esta ideia. Reescreva a definição da tabela por algum critério que julgues adequado. Faça todos os ajustes necessários e comente-os.

RESPOSTA: O particionamento da tabela enrollment pode aprimorar o desempenho, especialmente em cenários com grandes volumes de dados. Ele pode ser feito com base em critérios lógicos, como tenant_id ou enrollment_date, dependendo das necessidades específicas das consultas.

Exemplo de particionamento por tenant_id:
CREATE TABLE enrollment (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    institution_id INTEGER,
    person_id INTEGER NOT NULL,
    course_id INTEGER NOT NULL,
    enrollment_date DATE,
    status VARCHAR(20),
    is_deleted BOOLEAN DEFAULT FALSE
) PARTITION BY LIST (tenant_id);

-- Partições para diferentes tenants
CREATE TABLE enrollment_tenant_1 PARTITION OF enrollment FOR VALUES IN (1);
CREATE TABLE enrollment_tenant_2 PARTITION OF enrollment FOR VALUES IN (2);

Esse particionamento é vantajoso para consultas que focam em um tenant_id específico, pois o banco de dados irá pesquisar apenas nas partições relevantes. 
Uma alternativa seria particionar a tabela com base na data de matrícula (enrollment_date), caso as consultas se concentrem em dados temporais.

8. Sinta-se a vontade para sugerir e aplicar qualquer ajuste que achares relevante. Comente-os


## Critérios de avaliação
- Organização, clareza e lógica
- Utilização de boas práticas
- Documentação justificando o porquê das escolhas

## Instruções de entrega
1. Crie um fork do repositório no seu GitHub
2. Faça o push do código desenvolvido no seu Github
3. Informe ao recrutador quando concluir o desafio junto com o link do repositório
4. Após revisão do projeto, em conjunto com a equipe técnica, deixe seu repositório privado
