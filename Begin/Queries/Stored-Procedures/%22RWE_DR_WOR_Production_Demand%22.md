ALTER PROCEDURE "RWE_DR_WOR_Production_Demand"
(IN DocNum INT)

LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	
	
WhsCode NVARCHAR(8);		
DocEntry INT;
				
/*				
* Procedure version: 20260810.ASH				
* Report Name: Shortcut 			
* Creation Date: 10.08.2026				
* Creator: RoinWorld (Alex Shtyrov)				
* Requested by: Elena Kris		
* Report Description: 			
* Object type: 
* Category: Shortcut				
*		CALL "RWE_DR_WOR_Production_Demand"(4,?);
* Change log: 	
* 		
* 10/08/2026 - Alex Shtyrov - Initial Version
*/	

BEGIN SEQUENTIAL EXECUTION	

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


SELECT R."DocEntry" INTO DocEntry FROM NNM1 N
INNER JOIN OWOR R ON N."ObjectCode" = R."ObjType" AND N."Series" = R."Series"
WHERE R."DocNum" = :DocNum;


CALL "RWE_PROC_WO_IssueProgress" (:DocEntry,:lt_result);

SELECT W1."DocNum" AS "Родит.Пр.Заказ"
,W1."PostDate" AS "Дата Род.Пр.Заказа"
,W1."DueDate" AS "План.дата Род.Пр.Заказа"
,W2."DocNum" AS "Произв.Заказ"
,W2."PostDate" AS "Дата Произв.Заказа"
,W2."DueDate" AS "План.дата Произв.Заказа"
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