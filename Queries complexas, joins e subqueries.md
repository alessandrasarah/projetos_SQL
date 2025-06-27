# projetos_SQL
**Schema (PostgreSQL v17)**

    -- Esquema do banco de dados
    CREATE TABLE clientes (
      id INT PRIMARY KEY,
      nome VARCHAR(50),
      cidade VARCHAR(50)
    );
    
    CREATE TABLE pedidos (
      id INT PRIMARY KEY,
      cliente_id INT,
      data DATE,
      valor DECIMAL,
      FOREIGN KEY (cliente_id) REFERENCES clientes(id)
    );
    
    CREATE TABLE produtos (
      id INT PRIMARY KEY,
      nome VARCHAR(50),
      categoria VARCHAR(30)
    );
    
    CREATE TABLE pedidos_produtos (
      pedido_id INT,
      produto_id INT,
      quantidade INT,
      PRIMARY KEY (pedido_id, produto_id),
      FOREIGN KEY (pedido_id) REFERENCES pedidos(id),
      FOREIGN KEY (produto_id) REFERENCES produtos(id)
    );
    
    -- Inserção de dados
    INSERT INTO clientes VALUES (1, 'Maria', 'São Paulo');
    INSERT INTO clientes VALUES (2, 'João', 'Rio de Janeiro');
    INSERT INTO clientes VALUES (3, 'Ana', 'Curitiba');
    
    INSERT INTO pedidos VALUES (101, 1, '2024-01-10', 250.00);
    INSERT INTO pedidos VALUES (102, 2, '2024-02-15', 150.00);
    INSERT INTO pedidos VALUES (103, 1, '2024-03-20', 100.00);
    
    INSERT INTO produtos VALUES (10, 'Notebook', 'Eletrônicos');
    INSERT INTO produtos VALUES (11, 'Fone de ouvido', 'Eletrônicos');
    INSERT INTO produtos VALUES (12, 'Camiseta', 'Moda');
    
    INSERT INTO pedidos_produtos VALUES (101, 10, 1);
    INSERT INTO pedidos_produtos VALUES (101, 11, 2);
    INSERT INTO pedidos_produtos VALUES (102, 12, 3);
    INSERT INTO pedidos_produtos VALUES (103, 10, 1);

---SUBQUERIES

**Query #1**

    select *
    from pedidos
    where valor > (select avg(valor) from pedidos);

| id  | cliente_id | data       | valor  |
| --- | ---------- | ---------- | ------ |
| 101 | 1          | 2024-01-10 | 250.00 |

**Query #2**

    select cliente_id, media
    from (
      select cliente_id, avg(valor) as media
      from pedidos
      group by cliente_id
      )
      where media >50

| cliente_id | media                |
| ---------- | -------------------- |
| 2          | 150.0000000000000000 |
| 1          | 175.0000000000000000 |

**Query #3**

    select nome
      from clientes c
      where exists (
        select 1 from pedidos p
        where p.cliente_id = c.id and p.valor > 200
        );

| nome  |
| ----- |
| Maria |

**Query #4**

    -- Criando cte
    with total_por_clientes as(
      select
      cliente_id,
      sum(valor) as total --soma o valor total dos pedidos por clientes
      from pedidos
      group by 
      cliente_id
      )
      
      --consulta principal
      select
      c.nome, --nome do cliente
      t.total --total gasto
      
      from  total_por_clientes t --usa o cte como base da consulta (com o alias t)
      join
      clientes c on c.id = t.cliente_id -- junta com a tabela de clientes para trazer o nome

| nome  | total  |
| ----- | ------ |
| Maria | 350.00 |
| João  | 150.00 |

--- FUNÇÕES JANELA

--Analisar a evolução das compras dos clientes ao longo do tempo
--Função coalesce para substituir valores nulos

**Query #5**

    select
    c.nome,
    coalesce(sum(p.valor),0) as total,
    rank() over(
      order by coalesce(sum(p.valor),0) desc -- cria um ranking baseado no total gasto (do maior para o menor)
      ) as posicao
      
      from
      clientes c
      left join 
      pedidos p
      on c.id = p.cliente_id
      group by
      c.nome

| nome  | total  | posicao |
| ----- | ------ | ------- |
| Maria | 350.00 | 1       |
| João  | 150.00 | 2       |
| Ana   | 0      | 3       |

**Query #6**

    select
    cliente_id,
    data,
    valor,
    lag(valor) over(
      partition by cliente_id --faz a comparação dentro de cada cliente
      order by data
      ) as anterior
      from pedidos;

| cliente_id | data       | valor  | anterior |
| ---------- | ---------- | ------ | -------- |
| 1          | 2024-01-10 | 250.00 |          |
| 1          | 2024-03-20 | 100.00 | 250.00   |
| 2          | 2024-02-15 | 150.00 |          |

**Query #7**

    select cliente_id, data, valor,
    rank() over(partition by cliente_id order by valor desc) as ranking_interno
    from pedidos;

| cliente_id | data       | valor  | ranking_interno |
| ---------- | ---------- | ------ | --------------- |
| 1          | 2024-01-10 | 250.00 | 1               |
| 1          | 2024-03-20 | 100.00 | 2               |
| 2          | 2024-02-15 | 150.00 | 1               |

**Query #8**

    select
    cliente_id,
    data,
    valor,
    lead(valor) over (
      partition by cliente_id
      order by data
      )
      As proxima_compra
      from pedidos;

| cliente_id | data       | valor  | proxima_compra |
| ---------- | ---------- | ------ | -------------- |
| 1          | 2024-01-10 | 250.00 | 100.00         |
| 1          | 2024-03-20 | 100.00 |                |
| 2          | 2024-02-15 | 150.00 |                |

**Query #9**

    -- CTE 1: calcular o total gasto por cada cliente
    with total_por_cliente as (
      select
        cliente_id,
        sum(valor) as total -- soma todos os pedidos
      from pedidos
      group by cliente_id
    ),
    
    -- CTE 2: ranqueia os clientes com base no total que eles já gastaram
    ranking as (
      select 
        cliente_id,
        total,
        rank() over (
          order by total desc -- criar o ranking do maior para o menor
        ) as posicao
      from total_por_cliente
    ),
    
    -- CTE 3: encontrar a maior compra individual de cada cliente
    maior_compra as (
      select
        cliente_id,
        max(valor) as maior_valor
      from pedidos
      group by cliente_id
    )
    
    -- Consulta final
    select
      c.nome,
      r.total,
      r.posicao,
      m.maior_valor
    from ranking r
    join clientes c on c.id = r.cliente_id
    left join maior_compra m on m.cliente_id = c.id;

| nome  | total  | posicao | maior_valor |
| ----- | ------ | ------- | ----------- |
| Maria | 350.00 | 1       | 250.00      |
| João  | 150.00 | 2       | 150.00      |



