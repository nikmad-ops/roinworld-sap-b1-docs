ALTER PROCEDURE "RWE_REP_ProdOrder_Explosion" 
(IN DocEntry INT
,IN RepType NVARCHAR(15))

LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN
    
DECLARE lt_result TABLE 
("SalesOrderDocEntry"      INTEGER,
 "SalesOrderDocNum"        INTEGER,
 "BomDepth"                INTEGER,       -- 0 = сам заказ на готовую продукцию
 "ItemCode"                NVARCHAR(50),
 "ItemName"                NVARCHAR(200),
 "WhsCode"                 NVARCHAR(8),
 "RequiredQty"             DECIMAL(19,6), -- WOR1."PlannedQty" (уже итоговое, не на единицу)
 "UomCode"                 NVARCHAR(10),
 "ParentItemCode"          NVARCHAR(50),
 "ParentProdOrderDocEntry" INTEGER,
 "ParentProdOrderDocNum"   INTEGER,
 "ProdOrderDocEntry"       INTEGER,       -- заполнено, если для ЭТОЙ позиции есть реальный производственный заказ
 "ProdOrderDocNum"         INTEGER,
 "ProdOrderStatusRaw"      NVARCHAR(1),   -- 'P'/'R'/'L' - для диагностики
 "PostDate"                DATE,
 "DueDate"                 DATE,
 "Status"                  NVARCHAR(60)   -- одно из 5 текстовых значений)
);

CALL "RWE_PROC_ProdOrder_Explosion" (:DocEntry,:lt_result);

IF :RepType = 'Required'
THEN 
	SELECT R."Status" AS "Статус"
	,O."DocNum" AS "Связ.Заказ на продажу"
	,R."BomDepth" AS "Глубина BoM"
	,IT."ItemCode" AS "Код товара"
	,IT."ItemName" AS "Название товара"
	,R."WhsCode" AS "Склад"
	,R."RequiredQty" AS "Треб.к-во"
	,R."UomCode" AS "ЕИ"
	,OW."DocNum" AS "Родит.Пр.Заказ"
	FROM :lt_result R
	INNER JOIN OITM IT ON R."ItemCode" = IT."ItemCode"
	INNER JOIN OWOR OW ON OW."DocEntry" = R."ParentProdOrderDocEntry"
	LEFT JOIN ORDR O ON O."ObjType" = '17' AND O."DocEntry" = R."SalesOrderDocEntry"
	WHERE UPPER(R."Status") = UPPER('Произв.заказ не создан')
	;
END IF;

IF :RepType = 'Planned'
THEN
	SELECT R."Status" AS "Статус"
	,O."DocNum" AS "Связ.Заказ на продажу"
	,R."BomDepth" AS "Глубина BoM"
	,IT."ItemCode" AS "Код товара"
	,IT."ItemName" AS "Название товара"
	,R."WhsCode" AS "Склад"
	,R."RequiredQty" AS "Треб.к-во"
	,R."UomCode" AS "ЕИ"
	,OW1."DocNum" AS "# Пр.Заказа"
	,OW2."DocNum" AS "Родит.Пр.Заказ"
	FROM :lt_result R
	INNER JOIN OITM IT ON R."ItemCode" = IT."ItemCode"
	INNER JOIN OWOR OW1 ON OW1."DocEntry" = R."ProdOrderDocEntry"
	LEFT JOIN OWOR OW2 ON OW2."DocEntry" = R."ParentProdOrderDocEntry"
	LEFT JOIN ORDR O ON O."ObjType" = '17' AND O."DocEntry" = R."SalesOrderDocEntry"
	WHERE UPPER(R."Status") = UPPER('Произв.заказ создан, Planned')
	ORDER BY OW1."Priority" ASC,OW1."PostDate"
	;
END IF;


IF :RepType = 'Released'
THEN
	SELECT R."Status" AS "Статус"
	,O."DocNum" AS "Связ.Заказ на продажу"
	,R."BomDepth" AS "Глубина BoM"
	,IT."ItemCode" AS "Код товара"
	,IT."ItemName" AS "Название товара"
	,R."WhsCode" AS "Склад"
	,R."RequiredQty" AS "Треб.к-во"
	,R."UomCode" AS "ЕИ"
	,OW1."DocNum" AS "# Пр.Заказа"
	,OW2."DocNum" AS "Родит.Пр.Заказ"
	FROM :lt_result R
	INNER JOIN OITM IT ON R."ItemCode" = IT."ItemCode"
	INNER JOIN OWOR OW1 ON OW1."DocEntry" = R."ProdOrderDocEntry"
	LEFT JOIN OWOR OW2 ON OW2."DocEntry" = R."ParentProdOrderDocEntry"
	LEFT JOIN ORDR O ON O."ObjType" = '17' AND O."DocEntry" = R."SalesOrderDocEntry"
	WHERE UPPER(R."Status") = UPPER('Произв.заказ создан, Released')
	ORDER BY R."BomDepth" DESC, OW1."Priority" ASC,OW1."PostDate"
	;
END IF;
		
END;