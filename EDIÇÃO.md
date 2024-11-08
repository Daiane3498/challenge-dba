
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

### 1. Identifique as chaves primárias e estrangeiras necessárias para garantir a integridade referencial. Defina-as corretamente. 

RESPOSTA:
### Tabela tenant
Chave primária: id (do tipo SERIAL).

### Tabela person
Chave primária: id (do tipo SERIAL).

### Tabela institution
Chave primária: id (do tipo SERIAL).
Chave estrangeira: tenant_id (relaciona-se com a tabela tenant).

### Tabela course
Chave primária: id (do tipo SERIAL).
Chave estrangeira: tenant_id (relaciona-se com a tabela tenant).
Chave estrangeira: institution_id (relaciona-se com a tabela institution).

### Tabela enrollment
Chave primária: id (do tipo SERIAL).
Chaves estrangeiras:
tenant_id (relaciona-se com a tabela tenant).
institution_id (relaciona-se com a tabela institution).
person_id (relaciona-se com a tabela person).
course_id (relaciona-se com a tabela course).

### 2. Construa índices que consideras essenciais para operações básicas do banco e de consultas possíveis para a estrutura sugerida.

RESPOSTA:
### Tabela tenant:
Índice na chave primária (id)

CREATE INDEX idx_tenant_id ON tenant(id);

### Tabela person:
Índices em id, name e no campo metadata (GIN para JSONB):

CREATE INDEX idx_person_id ON person(id);

CREATE INDEX idx_person_name ON person(name);

CREATE INDEX idx_person_metadata ON person USING GIN(metadata);

### Tabela institution:
Índices em tenant_id e id:

CREATE INDEX idx_institution_tenant_id ON institution(tenant_id);

CREATE INDEX idx_institution_id ON institution(id);

### Tabela course:
Índices em tenant_id, institution_id e id:

CREATE INDEX idx_course_tenant_institution ON course(tenant_id, institution_id);

CREATE INDEX idx_course_id ON course(id);

### Tabela enrollment:
Índices compostos em (tenant_id, institution_id, person_id), status, enrollment_date, e is_deleted:

CREATE INDEX idx_enrollment_tenant_institution_person ON enrollment(tenant_id, institution_id, person_id);

CREATE INDEX idx_enrollment_status ON enrollment(status);

CREATE INDEX idx_enrollment_enrollment_date ON enrollment(enrollment_date);

CREATE INDEX idx_enrollment_is_deleted ON enrollment(is_deleted);


### 3. Considere que em enollment só pode existir um único person_id por tenant e institution. Mas institution poderá ser nulo. Como garantir a integridade desta regra?

RESPOSTA:Para garantir que um aluno (person_id) só possa ser matriculado uma vez em um curso para um mesmo tenant, com institution podendo ser nulo, utilizamos uma restrição de unicidade. A função COALESCE é usada para tratar institution_id como um valor fixo (como -1) quando for NULL, garantindo que a combinação de tenant_id, institution_id (ou -1 para NULL) e person_id seja única.
```sql
CREATE TABLE enrollment (
    id SERIAL PRIMARY KEY,                     -- ID único para a matrícula
    tenant_id INTEGER NOT NULL,                -- ID do tenant (obrigatório)
    institution_id INTEGER,                    -- ID da instituição (pode ser nulo)
    person_id INTEGER NOT NULL,                -- ID da pessoa (aluno) (obrigatório)
    course_id INTEGER NOT NULL,                -- ID do curso (obrigatório)
    enrollment_date DATE,                      -- Data da matrícula
    status VARCHAR(20),                        -- Status da matrícula (ex: "ativo", "inativo")
    is_deleted BOOLEAN DEFAULT FALSE,          -- Exclusão lógica (marca a matrícula como deletada sem removê-la)
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE,           -- Relaciona com a tabela tenant
    FOREIGN KEY (institution_id) REFERENCES institution(id) ON DELETE CASCADE, -- Relaciona com a tabela institution
    FOREIGN KEY (person_id) REFERENCES person(id) ON DELETE CASCADE,           -- Relaciona com a tabela person (aluno)
    FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,           -- Relaciona com a tabela course
     CONSTRAINT unique_enrollment UNIQUE (tenant_id, COALESCE(institution_id, -1), person_id), -- Restrição de unicidade
      CHECK (is_deleted IN (FALSE, TRUE))   -- Garante que is_deleted só pode ter valores TRUE ou FALSE
```
### 4. Caso eu queira incluir conceitos de exclusão lógica na tabela enrollment. Como eu poderia fazer? Quais as alterações necessárias nas definições anteriores?

RESPOSTA:A exclusão lógica é implementada com o campo is_deleted, que indica se o registro foi excluído logicamente (marcado como TRUE), sem ser removido fisicamente. Para garantir a eficiência nas consultas, um índice sobre is_deleted deve ser criado para otimizar filtros de registros não excluídos.
```sql
CREATE TABLE enrollment (
   id SERIAL PRIMARY KEY,                     -- ID único para a matrícula
    tenant_id INTEGER NOT NULL,                -- ID do tenant (obrigatório)
    institution_id INTEGER,                    -- ID da instituição (pode ser nulo)
    person_id INTEGER NOT NULL,                -- ID da pessoa (aluno) (obrigatório)
    course_id INTEGER NOT NULL,                -- ID do curso (obrigatório)
    enrollment_date DATE,                      -- Data da matrícula
    status VARCHAR(20),                        -- Status da matrícula (ex: "ativo", "inativo")
    is_deleted BOOLEAN DEFAULT FALSE,          -- Exclusão lógica (marca a matrícula como deletada sem removê-la)
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE,           -- Relaciona com a tabela tenant
    FOREIGN KEY (institution_id) REFERENCES institution(id) ON DELETE CASCADE, -- Relaciona com a tabela institution
    FOREIGN KEY (person_id) REFERENCES person(id) ON DELETE CASCADE,           -- Relaciona com a tabela person (aluno)
    FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,           -- Relaciona com a tabela course
    CONSTRAINT unique_enrollment UNIQUE (tenant_id, COALESCE(institution_id, -1), person_id), -- Restrição de unicidade
    CHECK (is_deleted IN (FALSE, TRUE))  -- Garante que is_deleted só pode ter valores TRUE ou FALSE
);
```
Índice para otimizar consultas de exclusão lógica
CREATE INDEX idx_enrollment_is_deleted ON enrollment(is_deleted);


### 5. Construa uma consulta que retorne o número de matrículas por curso em uma determinada instituição.Filtre por tenant_id e institution_id obrigatoriamente. Filtre também por uma busca qualquer -full search - no campo metadata da tabela person que contém informações adicionais no formato JSONB. Considere aqui também a exclusão lógica e exiba somente registros válidos.

RESPOSTA: A consulta tem como objetivo retornar o número de matrículas por curso em uma instituição específica dentro de um tenant. Ela considera apenas registros válidos, excluindo os alunos que foram deletados logicamente. Além disso, a consulta realiza uma busca no campo metadata (tipo JSONB) da tabela person, permitindo filtrar os dados conforme o conteúdo armazenado nesse campo.
```sql
SELECT
    c.name AS curso,                         -- Nome do curso
    COUNT(m.id) AS num_matriculas            -- Contagem de matrículas por curso
FROM
    matricula m                              -- Tabela de matrículas
JOIN
    curso c ON c.id = m.course_id            -- Tabela de cursos
JOIN
    person p ON p.id = m.person_id          -- Tabela de alunos (pessoas)
WHERE
    m.tenant_id = :tenant_id                -- Filtro obrigatório por tenant_id
    AND m.institution_id = :institution_id  -- Filtro obrigatório por institution_id
    AND p.deleted = FALSE                   -- Exclui registros deletados logicamente
    AND p.metadata @> :search_term          -- Filtro no campo JSONB (metadata)
GROUP BY
    c.name                                   -- Agrupa os resultados pelo nome do curso
ORDER BY
    num_matriculas DESC;                    -- Ordena os cursos pela quantidade de matrículas em ordem decrescente
```

### 6. Construa uma consulta que retorne os alunos de um curso em uma tenant e institution específicos. Esta é uma consulta para atender a requisição que tem por objetivo alimentar uma listagem de alunos em determinado curso. Tenha em mente que poderá retornar um número grande de registros por se tratar de um curso EAD. Use boas práticas. Considere aqui também a exclusão lógica e exiba somente registros válidos.

RESPOSTA:Considerando que a instituição e o tenant específicos são fornecidos como parâmetros. Como se trata de um curso EAD, que pode ter muitos registros, a consulta deve ser otimizada e considerar a exclusão lógica, exibindo apenas alunos válidos.
```sql
SELECT
    p.id AS aluno_id,                     -- ID do aluno
    p.nome AS aluno_nome,                 -- Nome do aluno
    c.name AS curso_nome                  -- Nome do curso
FROM
    matricula m                            -- Tabela de matrículas
JOIN
    person p ON p.id = m.person_id         -- Tabela de alunos (pessoas)
JOIN
    curso c ON c.id = m.course_id          -- Tabela de cursos
WHERE
    m.tenant_id = :tenant_id              -- Filtro obrigatório por tenant_id
    AND m.institution_id = :institution_id -- Filtro obrigatório por institution_id
    AND m.course_id = :course_id          -- Filtro obrigatório por course_id
    AND p.deleted = FALSE                 -- Exclui registros deletados logicamente
ORDER BY
    p.nome;                               -- Ordena os resultados por nome do aluno (ou outro critério)
```

### 7. Suponha que decidimos particionar a tabela enrollment. Desenvolva esta ideia. Reescreva a definição da tabela por algum critério que julgues adequado. Faça todos os ajustes necessários e comente-os.

RESPOSTA: O particionamento da tabela enrollment pode aprimorar o desempenho, especialmente em cenários com grandes volumes de dados. Ele pode ser feito com base em critérios lógicos, como tenant_id ou enrollment_date, dependendo das necessidades específicas das consultas.

Exemplo de particionamento por tenant_id:
```sql
CREATE TABLE enrollment (
    id SERIAL PRIMARY KEY,                    -- ID da matrícula (chave primária)
    tenant_id INTEGER NOT NULL,               -- ID do tenant (não pode ser nulo)
    institution_id INTEGER,                   -- ID da instituição (pode ser nulo)
    person_id INTEGER NOT NULL,               -- ID da pessoa (aluno) (não pode ser nulo)
    course_id INTEGER NOT NULL,               -- ID do curso (não pode ser nulo)
    enrollment_date DATE,                     -- Data de matrícula
    status VARCHAR(20),                       -- Status da matrícula
    is_deleted BOOLEAN DEFAULT FALSE          -- Campo para exclusão lógica (não exclui fisicamente)
) PARTITION BY LIST (tenant_id);              -- Particionamento por lista, baseado no tenant_id
```


Partições para diferentes tenants
```sql
CREATE TABLE enrollment_tenant_1 PARTITION OF enrollment FOR VALUES IN (1);
CREATE TABLE enrollment_tenant_2 PARTITION OF enrollment FOR VALUES IN (2);
```
Esse particionamento é vantajoso para consultas que focam em um tenant_id específico, pois o banco de dados irá pesquisar apenas nas partições relevantes. 
Uma alternativa seria particionar a tabela com base na data de matrícula (enrollment_date), caso as consultas se concentrem em dados temporais.

### 8. Sinta-se a vontade para sugerir e aplicar qualquer ajuste que achares relevante. Comente-os


## Critérios de avaliação
- Organização, clareza e lógica
- Utilização de boas práticas
- Documentação justificando o porquê das escolhas

## Instruções de entrega
1. Crie um fork do repositório no seu GitHub
2. Faça o push do código desenvolvido no seu Github
3. Informe ao recrutador quando concluir o desafio junto com o link do repositório
4. Após revisão do projeto, em conjunto com a equipe técnica, deixe seu repositório privado
