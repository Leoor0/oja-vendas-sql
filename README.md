-- =============================================
--  Projeto: Loja de Roupas (Camisas, Calças, Vestidos)
--  Padrão: Venda + Itens da Venda
--  Dialeto: MySQL 8.x
-- =============================================

-- (Opcional) Criar e usar um schema dedicado
-- CREATE DATABASE IF NOT EXISTS loja_db DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
-- USE loja_db;

-- Ordem de drop para evitar problemas de FK
DROP TRIGGER IF EXISTS trg_itens_venda_ai;
DROP TRIGGER IF EXISTS trg_itens_venda_au;
DROP TRIGGER IF EXISTS trg_itens_venda_ad;
DROP PROCEDURE IF EXISTS recalcula_total;
DROP PROCEDURE IF EXISTS abrir_venda;
DROP TABLE IF EXISTS itens_venda;
DROP TABLE IF EXISTS venda;
DROP TABLE IF EXISTS produto;
DROP TABLE IF EXISTS cliente;

-- =========================
-- Tabelas
-- =========================
CREATE TABLE cliente (
  idcliente   INT PRIMARY KEY,
  nome        VARCHAR(80) NOT NULL,
  telefone    VARCHAR(20)
);

CREATE TABLE produto (
  idproduto   INT PRIMARY KEY,
  descricao   VARCHAR(80) NOT NULL,
  tipo        VARCHAR(10) NOT NULL
               CHECK (tipo IN ('CAMISA','CALCA','VESTIDO')),
  preco_base  DECIMAL(10,2) NOT NULL CHECK (preco_base >= 0)
);

CREATE TABLE venda (
  idvenda     INT PRIMARY KEY,
  data        DATE NOT NULL,
  idcliente   INT NOT NULL,
  valor_total DECIMAL(10,2) NOT NULL DEFAULT 0 CHECK (valor_total >= 0),
  CONSTRAINT fk_venda_cliente FOREIGN KEY (idcliente)
    REFERENCES cliente(idcliente)
);

CREATE TABLE itens_venda (
  iditem      INT PRIMARY KEY,
  idvenda     INT NOT NULL,
  idproduto   INT NOT NULL,
  qtde        INT NOT NULL CHECK (qtde > 0),
  valor_unit  DECIMAL(10,2) NOT NULL CHECK (valor_unit >= 0),
  CONSTRAINT fk_item_venda FOREIGN KEY (idvenda) REFERENCES venda(idvenda),
  CONSTRAINT fk_item_prod  FOREIGN KEY (idproduto) REFERENCES produto(idproduto)
);

-- Índices recomendados
CREATE INDEX idx_itens_venda_idvenda   ON itens_venda(idvenda);
CREATE INDEX idx_itens_venda_idproduto ON itens_venda(idproduto);
CREATE INDEX idx_venda_data            ON venda(data);
CREATE INDEX idx_venda_idcliente       ON venda(idcliente);

-- =========================
-- Procedures e Triggers
-- =========================
DELIMITER //
CREATE PROCEDURE recalcula_total(IN p_idvenda INT)
BEGIN
  UPDATE venda v
     SET valor_total = IFNULL((
       SELECT SUM(iv.qtde * iv.valor_unit)
         FROM itens_venda iv
        WHERE iv.idvenda = p_idvenda
     ),0)
   WHERE v.idvenda = p_idvenda;
END//
DELIMITER ;

DELIMITER //
CREATE PROCEDURE abrir_venda(IN p_idvenda INT, IN p_data DATE, IN p_idcliente INT)
BEGIN
  INSERT INTO venda(idvenda, data, idcliente, valor_total)
  VALUES (p_idvenda, p_data, p_idcliente, 0);
END//
DELIMITER ;

DELIMITER //
CREATE TRIGGER trg_itens_venda_ai
AFTER INSERT ON itens_venda
FOR EACH ROW
BEGIN
  CALL recalcula_total(NEW.idvenda);
END//
DELIMITER ;

DELIMITER //
CREATE TRIGGER trg_itens_venda_au
AFTER UPDATE ON itens_venda
FOR EACH ROW
BEGIN
  CALL recalcula_total(NEW.idvenda);
END//
DELIMITER ;

DELIMITER //
CREATE TRIGGER trg_itens_venda_ad
AFTER DELETE ON itens_venda
FOR EACH ROW
BEGIN
  CALL recalcula_total(OLD.idvenda);
END//
DELIMITER ;

-- =========================
-- Dados de exemplo
-- =========================
INSERT INTO cliente (idcliente,nome,telefone) VALUES
 (1,'João','(11)1111-1111'),
 (2,'Maria','(11)2222-2222');

INSERT INTO produto (idproduto,descricao,tipo,preco_base) VALUES
 (1,'Camisa social','CAMISA',120.00),
 (2,'Calça jeans','CALCA',180.00),
 (3,'Vestido longo','VESTIDO',250.00);

INSERT INTO venda (idvenda, data, idcliente, valor_total) VALUES
 (1,'2025-08-15',1,0),
 (2,'2025-08-16',2,0);

INSERT INTO itens_venda (iditem,idvenda,idproduto,qtde,valor_unit) VALUES
 (1,1,1,2,120.00),  -- 2 camisas
 (2,1,2,1,180.00),  -- 1 calça
 (3,2,3,1,250.00);  -- 1 vestido

-- Ajusta os totais das vendas inseridas
CALL recalcula_total(1);
CALL recalcula_total(2);

-- =========================
-- Consultas pedidas (use datas do período)
-- =========================
-- Parâmetros de período (ajuste conforme necessário)
SET @ini = '2025-08-01';
SET @fim  = '2025-08-31';

-- (a) Quantidade de produtos vendidos por data (período)
SELECT v.data, SUM(iv.qtde) AS qtde_total
  FROM venda v
  JOIN itens_venda iv ON iv.idvenda = v.idvenda
 WHERE v.data BETWEEN @ini AND @fim
 GROUP BY v.data
 ORDER BY v.data;

-- (b1) Valor vendido por produto (período)
SELECT p.idproduto, p.descricao,
       SUM(iv.qtde * iv.valor_unit) AS valor_vendido
  FROM venda v
  JOIN itens_venda iv ON iv.idvenda = v.idvenda
  JOIN produto p      ON p.idproduto = iv.idproduto
 WHERE v.data BETWEEN @ini AND @fim
 GROUP BY p.idproduto, p.descricao
 ORDER BY valor_vendido DESC;

-- (b2) Valor vendido por cliente (período)
SELECT c.idcliente, c.nome,
       SUM(iv.qtde * iv.valor_unit) AS valor_vendido
  FROM venda v
  JOIN cliente c      ON c.idcliente = v.idcliente
  JOIN itens_venda iv ON iv.idvenda = v.idvenda
 WHERE v.data BETWEEN @ini AND @fim
 GROUP BY c.idcliente, c.nome
 ORDER BY valor_vendido DESC;
