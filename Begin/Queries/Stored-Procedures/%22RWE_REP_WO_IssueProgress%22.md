ALTER PROCEDURE "RWE_REP_WO_IssueProgress" 
(IN DocEntry INT
,IN RepType NVARCHAR(15))

LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN

DECLARE lt_result TABLE (
		"ParentProdOrderDocEntry" INTEGER,
        "ParentProdOrderDocNum"   INTEGER,
        "ComponentLineNum"        INTEGER,
        "ComponentItemCode"       NVARCHAR(50),
        "ComponentItemName"       NVARCHAR(200),
        "ComponentPlannedQty"     DECIMAL(19,6), -- WOR1."PlannedQty" родителя - сколько нужно по спецификации
        "ChildProdOrderDocEntry"  INTEGER,
        "ChildProdOrderDocNum"    INTEGER,
        "ChildProdOrderStatusRaw" NVARCHAR(1),   -- 'P'/'R'/'L'
        "ReceivedQty"             DECIMAL(19,6), -- SUM(IGN1.Quantity) по дочернему заказу
        "IssuedQty"               DECIMAL(19,6), -- SUM(IGE1.Quantity) по строке компонента родителя
        "GapQty"                  DECIMAL(19,6), -- ReceivedQty - IssuedQty
        "Status"                  NVARCHAR(60)
);

CALL "RWE_PROC_WO_IssueProgress" (:DocEntry,:lt_result);

SELECT W1."DocNum" AS "Родит.Пр.Заказ"
,W1."PostDate" AS "Дата Род.Пр.Заказа"
,W1."DueDate" AS "План дата Род.Пр.Заказа"
,W2."DocNum" AS "Произв.Заказ"
,W2."PostDate" AS "Дата Произв.Заказа"
,W2."DueDate" AS "План дата Произв.Заказа"
,R."Status" AS "Статус"
,IT."ItemCode" AS "Код товара"
,IT."ItemName" AS "Название компонента"
,R."ComponentPlannedQty" AS "Плановое к-во"
,R."ReceivedQty" AS "Полученное к-во"
,R."IssuedQty" AS "Выданное к-во"
,R."GapQty" AS "К-во Отклонения"
FROM :lt_result R
INNER JOIN OITM IT ON R."ComponentItemCode" = IT."ItemCode"
LEFT JOIN OWOR W1 ON R."ParentProdOrderDocEntry" = W1."DocEntry"
LEFT JOIN OWOR W2 ON R."ChildProdOrderDocEntry" = W2."DocEntry"
;

END
