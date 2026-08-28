ALTER PROCEDURE "RWE_REP_WO_RootStatus" 
(IN DocEntry INT
,IN RepType NVARCHAR(15))

LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN

DECLARE lt_result TABLE(
        "RowType"                 NVARCHAR(10),  -- 'SUMMARY' или 'DETAIL' - какой отчёт сейчас в результате
        "RootProdOrderDocEntry"   INTEGER,        -- заказ верхнего уровня (в SUMMARY совпадает с ProdOrderDocEntry)
        "RootProdOrderDocNum"     INTEGER,
        "BomDepth"                INTEGER,        -- только DETAIL: 0 = сам заказ на готовую продукцию; в SUMMARY - NULL
        "ItemCode"                NVARCHAR(50),   -- SUMMARY: товар заказа верхнего уровня; DETAIL: товар этой строки
        "ItemName"                NVARCHAR(200),
        "WhsCode"                 NVARCHAR(8),    -- только DETAIL
        "RequiredQty"             DECIMAL(19,6),  -- только DETAIL
        "UomCode"                 NVARCHAR(10),   -- только DETAIL
        "ParentItemCode"          NVARCHAR(50),   -- только DETAIL
        "ParentProdOrderDocEntry" INTEGER,        -- только DETAIL
        "ParentProdOrderDocNum"   INTEGER,        -- только DETAIL
        "ProdOrderDocEntry"       INTEGER,        -- SUMMARY: сам заказ верхнего уровня; DETAIL: заказ НА ЭТУ позицию, если создан
        "ProdOrderDocNum"         INTEGER,
        "ProdOrderStatusRaw"      NVARCHAR(1),    -- 'P'/'R'/'L'
        "SalesOrderDocEntry"      INTEGER,        -- заполнено, только если у заказа верхнего уровня есть связь с Заказом на продажу
        "SalesOrderDocNum"        INTEGER,
        "PostDate"                DATE,
        "DueDate"                 DATE,
        "TotalComponentsChecked" INTEGER,        -- только SUMMARY: сколько компонентов всего проверено по всей цепочке вниз
        "MissingCount"            INTEGER,        -- только SUMMARY: сколько из них - недостающие произв.заказы
        "MissingItems"            NVARCHAR(2000), -- только SUMMARY: !!! см. примечание про STRING_AGG выше
        "MaxDepthReached"         INTEGER,        -- только SUMMARY: диагностика - до какого уровня дошло разузлование
        "Status"                  NVARCHAR(500)
    );
    
CALL "RWE_PROC_WO_RootStatus"  (:DocEntry,:RepType,:lt_result);  

IF :RepType = 'S'
THEN
	SELECT R."Status" AS "Статус строки"
	,W1."DocNum" AS "# Род.Пр.Заказа"
	,R."DueDate" AS "Дата исполнения"
	,IT."ItemCode" AS "Код товара"
	,IT."ItemName" AS "Наименование товара"
	,CASE WHEN R."ProdOrderStatusRaw" = 'P' THEN 'Запланирован'
		  WHEN R."ProdOrderStatusRaw" = 'L' THEN 'Закрыт' 
		  WHEN R."ProdOrderStatusRaw" = 'R' THEN 'Отпущен' ELSE '-' END AS "Статус Пр.Заказа"
	,R."TotalComponentsChecked" AS "Подсчёт компонентов"
	,R."MissingItems"  AS "Отсутст.Позиции"
	
	,R1."DocNum" AS "Связ.Заказ на Продажу"	  
	FROM :lt_result R
	LEFT JOIN OWOR W1 ON W1."DocEntry" = R."RootProdOrderDocEntry" 
	LEFT JOIN OITM IT ON R."ItemCode" = IT."ItemCode"
	LEFT JOIN ORDR R1 ON R."SalesOrderDocEntry" = R1."DocEntry"
	ORDER BY "BomDepth" ASC
	;
END IF;

IF :RepType = 'D'
THEN
	SELECT R."Status" AS "Статус строки"
	,W1."DocNum" AS "# Род.Пр.Заказа"
	,R."BomDepth"  AS "Глубина Специф-и"
	,IT."ItemCode" AS "Код товара"
	,IT."ItemName" AS "Наименование товара"
	,R."WhsCode" AS "Склад"
	,R."RequiredQty" AS "Треб.к-во"
	,R."UomCode" AS "ЕИ"
	,W2."DocNum" AS "# Пр.Заказа"
	,CASE WHEN R."ProdOrderStatusRaw" = 'P' THEN 'Запланирован'
		  WHEN R."ProdOrderStatusRaw" = 'L' THEN 'Закрыт' 
		  WHEN R."ProdOrderStatusRaw" = 'R' THEN 'Отпущен' ELSE '-' END AS "Статус Заказа"
	FROM :lt_result R
	LEFT JOIN OWOR W1 ON W1."DocEntry" = R."RootProdOrderDocEntry" 
	LEFT JOIN OWOR W2 ON W2."DocEntry" = R."ParentProdOrderDocEntry"
	LEFT JOIN OITM IT ON R."ItemCode" = IT."ItemCode"
	ORDER BY "BomDepth" ASC
	;
END IF;

END;  
    