# 📝 API de Registro de Atividades
Um projeto totalmente funcional, desenvolvido em PHP 8.1+ com Symfony, demonstrando uma API REST robusta utilizando princípios de DDD (Domain-Driven Design).

Este projeto foi desenhado para servir como um modelo de backend escalável, apresentando funcionalidades avançadas como histórico de status, categorias dinâmicas, e segurança via JWT.


### O que este projeto demonstra:
* Arquitetura DDD: Separação clara entre Domínio, Aplicação, Infraestrutura e Interface de Usuário.

* Persistência Avançada: Uso do Doctrine ORM com MySQL, implementando Soft-Delete e histórico de logs.

* Segurança: Autenticação via JWT e hashing de senhas.

* Qualidade de Código: Testes unitários e testes de integração .

* Padronização REST: Respostas JSON consistentes para sucesso e erro.
  
---

## Critérios de aceite

* Padrão de desenvolvimento DDD aplicado.
* Status Inicial: Toda tarefa nasce como pendente.
* Imutabilidade de Histórico: Alterações de status são registradas permanentemente na tabela TASK_STATUS_HISTORY.
* Prioridades: Codificadas como Enum (1=Alta, 2=Média, 3=Baixa).
* Soft-Delete: Tarefas "excluídas" recebem um timestamp em deleted_at, mantendo a integridade do histórico.

---

## Requisitos para Inicialização do Projeto

* **Linguagem**: PHP 8.1+
* **Framework**: Symfony 6.x
* **ORM**: Doctrine
* **Banco de dados**: MySQL

---
## Regras de Negócio 

| ID    | Regra                          | Descrição                                                                 | Justificativa                                                                                                   |
|------ |--------------------------------|---------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| RN-01 | Imutabilidade de Tarefa Inativa | A edição de dados e exclusão só são permitidas se o status for `em_andamento`. | Garante que o usuário só altere o escopo de tarefas ativas, preservando o histórico do que já foi feito ou não iniciado. |
| RN-02 | Prioridade Padrão               | Se a prioridade não for informada na criação, o sistema atribui automaticamente `2 (Normal)`. | Evita inconsistência no banco de dados e garante que toda tarefa tenha um peso para ordenação.                  |
| RN-03 | Bloqueio de Retrocesso          | É proibido transitar uma tarefa de `em_andamento` para `pendente`.           | Preserva o fluxo lógico de progresso; uma tarefa iniciada deve ser concluída ou cancelada, nunca “reiniciada”. |
| RN-04 | Exclusão Condicional            | O Soft Delete só é permitido se a tarefa estiver `em_andamento`.            | Previne a remoção acidental de tarefas finalizadas ou que ainda não começaram.                                  |
| RN-05 | Auditoria Seletiva              | Apenas alterações de **Status** geram log em `HISTORY`.                   | Evita poluição com alterações triviais, focando no ciclo de vida da tarefa.                                     |
| RN-06 | Escopo de Categorias            | Apenas Admin cria categorias Globais. Usuários comuns criam categorias Privadas. | Mantém a organização do sistema e evita poluição da lista global.                                               |
| RN-07 | Segurança na Associação         | O usuário só pode vincular tarefas a categorias que possui ou que são Globais. | Previne acesso indevido e manipulação de recursos de terceiros.                                                 |

---

## Definição Status

**pendente:**
- Task criada com data de criação.
- Nenhuma ação iniciada.
- Não permite edição nem exclusão.

**em_andamento:**
- Task iniciada pelo usuário.
- Permite edição e exclusão (soft delete).
- Permite transição para concluída.

**concluída:**
- Task finalizada com registro de data de conclusão.
- Registro imutável.
- Apenas visualização.

---


## Diagrama DB

```mermaid
erDiagram
    USER {
        int id PK "Autoincrement"
        string name "Not Null"
        string email "Not Null, Unique"
        string password_hash "Not Null"
        datetime created_at "Not Null"
        datetime updated_at "Not Null"
    }

    TASK {
        int id PK "Autoincrement"
        int user_id FK "Not Null"
        string name "Not Null"
        string description "Nullable"
        string status "Not Null"
        int priority "Not Null"
        datetime created_at "Not Null"
        datetime updated_at "Not Null"
        datetime completed_at "Nullable"
        datetime deleted_at "Nullable"
    }

    CATEGORY {
    int id PK "Autoincrement"
    int user_id FK "Nullable"
    string name "Not Null"
    datetime created_at "Not Null"
    }

    TASK_CATEGORY {
          int task_id PK, FK
          int category_id PK, FK
    }

    TASK_STATUS_HISTORY {
        int id PK "Autoincrement"
        int task_id FK "Not Null"
        string old_status "Not Null"
        string new_status "Not Null"
        datetime changed_at "Not Null"
    }

    USER ||--o{ TASK : owns
    TASK ||--o{ TASK_CATEGORY : has
    CATEGORY ||--o{ TASK_CATEGORY : groups
    TASK ||--o{ TASK_STATUS_HISTORY : logs
```

---

## Diagrama de fluxo

```mermaid
flowchart TD
  
    Start(["Início"]) --> Auth["Autenticação JWT"]
    Auth -- "Login Válido" --> UserSession["Sessão Usuário"]
    Auth -- "Falha" --> Error401["Erro 401: Unauthorized"]
    
    UserSession --> Dashboard["Dashboard Principal"]
    Dashboard -- "Operações" --> TaskFlow
    Dashboard -- "Operações" --> CategoryFlow
    Dashboard -- "Visualização" --> HistoryFlow

    subgraph TaskFlow ["Gestão de Tarefas"]
        direction TB
        
        T1["CRIAR TAREFA"] --> T1a["Definir Dados"]
        T1a --> T1b["Status Inicial: 'pendente'<br>Registra Histórico de Criação"]

        T2["ATUALIZAR TAREFA"] --> T0{"Tarefa Editável?"}
        T0 -- "Não: Deletada ou Concluída" --> T_Locked["Erro: Registro Imutável"]
        
        T0 -- "Sim" --> T2a{"Alterar Status?"}
        T2a -- "Não" --> T2e["Atualizar Nome/Descrição/Prioridade"]
        
        T2a -- "Sim" --> T2b{"Validar Transição?"}
        T2b -- "Retrocesso" --> T_Err["Erro: Retrocesso Proibido"]
        T2b -- "Inválida" --> T_Err2["Erro: Transição Não Permitida"]
        T2b -- "Válida" --> T2d["Atualizar Status<br>Registra Histórico de Alteração"]
        
        T2d --> T2f{"Novo status é 'concluída'?"}
        T2f -- "Sim" --> T4["Definir Data de Conclusão"]

        T3["EXCLUIR TAREFA"] --> T3a{"Status: 'em_andamento'?"}
        T3a -- "Não" --> T3b["Erro: Apenas tarefas em andamento"]
        T3a -- "Sim" --> T3c["Aplicar Soft Delete<br>Registra Histórico de Exclusão"]
    end

    subgraph CategoryFlow ["Gestão de Categorias"]
        C1["CRIAR CATEGORIA"] --> C1a{"É Admin?"}
        C1a -- "Sim" --> C1b["Pode criar Categoria Global"]
        C1a -- "Não" --> C1c["Cria Categoria Privada"]
        
        C2["ASSOCIAR TAREFA"] --> C2a{"Dono ou Categoria Global?"}
        C2a -- "Não" --> C2b["Erro 403: Forbidden"]
        C2a -- "Sim" --> C2c["Vincular Tarefa à Categoria"]
    end


    subgraph HistoryFlow ["Auditoria Imutável"]
        H1["VISUALIZAR HISTÓRICO"] --> H1a["Consulta Logs de Status"]
        H1a --> H1b["Registros não podem ser apagados ou alterados"]
    end
```

---

## 🔌 Formato de Resposta API

Sucesso (Ex: 201 Created):

```json
{
  "success": true,
  "status_code": 201,
  "message": "Tarefa criada com sucesso.",
  "data": { "id": 1, "name": "Estudar Symfony" }
}
```

Erro (Ex: 400 Bad Request):

```json
{
  "success": false,
  "message": "Status inválido",
  "errors": { ["Valor inválido"] }
}
```

---

## 🤝 Como contribuir

Encontrou um bug ou tem uma sugestão de melhoria?

* Abra uma Issue detalhando o problema.
* Envie um Pull Request referenciando a Issue.
* Certifique-se de que os testes estão passando: `php bin/phpunit`.


---

## ⭐ Gostou do projeto? 
Sinta-se à vontade para dar um fork e usar como base para seus próprios estudos


